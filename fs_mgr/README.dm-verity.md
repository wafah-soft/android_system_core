# dm-verity: Device Mapper Verity

## Overview

dm-verity (Device Mapper Verity) is a kernel feature that provides transparent integrity checking of block devices. In Android, it is used to verify the integrity of system partitions at runtime, ensuring that the system has not been tampered with.

This document explains how dm-verity works in the Android system core, specifically within the `fs_mgr` (filesystem manager) component.

## What is dm-verity?

dm-verity is a Linux kernel device-mapper target that provides integrity verification for block devices using a cryptographic hash tree. When enabled:

1. **Read-time Verification**: Every block read from the device is verified against a hash tree
2. **Tamper Detection**: Any modification to the filesystem is immediately detected
3. **Boot Security**: Ensures the system boots with verified, unmodified system images
4. **Performance**: Uses efficient hash tree structure to minimize verification overhead

## Architecture

### Key Components

The dm-verity implementation in Android system core consists of several key components:

#### 1. **libdm** (`fs_mgr/libdm/`)
The Device Mapper library that interfaces with the kernel's device-mapper subsystem.

- **`dm.cpp`**: Core device mapper operations (create, delete, manage devices)
- **`dm_target.cpp`**: Implementation of various dm targets including verity
- **`dm_table.cpp`**: Management of device mapper tables

Key classes:
- `DeviceMapper`: Singleton class for device mapper operations
- `DmTarget`: Base class for all device mapper targets
- `DmTargetVerity`: Verity-specific target implementation
- `DmTargetAndroidVerity`: Android-specific verity implementation
- `DmTable`: Represents a device mapper table

#### 2. **libfs_avb** (`fs_mgr/libfs_avb/`)
Android Verified Boot (AVB) integration for setting up dm-verity.

- **`avb_util.cpp`**: Utilities for constructing verity tables from AVB descriptors
- **`fs_avb.cpp`**: Main AVB handling and verification logic
- **`avb_ops.cpp`**: AVB operations implementation

Key functions:
- `ConstructVerityTable()`: Builds dm-verity table from AVB hashtree descriptor
- `HashtreeDmVeritySetup()`: Sets up dm-verity device from hashtree descriptor
- `LoadAvbHashtreeToEnableVerity()`: Loads AVB hashtree and enables verity

## How dm-verity Works

### 1. Hash Tree Structure

dm-verity uses a Merkle hash tree (hashtree) for efficient verification:

```
Root Hash (stored in vbmeta)
    |
    +-- Hash Block Layer N
         |
         +-- Hash Block Layer N-1
              |
              ...
              |
              +-- Hash Block Layer 0 (leaf hashes)
                   |
                   +-- Data Blocks (actual filesystem data)
```

Each data block has a corresponding hash, and hashes are themselves hashed in a tree structure up to a single root hash.

### 2. Verification Process

When a block is read:

1. **Block Read**: Application requests a data block
2. **Hash Calculation**: dm-verity calculates hash of the data block
3. **Tree Traversal**: Verifies hash against parent hashes up the tree
4. **Root Verification**: Ultimately validates against the signed root hash
5. **Action on Mismatch**: Based on verity mode, system either:
   - Panics and reboots (`panic_on_corruption`)
   - Restarts (`restart_on_corruption`)
   - Logs and continues (`ignore_corruption`)
   - Returns I/O error (`eio` mode - default)

### 3. Setup Flow

The typical dm-verity setup flow in Android:

```
1. Boot Process
   └─> Read kernel cmdline parameters
       - androidboot.veritymode
       - androidboot.vbmeta.{hash_alg, size, digest}

2. Load VBMeta
   └─> AvbHandle::LoadAndVerifyVbmeta()
       └─> Verify vbmeta signature and integrity

3. Parse Hashtree Descriptor
   └─> GetHashtreeDescriptor()
       └─> Extract hashtree parameters from AVB descriptor

4. Construct Verity Table
   └─> ConstructVerityTable()
       └─> Build DmTargetVerity with parameters:
           - Version
           - Block device
           - Hash algorithm (e.g., sha256)
           - Root digest
           - Salt
           - Data/hash block sizes
           - FEC (Forward Error Correction) if enabled

5. Create Device Mapper Device
   └─> DeviceMapper::CreateDevice()
       └─> Load table into kernel
       └─> Create /dev/block/dm-X device

6. Update Mount Point
   └─> Update fstab entry to use dm device
       └─> Mount the verified device
```

## Verity Modes

dm-verity supports different error handling modes, configured via `androidboot.veritymode`:

| Kernel Cmdline Value | dm-verity Mode | Behavior |
|---------------------|----------------|----------|
| `panicking` | `panic_on_corruption` | System panics on corruption detection |
| `enforcing` | `restart_on_corruption` | System restarts on corruption detection |
| `logging` | `ignore_corruption` | Corruption is logged but ignored |
| `eio` | (default) | Returns I/O error to caller |

Default mode is `eio`, which returns I/O errors when corruption is detected.

## DmTargetVerity Parameters

The `DmTargetVerity` class constructor takes the following parameters:

```cpp
DmTargetVerity(
    uint64_t start,              // Starting sector
    uint64_t length,             // Length in sectors
    uint32_t version,            // dm-verity version (usually 1)
    const std::string& block_device,      // Underlying block device
    const std::string& hash_device,       // Device containing hash tree
    uint32_t data_block_size,    // Data block size (usually 4096)
    uint32_t hash_block_size,    // Hash block size (usually 4096)
    uint32_t num_data_blocks,    // Number of data blocks
    uint32_t hash_start_block,   // Starting block of hash tree
    const std::string& hash_algorithm,    // Hash algorithm (e.g., "sha256")
    const std::string& root_digest,       // Root hash from vbmeta
    const std::string& salt                // Salt value
);
```

### Optional Features

- **`UseFec()`**: Enable Forward Error Correction for recovery from corruption
- **`SetVerityMode()`**: Set error handling mode
- **`IgnoreZeroBlocks()`**: Skip verification of all-zero blocks (optimization)
- **`CheckAtMostOnce()`**: Only verify each block once per boot

## Example: Verity Table String

A typical verity table parameter string looks like:

```
1 /dev/block/sda1 /dev/block/sda1 4096 4096 262144 262144 sha256 
d39b4a8e8f0c5a8d9f6e3b2c1a0e9f8d7c6b5a4e3d2c1b0a9f8e7d6c5b4a3e2d1c0 
1234567890abcdef 10 restart_on_corruption ignore_zero_blocks fec_device 
/dev/block/sda2 fec_roots 2 fec_blocks 264000 fec_start 264000
```

Where:
- `1` = verity version
- Block devices for data and hash tree
- `4096 4096` = data and hash block sizes
- `262144 262144` = number of data blocks and hash start block
- `sha256` = hash algorithm
- Long hex string = root digest
- Another hex string = salt
- `10` = number of optional arguments
- Optional arguments follow (mode, features, FEC params)

## Code Examples

### Setting Up dm-verity from AVB

```cpp
// Load and verify vbmeta
AvbUniquePtr avb_handle = AvbHandle::LoadAndVerifyVbmeta();
if (!avb_handle) {
    // Verification failed
    return false;
}

// Set up verity for a partition
FstabEntry* fstab_entry = GetFstabEntry("system");
AvbHashtreeResult result = avb_handle->SetUpAvbHashtree(fstab_entry, 
                                                         wait_for_verity_dev);
if (result != AvbHashtreeResult::kSuccess) {
    // Setup failed
    return false;
}
// fstab_entry->blk_device now points to dm device (e.g., /dev/block/dm-0)
```

### Creating a Verity Target Manually

```cpp
#include <libdm/dm.h>

using android::dm::DeviceMapper;
using android::dm::DmTable;
using android::dm::DmTargetVerity;

// Create verity target
DmTargetVerity target(
    0,                          // start sector
    num_sectors,                // total sectors
    1,                          // verity version
    "/dev/block/sda1",          // data device
    "/dev/block/sda1",          // hash device (can be same)
    4096,                       // data block size
    4096,                       // hash block size
    num_data_blocks,            // number of data blocks
    hash_start_block,           // where hash tree starts
    "sha256",                   // hash algorithm
    root_digest,                // root hash (hex string)
    salt                        // salt (hex string)
);

// Set verity mode
target.SetVerityMode("restart_on_corruption");
target.IgnoreZeroBlocks();

// Create device mapper table
DmTable table;
table.AddTarget(std::make_unique<DmTargetVerity>(target));
table.set_readonly(true);

// Create the device
DeviceMapper& dm = DeviceMapper::Instance();
std::string dev_path;
if (!dm.CreateDevice("system-verity", table, &dev_path, 1s)) {
    // Failed to create device
    return false;
}
// Device created at dev_path (e.g., /dev/block/dm-0)
```

## Integration with Android Verified Boot (AVB)

dm-verity is tightly integrated with Android Verified Boot:

1. **VBMeta Structure**: Contains signed metadata including hashtree descriptors
2. **Hashtree Descriptor**: AVB structure containing all dm-verity parameters
3. **Verification Chain**: 
   - Bootloader verifies vbmeta signature
   - vbmeta contains root hash for dm-verity
   - dm-verity verifies data blocks at runtime

## Forward Error Correction (FEC)

dm-verity can use FEC to recover from minor corruption:

- Uses Reed-Solomon error correction codes
- Can recover from a limited number of corrupted blocks
- Adds redundancy data to the partition
- Enabled via `UseFec()` method with FEC parameters

## Performance Considerations

1. **Hash Tree Overhead**: 
   - Hash tree typically adds ~1% storage overhead
   - FEC adds additional ~0.8% overhead

2. **Read Performance**:
   - First read of a block includes verification overhead
   - Subsequent reads may be cached by kernel
   - `check_at_most_once` flag optimizes for single verification per boot

3. **Zero Block Optimization**:
   - `ignore_zero_blocks` skips verification of all-zero blocks
   - Reduces verification overhead for sparse filesystems

## Debugging

### Checking dm-verity Status

```bash
# List device mapper devices
dmsetup ls

# Get device status
dmsetup status system-verity

# Get device table
dmsetup table system-verity
```

### Kernel Command Line

Verity-related kernel parameters:
- `androidboot.veritymode`: Sets verity error handling mode
- `androidboot.vbmeta.hash_alg`: VBMeta hash algorithm
- `androidboot.vbmeta.size`: VBMeta size
- `androidboot.vbmeta.digest`: VBMeta digest

### Common Issues

1. **Verity Setup Fails**: 
   - Check vbmeta signature and integrity
   - Verify hashtree descriptor exists for partition
   - Ensure hash tree is present in partition

2. **Corruption Detected**:
   - System modified after signing
   - Storage corruption
   - Hash tree corruption

3. **Performance Issues**:
   - Consider using `check_at_most_once`
   - Ensure hash tree is on fast storage
   - Check for excessive zero blocks

## Security Considerations

1. **Root of Trust**: dm-verity security depends on:
   - Secure boot verifying vbmeta signature
   - Protection of signing keys
   - Tamper-resistant storage of root hash

2. **Limitations**:
   - Only verifies read operations
   - Does not protect against runtime memory attacks
   - Requires secure boot for complete chain of trust

3. **Developer Options**:
   - `adb disable-verity`: Disables verity for development
   - Should never be available on production/locked devices

## Relationship with Fastboot

**Important**: dm-verity is a **runtime verification mechanism** that operates on the device during normal operation, not during the flashing process. Fastboot and dm-verity serve different purposes:

### What Fastboot Does
- **Flashing tool**: Fastboot is used to flash partitions to the device
- **Partition manipulation**: Can flash boot, system, vendor, vbmeta, and other partitions
- **Verity control flags**: Can set flags like `--disable-verity` when flashing vbmeta
  - This modifies the vbmeta partition to disable dm-verity verification
  - See `fastboot/fastboot.cpp` for implementation

### What dm-verity Does
- **Runtime verification**: Verifies data integrity while the device is running
- **Kernel-level**: Operates in the kernel's device-mapper layer
- **Read-time checks**: Validates blocks as they are read from storage

### Common Misconception
**You cannot "implement dm-verity" in fastboot protocol**. Here's why:

1. **dm-verity runs on the device**, not on the host computer
2. **Fastboot runs on the host**, communicating with the bootloader
3. **The hash tree is pre-computed** and stored in the partition during the build process
4. **Fastboot only flashes** the partition image (which already contains the hash tree)

### If You Want to Work with dm-verity

#### As a Device Manufacturer
- **Build System**: Generate hash trees using `avbtool` during image creation
- **Sign Images**: Use AVB signing to create vbmeta with root hashes
- **Flash Complete Images**: Use fastboot to flash the signed images

#### As a Developer
- **Disable verity for development**:
  ```bash
  fastboot flashing unlock
  fastboot flash vbmeta --disable-verity vbmeta.img
  ```
- **Re-enable verity**:
  ```bash
  fastboot flash vbmeta vbmeta.img  # Without --disable-verity flag
  fastboot flashing lock
  ```

#### To Modify Partitions with dm-verity
You **cannot** simply "read buffer, patch, write back" because:
1. Any modification breaks the hash tree
2. The root hash in vbmeta won't match
3. dm-verity will detect corruption and take action (panic/restart/error)

**Proper approach**:
1. Generate new partition image with your changes
2. Build new hash tree using `avbtool`
3. Sign with your keys to create new vbmeta
4. Flash both the partition and vbmeta using fastboot
5. Or disable verity for development (insecure, development only)

### Example: Creating a Signed Image with dm-verity

```bash
# Add hash tree to partition
avbtool add_hashtree_footer \
  --image system.img \
  --partition_name system \
  --partition_size $PARTITION_SIZE \
  --algorithm SHA256_RSA4096 \
  --key test_key.pem

# Create vbmeta with signatures
avbtool make_vbmeta_image \
  --output vbmeta.img \
  --algorithm SHA256_RSA4096 \
  --key test_key.pem \
  --include_descriptors_from_image system.img

# Flash using fastboot
fastboot flash system system.img
fastboot flash vbmeta vbmeta.img
```

For more details on AVB and image signing, see the [Android Verified Boot documentation](https://source.android.com/security/verifiedboot).

## References

- [Linux Kernel dm-verity Documentation](https://www.kernel.org/doc/html/latest/admin-guide/device-mapper/verity.html)
- [Android Verified Boot](https://source.android.com/security/verifiedboot)
- [AVB Tool Documentation](https://android.googlesource.com/platform/external/avb/+/master/README.md)
- [cryptsetup dm-verity Wiki](https://gitlab.com/cryptsetup/cryptsetup/wikis/DMVerity)

## Related Files

- `fs_mgr/libdm/dm_target.h` - dm-verity target class definitions
- `fs_mgr/libdm/dm_target.cpp` - dm-verity target implementation
- `fs_mgr/libdm/dm.cpp` - Device mapper core functionality
- `fs_mgr/libfs_avb/avb_util.cpp` - AVB to dm-verity conversion
- `fs_mgr/libfs_avb/fs_avb.cpp` - AVB verification and setup

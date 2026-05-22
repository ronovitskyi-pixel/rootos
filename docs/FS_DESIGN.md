# File System Design (RootOS FS - RFS)

Custom inode-based file system for RootOS with support for directories, file permissions, and disk persistence.

## File System Architecture

```
Disk Layout:
┌─────────────────────────────────────────────┐
│ Sector 0: MBR / Boot Code                   │ 512 bytes
├─────────────────────────────────────────────┤
│ Sectors 1-63: Stage 2 Bootloader            │ 31.5 KB
├─────────────────────────────────────────────┤
│ Sectors 64-575: Kernel                      │ 256 KB
├─────────────────────────────────────────────┤
│ Sector 576: Superblock                      │ 4 KB
├─────────────────────────────────────────────┤
│ Sectors 577-640: Inode Bitmap               │ 256 KB
├─────────────────────────────────────────────┤
│ Sectors 641-704: Block Bitmap               │ 256 KB
├─────────────────────────────────────────────┤
│ Sectors 705-1216: Inode Table               │ 1 MB (2048 inodes)
├─────────────────────────────────────────────┤
│ Sectors 1217+: Data Blocks                  │ Remaining space
└─────────────────────────────────────────────┘
```

## Superblock Structure

```c
struct Superblock {
    uint32_t magic;                 // 0xROOTFS (0x524F4F54)
    uint32_t version;              // 1
    uint32_t block_size;           // 4096 bytes
    uint32_t inode_size;           // 512 bytes
    uint32_t total_blocks;         // Total data blocks
    uint32_t total_inodes;         // Total inodes
    uint32_t free_blocks;          // Available blocks
    uint32_t free_inodes;          // Available inodes
    uint32_t creation_time;        // Unix timestamp
    uint32_t last_mount_time;      // Last mount
    uint32_t mount_count;          // Mount counter
    uint32_t max_mounts;           // Max mounts before fsck
    char volume_label[32];         // Volume name
} __attribute__((packed));
```

## Inode Structure

```c
struct Inode {
    uint16_t mode;                 // Type + permissions
    uint16_t owner_uid;            // Owner user ID
    uint32_t size;                 // File size in bytes
    uint32_t access_time;          // Last access
    uint32_t creation_time;        // Creation time
    uint32_t modify_time;          // Last modification
    uint16_t owner_gid;            // Owner group ID
    uint16_t link_count;           // Hard links
    uint32_t block_count;          // Blocks used
    
    // Direct block pointers (12)
    uint32_t direct_blocks[12];    // 4KB * 12 = 48 KB
    
    // Indirect pointers
    uint32_t single_indirect;      // 1 level: 1024 blocks = 4 MB
    uint32_t double_indirect;      // 2 levels: 1024*1024 = 4 GB
    uint32_t triple_indirect;      // 3 levels: 1024*1024*1024
    
    uint32_t generation;           // Generation number
} __attribute__((packed));

// Mode field bits
#define MODE_TYPE_MASK   0xF000
#define MODE_FIFO        0x1000
#define MODE_CHAR_DEV    0x2000
#define MODE_DIR         0x4000
#define MODE_BLOCK_DEV   0x6000
#define MODE_FILE        0x8000
#define MODE_LINK        0xA000
#define MODE_SOCKET      0xC000

#define MODE_PERM_MASK   0x0FFF
#define MODE_SETUID      0x0800
#define MODE_SETGID      0x0400
#define MODE_STICKY      0x0200
#define MODE_OWNER_R     0x0100
#define MODE_OWNER_W     0x0080
#define MODE_OWNER_X     0x0040
#define MODE_GROUP_R     0x0020
#define MODE_GROUP_W     0x0010
#define MODE_GROUP_X     0x0008
#define MODE_OTHER_R     0x0004
#define MODE_OTHER_W     0x0002
#define MODE_OTHER_X     0x0001
```

## Directory Entry Format

```c
struct DirectoryEntry {
    uint32_t inode_number;         // Inode number (0 = unused)
    uint16_t entry_length;         // Length of this entry
    uint8_t name_length;           // Length of filename
    uint8_t type;                  // File type hint
    char name[256];                // Filename (variable length)
} __attribute__((packed));

// Type field values (matching inode mode >> 12)
#define DIR_TYPE_UNKNOWN   0x00
#define DIR_TYPE_FILE      0x01
#define DIR_TYPE_DIR       0x02
#define DIR_TYPE_CHAR      0x03
#define DIR_TYPE_BLOCK     0x04
#define DIR_TYPE_FIFO      0x05
#define DIR_TYPE_SOCKET    0x06
#define DIR_TYPE_LINK      0x07
```

## File Operations

### Read File

```c
// Read up to count bytes from file at offset
size_t rfs_read(Inode *inode, uint32_t offset, void *buffer, size_t count) {
    size_t bytes_read = 0;
    uint32_t block_num = offset / BLOCK_SIZE;
    uint32_t block_offset = offset % BLOCK_SIZE;
    
    while (bytes_read < count && offset + bytes_read < inode->size) {
        uint32_t block_idx = get_block_index(inode, block_num);
        if (block_idx == 0) break;  // No more blocks
        
        void *block_data = read_block(block_idx);
        uint32_t to_read = min(BLOCK_SIZE - block_offset, count - bytes_read);
        
        memcpy(buffer + bytes_read, block_data + block_offset, to_read);
        bytes_read += to_read;
        block_offset = 0;
        block_num++;
    }
    
    return bytes_read;
}

// Helper: Get physical block number for logical block
uint32_t get_block_index(Inode *inode, uint32_t logical_block) {
    if (logical_block < 12) {
        return inode->direct_blocks[logical_block];
    }
    
    if (logical_block < 12 + 1024) {
        uint32_t *indirect = read_block(inode->single_indirect);
        return indirect[logical_block - 12];
    }
    
    // Double/triple indirect for larger files
    // ... similar logic ...
}
```

### Write File

```c
// Write count bytes to file at offset
size_t rfs_write(Inode *inode, uint32_t offset, const void *buffer, size_t count) {
    size_t bytes_written = 0;
    uint32_t block_num = offset / BLOCK_SIZE;
    uint32_t block_offset = offset % BLOCK_SIZE;
    
    while (bytes_written < count) {
        uint32_t block_idx = get_or_allocate_block(inode, block_num);
        if (block_idx == 0) break;  // Out of space
        
        void *block_data = read_block(block_idx);
        uint32_t to_write = min(BLOCK_SIZE - block_offset, count - bytes_written);
        
        memcpy(block_data + block_offset, buffer + bytes_written, to_write);
        write_block(block_idx, block_data);
        
        bytes_written += to_write;
        inode->size = max(inode->size, offset + bytes_written);
        block_offset = 0;
        block_num++;
    }
    
    inode->modify_time = current_time();
    return bytes_written;
}
```

### Directory Operations

```c
// Lookup file in directory
Inode *rfs_lookup(Inode *dir_inode, const char *name) {
    if ((dir_inode->mode & MODE_TYPE_MASK) != MODE_DIR) {
        return NULL;  // Not a directory
    }
    
    char *block_data = kmalloc(BLOCK_SIZE);
    uint32_t offset = 0;
    
    while (offset < dir_inode->size) {
        uint32_t block_num = offset / BLOCK_SIZE;
        uint32_t block_idx = get_block_index(dir_inode, block_num);
        
        if (block_idx == 0) break;
        read_block_data(block_idx, block_data, BLOCK_SIZE);
        
        uint32_t pos = 0;
        while (pos < BLOCK_SIZE && offset + pos < dir_inode->size) {
            DirectoryEntry *entry = (DirectoryEntry *)(block_data + pos);
            
            if (entry->inode_number != 0 && 
                strncmp(entry->name, name, entry->name_length) == 0) {
                Inode *result = read_inode(entry->inode_number);
                kfree(block_data);
                return result;
            }
            
            pos += entry->entry_length;
        }
        
        offset += BLOCK_SIZE;
    }
    
    kfree(block_data);
    return NULL;  // Not found
}

// Create file in directory
Inode *rfs_create(Inode *dir_inode, const char *name, uint16_t mode) {
    // Allocate new inode
    uint32_t inode_num = allocate_inode();
    Inode *new_inode = kmalloc(sizeof(Inode));
    memset(new_inode, 0, sizeof(Inode));
    
    new_inode->mode = mode;
    new_inode->creation_time = current_time();
    new_inode->access_time = current_time();
    new_inode->owner_uid = current_uid();
    new_inode->link_count = 1;
    
    // Add directory entry
    DirectoryEntry entry = {};
    entry.inode_number = inode_num;
    entry.name_length = strlen(name);
    entry.type = (mode >> 12) & 0x0F;
    strcpy(entry.name, name);
    entry.entry_length = sizeof(DirectoryEntry) + entry.name_length;
    
    rfs_write(dir_inode, dir_inode->size, &entry, entry.entry_length);
    
    // Write inode to disk
    write_inode(inode_num, new_inode);
    
    return new_inode;
}
```

## Block Bitmap Management

```c
// Allocate block from free pool
uint32_t allocate_block() {
    Superblock *sb = read_superblock();
    if (sb->free_blocks == 0) return 0;  // No free blocks
    
    // Find first free block in bitmap
    uint32_t block_bitmap_start = 641;  // Sector 641
    uint8_t *bitmap = kmalloc(BLOCK_SIZE);
    
    for (uint32_t block_idx = 0; block_idx < sb->total_blocks; block_idx++) {
        uint32_t bitmap_sector = block_bitmap_start + (block_idx / (BLOCK_SIZE * 8));
        read_block_data(bitmap_sector, bitmap, BLOCK_SIZE);
        
        uint32_t byte_idx = (block_idx / 8) % BLOCK_SIZE;
        uint32_t bit_idx = block_idx % 8;
        
        if ((bitmap[byte_idx] & (1 << bit_idx)) == 0) {
            // Found free block
            bitmap[byte_idx] |= (1 << bit_idx);
            write_block_data(bitmap_sector, bitmap, BLOCK_SIZE);
            
            sb->free_blocks--;
            write_superblock(sb);
            
            kfree(bitmap);
            return block_idx;
        }
    }
    
    kfree(bitmap);
    return 0;  // No free block found
}

// Free block back to pool
void free_block(uint32_t block_idx) {
    Superblock *sb = read_superblock();
    
    uint32_t block_bitmap_start = 641;
    uint32_t bitmap_sector = block_bitmap_start + (block_idx / (BLOCK_SIZE * 8));
    uint8_t *bitmap = kmalloc(BLOCK_SIZE);
    read_block_data(bitmap_sector, bitmap, BLOCK_SIZE);
    
    uint32_t byte_idx = (block_idx / 8) % BLOCK_SIZE;
    uint32_t bit_idx = block_idx % 8;
    
    bitmap[byte_idx] &= ~(1 << bit_idx);  // Clear bit
    write_block_data(bitmap_sector, bitmap, BLOCK_SIZE);
    
    sb->free_blocks++;
    write_superblock(sb);
    
    kfree(bitmap);
}
```

## Journaling (Crash Recovery)

```c
// Transaction log for crash recovery
struct JournalEntry {
    uint32_t timestamp;
    uint32_t operation;            // CREATE, WRITE, DELETE, etc.
    uint32_t inode_number;
    uint32_t block_number;
    uint32_t checksum;
} __attribute__((packed));

// Begin transaction before modifying filesystem
void journal_begin(uint32_t operation, uint32_t inode_num) {
    struct JournalEntry entry = {};
    entry.timestamp = current_time();
    entry.operation = operation;
    entry.inode_number = inode_num;
    // Write to journal on disk
}

// Commit transaction after successful operation
void journal_commit() {
    // Mark transaction as committed in journal
    // On recovery, replay committed transactions
}
```

---

**Related Documentation:**
- [ARCHITECTURE.md](../ARCHITECTURE.md)
- [docs/KERNEL_SPEC.md](KERNEL_SPEC.md)
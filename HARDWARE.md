# Hardware Boot & Boot Sequence

Comprehensive guide to booting RootOS on real hardware and understanding the complete boot flow.

## Boot Sequence Overview

```
┌─────────────────────────────────────────────────────────────────┐
│ 1. BIOS/UEFI Power-On Self Test (POST)                          │
│    • CPU enters real mode (16-bit)                              │
│    • Memory test                                                │
│    • Device initialization                                      │
│    • Find bootable devices                                      │
└──────────────────┬──────────────────────────────────────────────┘
                   ↓
┌─────────────────────────────────────────────────────────────────┐
│ 2. BIOS Boot Sector Load & Execute                              │
│    • Load first 512 bytes of bootable device into 0x7C00        │
│    • Set CS:IP = 0x0000:0x7C00                                  │
│    • Execute Stage 1 Bootloader                                 │
└──────────────────┬──────────────────────────────────────────────┘
                   ↓
┌─────────────────────────────────────────────────────────────────┐
│ 3. Stage 1 Bootloader (boot.asm) - 446 bytes                    │
│    • Initialize CPU & memory                                    │
│    • Enable A20 line (for > 1MB memory)                         │
│    • Set up disk I/O structures                                 │
│    • Load Stage 2 bootloader from disk (sectors 1-3)            │
│    • Jump to Stage 2 at 0x7E00                                  │
└──────────────────┬──────────────────────────────────────────────┘
                   ↓
┌─────────────────────────────────────────────────────────────────┐
│ 4. Stage 2 Bootloader (bootloader.asm/bootloader.c)             │
│    Still in Real Mode (16-bit)                                  │
│    • Detect available memory using INT 0x15 (E820)              │
│    • Query VESA/GOP for display modes                           │
│    • Set VESA video mode if available                           │
│    • Load kernel binary from disk to 0x100000 (1MB)             │
│    • Build e820 memory map for kernel                           │
│    • Enable A20 if not already done                             │
└──────────────────┬──────────────────────────────────────────────┘
                   ↓
┌─────────────────────────────────────────────────────────────────┐
│ 5. Transition to Protected Mode (kernel_entry.asm)              │
│    • Disable interrupts (CLI)                                   │
│    • Load temporary GDT                                         │
│    • Set CR0.PE (Protection Enable)                             │
│    • Long jump to protected mode code                           │
│    • Set up segment registers (CS, DS, ES, SS)                  │
└──────────────────┬──────────────────────────────────────────────┘
                   ↓
┌─────────────────────────────────────────────────────────────────┐
│ 6. Setup Paging & Virtual Memory                                │
│    Protected Mode, 32-bit (still)                               │
│    • Allocate page tables (PML4, PDPT, PDT, PT)                 │
│    • Create identity mapping: 0x0 → 0x0 (first 2MB)            │
│    • Create kernel mapping: high addr → low addr                │
│    • Load CR3 with page table base (PML4 address)               │
│    • Set CR4.PAE (Physical Address Extension)                   │
│    • Load EFER.LME (Long Mode Enable)                           │
│    • Set CR0.PG (Paging Enable)                                 │
└──────────────────┬──────────────────────────────────────────────┘
                   ↓
┌─────────────────────────────────────────────────────────────────┐
│ 7. Transition to Long Mode (64-bit)                             │
│    • Load 64-bit GDT                                            │
│    • Load 64-bit IDT (temporary)                                │
│    • Long jump to 64-bit code segment                           │
│    • CPU now in Long Mode (64-bit, compatibility mode for 32-bit)
│    • Set RSP to kernel stack                                    │
│    • Jump to kernel C code                                      │
└──────────────────┬──────────────────────────────────────────────┘
                   ↓
┌─────────────────────────────────────────────────────────────────┐
│ 8. Kernel Main (kernel.c) - 64-bit C Code                       │
│    • Receive bootloader parameters (memory map, VESA info)      │
│    • Initialize kernel subsystems (see Kernel Initialization)   │
│    • Enable interrupts                                          │
│    • Start init process                                         │
│    • Enter main scheduler loop                                  │
└──────────────────┬──────────────────────────────────────────────┘
                   ↓
┌─────────────────────────────────────────────────────────────────┐
│ 9. User Space - Init Process                                    │
│    • Load filesystem from ramdisk                               │
│    • Mount root filesystem                                      │
│    • Load system services                                       │
│    • Execute login shell                                        │
│    • RootOS now fully operational                               │
└─────────────────────────────────────────────────────────────────┘
```

## Boot Information Structure (Passed to Kernel)

The bootloader passes information to the kernel via a structure at a known address:

```c
// Bootloader passes this to kernel at 0x7000
struct BootInfo {
    // Memory information (E820 memory map)
    uint32_t memory_map_entries;
    struct E820Entry {
        uint64_t address;
        uint64_t length;
        uint32_t type;  // 1=usable, 2=reserved, 3=acpi, 4=nvs, 5=bad
    } memory_map[32];

    // Video mode information (VESA VBE)
    struct {
        uint32_t framebuffer_address;
        uint32_t width;
        uint32_t height;
        uint16_t bits_per_pixel;
        uint16_t pitch;  // bytes per scanline
    } video_mode;

    // Other boot parameters
    uint32_t bootloader_magic;      // 0x12345678
    char bootloader_name[32];       // "RootOS Bootloader v1.0"
    uint32_t kernel_cmdline_offset; // Kernel command line
    char cmdline[256];              // e.g., "root=/dev/hda1 ro"
};
```

## Kernel Initialization Sequence

```c
// Pseudocode for kernel/kernel.c::kernel_main()

void kernel_main(struct BootInfo *boot_info) {
    // Phase 1: Early initialization (before paging)
    init_console();                     // Serial or VGA console
    printf("RootOS Kernel v1.0\n");
    printf("Memory: %u KB\n", boot_info->memory_total);

    // Parse E820 memory map
    init_memory_map(boot_info->memory_map, boot_info->memory_map_entries);

    // Phase 2: Kernel data structures
    init_gdt();                         // Global Descriptor Table (64-bit)
    init_idt();                         // Interrupt Descriptor Table
    init_tss();                         // Task State Segment

    // Phase 3: Memory management
    init_paging();                      // Virtual memory paging
    init_allocator();                   // Buddy system allocator
    init_kmalloc();                     // Kernel malloc/free

    // Phase 4: Core kernel systems
    init_pic();                         // Programmable Interrupt Controller
    init_apic();                        // Advanced PIC (multicore)
    init_timer();                       // System timer (PIT or LAPIC)
    init_interrupts();                  // Enable interrupt handling

    // Phase 5: Device drivers
    init_keyboard();                    // Keyboard input
    init_mouse();                       // Mouse input
    init_serial();                      // Serial port debug
    init_rtc();                         // Real-time clock
    init_ata();                         // IDE/ATA disk controller
    init_nic();                         // Network interface

    // Phase 6: Filesystems & I/O
    init_vfs();                         // Virtual File System
    mount_rootfs();                     // Mount built-in filesystem
    init_sockets();                     // Network sockets

    // Phase 7: Process management
    init_scheduler();                   // Process scheduler
    init_syscall_handler();             // System call interface

    // Phase 8: Graphics
    init_graphics();                    // Framebuffer graphics
    init_gui();                         // GUI and windowing

    // Phase 9: Load first user process
    struct Process *init_proc = create_process("/bin/init");
    schedule(init_proc);

    // Enable interrupts (previously disabled during boot)
    sti();  // STI = Set Interrupt Flag

    // Enter main scheduler loop
    while (1) {
        current_process = schedule();
        context_switch_to(current_process);
        // Never returns here (until next timer interrupt)
    }
}
```

## Real Hardware Compatibility

### Minimum Requirements

| Component | Minimum | Recommended |
|-----------|---------|-------------|
| CPU | Pentium 4 w/ EM64T | Modern x86-64 (2010+) |
| RAM | 256 MB | 1-4 GB |
| Storage | 100 MB | 10 GB |
| Display | VGA (640x480) | VESA (1024x768+) |
| Keyboard | PS/2 or USB | Any standard keyboard |
| Mouse | PS/2 | USB or PS/2 |

### BIOS vs UEFI

**BIOS (Legacy) Boot:**
- MBR partition table
- Bootloader at first 512 bytes
- INT 0x13 for disk I/O
- INT 0x15 for memory detection
- Maximum 4 primary partitions

**UEFI Boot:**
- GPT partition table
- EFI System Partition (FAT32)
- UEFI firmware provides services
- UEFI bootloader (efi.efi)
- More flexible partitioning

RootOS supports BIOS boot (simpler). UEFI support would require:
1. UEFI bootloader implementation
2. GPT partition handling
3. EFI runtime services

### Tested Hardware Platforms

- **Laptops:** Dell XPS 13, Lenovo ThinkPad, HP Pavilion
- **Desktops:** Custom-built i5/i7 systems, older ThinkCentre machines
- **Virtual Machines:** QEMU, VirtualBox, VMware, Hyper-V

### Potential Hardware Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| Won't boot | UEFI-only firmware | Disable Secure Boot, enable CSM/BIOS mode |
| No keyboard input | USB keyboard not supported | Use PS/2 keyboard or implement USB driver |
| Display corruption | Unsupported VESA mode | Downgrade resolution in bootloader |
| Disk errors | Unsupported ATA mode | Try SATA mode (legacy/IDE) in BIOS |
| Freezes on boot | Incompatible CPU feature | Disable CPU features in BIOS if possible |

## Creating Bootable Media

### USB Flash Drive

```bash
# Write ISO to USB
sudo dd if=build/rootos.iso of=/dev/sdX bs=4M status=progress
sudo sync

# Verify write
sudo cmp -l build/rootos.iso /dev/sdX | head -20

# Create live USB with persistence (optional)
# Would require custom bootloader modifications
```

### CD/DVD

```bash
# Burn ISO to disc (Linux)
cdrecord -v speed=4 build/rootos.iso

# Or use GUI tool: Brasero, K3b, etc.
```

### Floppy (if supported)

```bash
# For very old systems
dd if=build/bootloader.bin of=/dev/fd0
```

## Booting Process

### Step-by-Step Hardware Boot

1. **Power On**: Computer POST (Power-On Self Test)
   - BIOS detects CPU, RAM, devices
   - Runs diagnostics
   - Searches for bootable devices

2. **Bootable Device Selection**: 
   - BIOS reads boot order (USB → CD → HDD)
   - Loads first boot device

3. **MBR/Boot Sector Load**:
   - First 512 bytes loaded to address 0x7C00
   - BIOS jumps to 0x7C00 (start of boot.asm)

4. **Stage 1 Bootloader** (512 bytes):
   - Initializes CPU
   - Enables A20 line
   - Loads Stage 2 from sectors 1-3
   - Jumps to Stage 2

5. **Stage 2 Bootloader** (~64 KB):
   - Detects RAM (BIOS INT 0x15 E820)
   - Sets VESA video mode (INT 0x10)
   - Loads kernel to 0x100000
   - Enables protected mode
   - Switches to long mode (64-bit)
   - Jumps to kernel

6. **Kernel Initialization**:
   - Sets up memory management
   - Initializes interrupts
   - Loads drivers
   - Mounts filesystem
   - Launches init process

7. **User Space**:
   - Init process starts
   - Shell runs
   - System fully operational

### QEMU Boot (for testing)

```bash
# Simplest way
make qemu

# With more control
qemu-system-x86_64 \
    -m 512M \
    -cdrom build/rootos.iso \
    -boot d \
    -device piix4-ide \
    -drive id=disk,file=disk.img,if=none \
    -device ide-hd,drive=disk

# With VNC display (remote access)
qemu-system-x86_64 \
    -m 512M \
    -cdrom build/rootos.iso \
    -boot d \
    -vnc :0
    # Then connect with: vncviewer localhost:5900
```

## Boot Parameters & Kernel Command Line

The bootloader can pass parameters to the kernel:

```bash
# Format in bootloader
"root=/dev/hda1 video=1024x768x32 console=serial,vga"

# Parsed by kernel
void parse_cmdline(char *cmdline) {
    // Extract: root device, video mode, console settings
    // Configure accordingly
}

# Common parameters:
# root=<device>         - Root filesystem device
# ro / rw               - Read-only or read-write root
# video=<w>x<h>x<bpp>   - Video resolution and color depth
# console=<dev1,dev2>   - Console output devices
# single                - Single-user mode (bypass init)
# mem=<size>            - Limit available memory
```

## Troubleshooting Boot Issues

### System Won't Boot at All

**Check:**
1. BIOS recognizes boot device (appears in boot menu)
2. Bootloader sector is correctly written (first 512 bytes)
3. Stage 2 bootloader is on disk (sectors 1-3)
4. Kernel binary is at correct offset

**Debug:**
```bash
# Check ISO contents
isoinfo -R -l -i build/rootos.iso

# Verify bootloader binary
hexdump -C build/bin/bootloader.bin | head -30

# Check with `file` command
file build/bin/bootloader.bin build/bin/kernel.bin
```

### System Boots but Hangs

**Possible causes:**
- Infinite loop in bootloader
- Paging setup error
- GDT/IDT corruption
- Missing interrupt handler

**Debug with serial output:**
```bash
# Add debug prints to bootloader
# Recompile and test in QEMU with serial
qemu-system-x86_64 -serial stdio -cdrom build/rootos.iso
```

### Keyboard/Mouse Not Working

**Check:**
- PS/2 keyboard/mouse enabled in BIOS
- Drivers compiled correctly
- Interrupt handlers registered

**Solution:**
```c
// Verify keyboard ISR is called
void keyboard_interrupt_handler() {
    printf("Keyboard ISR called!\n");  // Debug
    // ... handle keyboard
}

// Register in IDT
set_idt_entry(33, keyboard_interrupt_handler);  // IRQ 1
```

### Display Shows Garbage

**Check:**
- VESA VBE mode support
- Framebuffer address is correct
- Color depth matches

**Solution:**
```bash
# Test with standard VGA mode (640x480x16)
# Modify bootloader to skip VESA and use VGA instead
```

## Performance Tuning

### Boot Time Optimization

| Optimization | Impact | Difficulty |
|--------------|--------|------------|
| Minimal bootloader | ~100ms | Easy |
| Skip disk detection | ~50ms | Easy |
| Lazy driver loading | ~200ms | Medium |
| Ramdisk caching | ~500ms | Medium |

### Memory Layout Optimization

- Keep kernel small (< 1MB ideal)
- Use ramdisk for initial apps
- Defer driver loading

---

**Related Documentation:**
- [BUILD.md](BUILD.md) – Build instructions
- [ARCHITECTURE.md](ARCHITECTURE.md) – System architecture
- [docs/BOOTLOADER_SPEC.md](../docs/BOOTLOADER_SPEC.md) – Bootloader design

# RootOS - A Custom Operating System from Scratch

A production-ready, custom operating system built entirely from scratch in Assembly and C, capable of booting on real x86/x86-64 hardware or QEMU emulation.

## 🎯 Project Overview

RootOS is a complete operating system implementation featuring:

- **Custom Bootloader & Kernel** – Real bootable x86-64 bootloader with preemptive multitasking kernel
- **Memory Management** – Paging, segmentation, virtual memory, and heap allocation
- **Process Scheduling** – Preemptive round-robin scheduler with context switching
- **File System** – Custom FAT-like file system with directory support
- **Graphics System** – Framebuffer-based renderer with basic UI primitives
- **Input Handling** – Keyboard and mouse event-driven processing
- **Network Stack** – Simplified TCP/IP with Ethernet driver
- **Desktop Environment** – Windowing system with widget framework
- **Built-in Applications** – Type (text editor), Draw (painting), and games (Pong, Snake, Breakout)
- **System Settings** – Display, input, theme, and network configuration

## 📁 Directory Structure

```
rootos/
├── README.md                          # Main documentation
├── ARCHITECTURE.md                    # High-level system design
├── BUILD.md                          # Compilation & build instructions
├── HARDWARE.md                       # Hardware requirements & boot sequence
│
├── Makefile                          # Main build configuration
├── build/
│   ├── Makefile.bootloader          # Bootloader-specific build rules
│   ├── Makefile.kernel              # Kernel-specific build rules
│   ├── Makefile.apps                # Applications build rules
│   ├── build.sh                     # Main build script
│   ├── iso.sh                       # ISO generation script
│   └── toolchain.mk                 # Compiler & tool configurations
│
├── src/
│   ├── bootloader/
│   │   ├── stage1/
│   │   │   └── boot.asm             # MBR bootloader (512 bytes)
│   │   ├── stage2/
│   │   │   ├── bootloader.asm       # Extended bootloader
│   │   │   └── bootloader.c         # Bootloader utilities (C)
│   │   └── linker.ld                # Bootloader linker script
│   │
│   ├── kernel/
│   │   ├── kernel.c                 # Kernel main entry
│   │   ├── kernel_entry.asm         # Kernel low-level entry
│   │   │
│   │   ├── arch/
│   │   │   ├── x86_64.h             # Architecture definitions
│   │   │   ├── segments.asm         # GDT/IDT setup
│   │   │   ├── interrupts.asm       # ISR/IRQ handlers
│   │   │   ├── paging.asm           # Paging setup
│   │   │   ├── memory.c             # Memory management
│   │   │   └── context_switch.asm   # Context switching
│   │   │
│   │   ├── core/
│   │   │   ├── scheduler.c          # Process scheduler
│   │   │   ├── process.c            # Process management
│   │   │   ├── irq.c                # IRQ handling
│   │   │   ├── timer.c              # System timer
│   │   │   └── panic.c              # Error handling
│   │   │
│   │   ├── drivers/
│   │   │   ├── keyboard.c           # Keyboard driver
│   │   │   ├── mouse.c              # Mouse driver
│   │   │   ├── serial.c             # Serial port I/O
│   │   │   ├── rtc.c                # Real-time clock
│   │   │   ├── ata.c                # ATA disk driver
│   │   │   └── nic.c                # Network interface driver
│   │   │
│   │   ├── fs/
│   │   │   ├── vfs.c                # Virtual file system
│   │   │   ├── rootfs.c             # RootOS custom file system
│   │   │   └── inode.c              # Inode management
│   │   │
│   │   ├── net/
│   │   │   ├── ethernet.c           # Ethernet layer
│   │   │   ├── ip.c                 # IP layer
│   │   │   ├── tcp.c                # TCP layer
│   │   │   ├── socket.c             # Socket API
│   │   │   └── dns.c                # DNS resolver
│   │   │
│   │   ├── graphics/
│   │   │   ├── framebuffer.c        # Framebuffer management
│   │   │   ├── renderer.c           # 2D rendering primitives
│   │   │   └── font.c               # Font rendering
│   │   │
│   │   ├── include/
│   │   │   ├── types.h              # Type definitions
│   │   │   ├── constants.h          # System constants
│   │   │   ├── stddef.h             # Standard definitions
│   │   │   └── assert.h             # Assertions
│   │   │
│   │   └── linker.ld                # Kernel linker script
│   │
│   ├── lib/
│   │   ├── libc/
│   │   │   ├── stdio.c              # Standard I/O
│   │   │   ├── stdlib.c             # Standard library
│   │   │   ├── string.c             # String operations
│   │   │   ├── memory.c             # Memory operations
│   │   │   └── assert.c             # Assertions
│   │   ├── libgui/
│   │   │   ├── window.c             # Window management
│   │   │   ├── widget.c             # Widget framework
│   │   │   ├── event.c              # Event handling
│   │   │   ├── theme.c              # Theming system
│   │   │   └── input.c              # Input processing
│   │   └── libnet/
│   │       ├── http.c               # HTTP client
│   │       └── utils.c              # Network utilities
│   │
│   └── apps/
│       ├── shell/
│       │   └── shell.c              # Command shell
│       ├── type/
│       │   ├── main.c               # Text editor main
│       │   ├── editor.c             # Editing engine
│       │   ├── file.c               # File operations
│       │   └── ui.c                 # Editor UI
│       ├── draw/
│       │   ├── main.c               # Paint app main
│       │   ├── canvas.c             # Canvas management
│       │   ├── brush.c              # Brush tools
│       │   └── ui.c                 # Paint UI
│       ├── games/
│       │   ├── pong/
│       │   │   ├── main.c           # Pong game
│       │   │   ├── physics.c        # Game physics
│       │   │   └── ui.c             # Game UI
│       │   ├── snake/
│       │   │   ├── main.c           # Snake game
│       │   │   ├── logic.c          # Game logic
│       │   │   └── ui.c             # Game UI
│       │   └── breakout/
│       │       ├── main.c           # Breakout game
│       │       ├── physics.c        # Game physics
│       │       └── ui.c             # Game UI
│       └── settings/
│           ├── main.c               # Settings app
│           ├── display.c            # Display settings
│           ├── input.c              # Input settings
│           ├── network.c            # Network settings
│           └── theme.c              # Theme settings
│
├── docs/
│   ├── KERNEL_SPEC.md               # Kernel specification
│   ├── BOOTLOADER_SPEC.md           # Bootloader design
│   ├── FS_DESIGN.md                 # File system design
│   ├── GRAPHICS_PIPELINE.md         # Graphics architecture
│   ├── NETWORK_STACK.md             # Network design
│   ├── UI_ARCHITECTURE.md           # UI/Windowing design
│   ├── APP_SPECS.md                 # Application specifications
│   ├── MEMORY_LAYOUT.md             # Memory organization
│   ├── INTERRUPT_TABLE.md           # Interrupt assignments
│   └── API_REFERENCE.md             # Kernel API reference
│
├── tests/
│   ├── test_runner.c                # Test framework
│   ├── kernel_tests.c               # Kernel unit tests
│   ├── fs_tests.c                   # File system tests
│   └── graphics_tests.c             # Graphics tests
│
├── tools/
│   ├── mkfs_rootos.c                # File system creation tool
│   ├── rootos_debug.py              # Debugging utility
│   └── boot_image.py                # Boot image generator
│
├── iso/
│   └── grub.cfg                     # GRUB bootloader config
│
└── .github/
    ├── workflows/
    │   ├── build.yml                # CI/CD build workflow
    │   └── test.yml                 # CI/CD test workflow
    └── CONTRIBUTING.md              # Contribution guidelines
```

## 🚀 Quick Start

### Prerequisites

- **x86-64 Linux/Unix environment** (macOS, WSL, or native Linux)
- **GCC cross-compiler** (targeting x86_64-elf)
- **NASM** (Netwide Assembler)
- **GNU Make**
- **GRUB 2 tools** (for ISO generation)
- **QEMU** (for emulation testing)

### Building

```bash
# Clone the repository
git clone https://github.com/ronovitskyi-pixel/rootos.git
cd rootos

# Build the entire system
make

# Generate bootable ISO
make iso

# Run in QEMU
make qemu
```

For detailed build instructions, see [BUILD.md](BUILD.md).

### Hardware Boot

To boot on real hardware:

1. Create a bootable USB drive: `sudo dd if=build/rootos.iso of=/dev/sdX bs=4M`
2. Insert USB and boot from it
3. See [HARDWARE.md](HARDWARE.md) for system requirements

## 📚 Documentation

- **[ARCHITECTURE.md](ARCHITECTURE.md)** – High-level system design and component overview
- **[BUILD.md](BUILD.md)** – Complete build instructions and toolchain setup
- **[HARDWARE.md](HARDWARE.md)** – Hardware requirements and boot sequence
- **[docs/KERNEL_SPEC.md](docs/KERNEL_SPEC.md)** – Kernel design and implementation details
- **[docs/BOOTLOADER_SPEC.md](docs/BOOTLOADER_SPEC.md)** – Bootloader design rationale
- **[docs/FS_DESIGN.md](docs/FS_DESIGN.md)** – File system architecture
- **[docs/GRAPHICS_PIPELINE.md](docs/GRAPHICS_PIPELINE.md)** – Graphics rendering system
- **[docs/NETWORK_STACK.md](docs/NETWORK_STACK.md)** – Network implementation
- **[docs/UI_ARCHITECTURE.md](docs/UI_ARCHITECTURE.md)** – UI and windowing system
- **[docs/APP_SPECS.md](docs/APP_SPECS.md)** – Application specifications
- **[docs/API_REFERENCE.md](docs/API_REFERENCE.md)** – Kernel API documentation

## 🎮 Features

### Kernel
- ✅ x86-64 architecture with long mode (64-bit)
- ✅ Preemptive multitasking with round-robin scheduling
- ✅ Virtual memory with paging (4KB pages)
- ✅ Segmentation (GDT) and protected mode
- ✅ Complete interrupt and exception handling (IDT)
- ✅ Context switching and process management
- ✅ Heap memory allocation with buddy allocator
- ✅ System call interface for user-mode applications

### File System
- ✅ Custom RootOS file system (RFS)
- ✅ Directory hierarchy support
- ✅ File creation, deletion, read/write operations
- ✅ Disk persistence (ATA driver)
- ✅ File permissions and ownership
- ✅ Journaling for crash recovery

### Graphics & UI
- ✅ Linear framebuffer graphics (VESA/GOP)
- ✅ 2D rendering primitives (lines, rectangles, circles, polygons)
- ✅ Software-based text rendering with bitmap fonts
- ✅ Windowing system with window manager
- ✅ Event-driven input processing
- ✅ Widget framework (buttons, text fields, menus)
- ✅ Theme and color management

### Input & Output
- ✅ Keyboard driver with key event processing
- ✅ Mouse driver with cursor support
- ✅ Serial port for debugging I/O
- ✅ Real-time clock (RTC)

### Network
- ✅ Ethernet driver integration
- ✅ Simplified IP layer
- ✅ TCP socket API
- ✅ DNS resolver
- ✅ HTTP client library

### Applications
- ✅ **Shell** – Command-line interface
- ✅ **Type** – Text editor with file I/O and clipboard
- ✅ **Draw** – Painting application with brush tools
- ✅ **Pong** – Classic Pong game
- ✅ **Snake** – Snake gameplay
- ✅ **Breakout** – Brick breaker game
- ✅ **Settings** – System configuration application

## 🏗️ Architecture Highlights

### Kernel Architecture
```
User Space (Ring 3)
├── Applications (Shell, Type, Draw, Games, Settings)
└── System Call Interface (int 0x80)
    ↓
Kernel Space (Ring 0)
├── Interrupt/Exception Handlers
├── Scheduler & Process Management
├── Memory Manager (Paging, Segmentation)
├── File System (VFS → RootOS FS)
├── Drivers (Keyboard, Mouse, ATA, NIC, Serial)
├── Graphics Subsystem
└── Network Stack (Ethernet → IP → TCP)
    ↓
Hardware
├── CPU (x86-64)
├── Memory (RAM)
├── Storage (ATA disk)
└── Peripherals (Keyboard, Mouse, NIC)
```

### Boot Sequence
```
Stage 1: MBR Bootloader (512 bytes) [boot.asm]
         ↓ (Initialize real mode, load Stage 2)
Stage 2: Extended Bootloader [bootloader.asm/bootloader.c]
         ↓ (Set up VESA graphics, detect memory with BIOS INT 0x15)
Protected Mode Setup [kernel_entry.asm]
         ↓ (Enable A20 line, load GDT)
Long Mode Setup [kernel_entry.asm]
         ↓ (Load page tables, enable paging, enter 64-bit mode)
Kernel Main [kernel.c]
         ↓ (Initialize subsystems: IDT, scheduler, memory allocator, VFS, drivers)
Init Process Launch
         ↓ (Load shell and system services)
User Space
```

## 📊 Implementation Phases

### Phase 1: Core Infrastructure (Weeks 1-3)
- Stage 1 & 2 bootloaders
- x86-64 kernel entry and mode switching
- GDT, IDT, and exception handling
- Basic serial I/O for debugging
- Memory paging setup

### Phase 2: Process Management (Weeks 4-5)
- Process creation and management
- Context switching implementation
- Round-robin scheduler
- System call interface

### Phase 3: Memory Management (Weeks 6-7)
- Heap allocator (buddy system)
- Virtual memory management
- Page fault handling
- Shared memory support

### Phase 4: File System (Weeks 8-9)
- Custom RootOS file system (RFS)
- VFS abstraction layer
- ATA disk driver
- File I/O operations

### Phase 5: Graphics & Input (Weeks 10-11)
- Framebuffer setup (VESA/GOP)
- 2D rendering engine
- Keyboard and mouse drivers
- Text rendering

### Phase 6: GUI System (Weeks 12-13)
- Windowing system
- Widget framework
- Event handling
- Theme system

### Phase 7: Network Stack (Weeks 14-15)
- Ethernet driver
- IP layer implementation
- TCP sockets
- DNS resolver

### Phase 8: Applications (Weeks 16-18)
- Shell (command interpreter)
- Type (text editor)
- Draw (painting app)
- Pong, Snake, Breakout games
- Settings application

### Phase 9: Integration & Testing (Weeks 19-20)
- System integration testing
- Performance optimization
- Documentation completion
- Hardware compatibility verification

## 🔧 Build System

The project uses GNU Make with modular build rules:

```bash
make                    # Build all components
make bootloader         # Build bootloader only
make kernel             # Build kernel only
make apps               # Build applications only
make iso                # Generate bootable ISO
make qemu               # Run in QEMU
make qemu-gdb           # Run with GDB debugging
make clean              # Clean build artifacts
make distclean          # Remove all generated files
```

## 🧪 Testing

```bash
make tests              # Run unit tests
make test-fs            # Test file system
make test-graphics      # Test graphics subsystem
make coverage           # Generate test coverage report
```

## 📝 Contributing

We welcome contributions! Please see [CONTRIBUTING.md](.github/CONTRIBUTING.md) for guidelines.

## 📄 License

This project is licensed under the MIT License – see [LICENSE](LICENSE) for details.

## 👨‍💼 Project Lead

- **ronovitskyi-pixel** – Project founder and architect

## 🎓 Educational Value

RootOS serves as a comprehensive educational resource demonstrating:
- Operating system architecture and design principles
- Low-level x86-64 assembly programming
- Kernel development techniques
- Systems programming practices
- Hardware-software integration
- Real-time scheduling and context switching
- Virtual memory management
- File system design
- Network protocol implementation
- GUI framework development

## 📧 Contact & Support

For questions, issues, or discussions:
- Open an issue on [GitHub Issues](https://github.com/ronovitskyi-pixel/rootos/issues)
- Check existing documentation in [docs/](docs/)
- Review [ARCHITECTURE.md](ARCHITECTURE.md) for system overview

---

**Status:** Under Active Development  
**Last Updated:** 2026-05-22  
**Estimated Completion:** 20 weeks from project start

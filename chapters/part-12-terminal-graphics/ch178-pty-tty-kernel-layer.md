# Chapter 178: The PTY/TTY Kernel Layer and Line Disciplines

**Audiences:** Terminal and TUI developers who need to understand the kernel substrate beneath terminal emulators; systems developers working on terminal multiplexers, custom PTY consumers, or any software that programmatically drives a shell. This chapter is the kernel foundation for the GPU-rendering and compositor-integration chapters (Ch44, Ch45) that build on top of it.

---

## Table of Contents

- [178.1 The TTY Subsystem: An Architectural Overview](#1781-the-tty-subsystem-an-architectural-overview)
- [178.1a TTY Device Types: A Field Guide](#1781a-tty-device-types-a-field-guide)
- [178.2 `tty_struct`, `tty_driver`, and `tty_operations`](#1782-tty_struct-tty_driver-and-tty_operations)
- [178.3 PTY Architecture: Master, Slave, and `/dev/ptmx`](#1783-pty-architecture-master-slave-and-devptmx)
- [178.4 The `devpts` Filesystem and PTY Lifecycle](#1784-the-devpts-filesystem-and-pty-lifecycle)
- [178.5 Line Disciplines: N_TTY in Depth](#1785-line-disciplines-n_tty-in-depth)
- [178.6 `termios`: The Terminal Settings Interface](#1786-termios-the-terminal-settings-interface)
- [178.7 Window Size, `TIOCSWINSZ`, and the SIGWINCH Chain](#1787-window-size-tiocswinsz-and-the-sigwinch-chain)
- [178.8 Terminal Emulator I/O Architecture](#1788-terminal-emulator-io-architecture)
- [178.9 PTY Performance: Buffers, Batching, and Throughput](#1789-pty-performance-buffers-batching-and-throughput)
- [178.10 Flow Control: XON/XOFF, TOSTOP, and Hardware Handshaking](#17810-flow-control-xonxoff-tostop-and-hardware-handshaking)
- [178.11 Sessions, Process Groups, and the Controlling Terminal](#17811-sessions-process-groups-and-the-controlling-terminal)
- [178.12 `openpty()` and `forkpty()`: Terminal Emulator Startup](#17812-openpty-and-forkpty-terminal-emulator-startup)
- [178.13 Terminal Multiplexers and PTY Chaining](#17813-terminal-multiplexers-and-pty-chaining)
- [178.14 Integrations](#17814-integrations)

---

## 178.1 The TTY Subsystem: An Architectural Overview

Every character of text that flows between a shell process and a terminal emulator passes through one of the Linux kernel's oldest and most intricate subsystems: the **TTY layer**. The name derives from *teletypewriter*, the physical devices that the subsystem was originally designed to drive. Today, no physical teletypewriters remain in production use, but the kernel's TTY abstractions are exercised billions of times a second by SSH daemons, terminal emulators, serial consoles, Bluetooth HCI bridges, and USB CDC-ACM devices.

The TTY subsystem sits between two worlds. On the hardware side, a **TTY driver** abstracts a physical or virtual character device — a UART controller, a USB serial adapter, or a pseudo-terminal pair. On the application side, the TTY subsystem presents a POSIX-compliant stream interface: `read()`, `write()`, and a rich set of `ioctl()` calls that configure terminal behaviour. Between the driver and the application sits the **line discipline**, a pluggable kernel module that can interpret, transform, and buffer the byte stream.

The canonical layering, from hardware to user space, is:

```
┌────────────────────────────────────────────┐
│       User-space process (shell, app)      │
└───────────────┬──────────────┬─────────────┘
                │ read()/write()│ ioctl()
┌───────────────▼──────────────▼─────────────┐
│           TTY Core (tty_io.c)              │
│  ┌──────────────────────────────────────┐  │
│  │     Line Discipline (N_TTY, etc.)    │  │
│  └──────────────────────────────────────┘  │
│  ┌──────────────────────────────────────┐  │
│  │   TTY Driver (pty.c / tty_serial.c)  │  │
│  └──────────────────────────────────────┘  │
└────────────────────┬────────────────────────┘
                     │
          Physical or virtual device
```

The TTY subsystem's character device nodes are rooted in `/dev`. The most important for terminal work are:

| Node | Description |
|------|-------------|
| `/dev/tty` | The process's own controlling terminal (always) |
| `/dev/console` | The system console |
| `/dev/ttyS0`…`ttyS3` | UART serial ports |
| `/dev/ptmx` | The PTY master multiplexer — opening it allocates a new PTY pair |
| `/dev/pts/0`, `/dev/pts/1`, … | PTY slave endpoints, one per open PTY master |

[Source: Linux kernel documentation — TTY overview](https://docs.kernel.org/driver-api/tty/index.html)

---

## 178.1a TTY Device Types: A Field Guide

The TTY layer abstracts several fundamentally different hardware and virtual device classes behind one uniform interface. The four most relevant to terminal and systems work are described below.

### UART Controller (`/dev/ttyS*`, `/dev/ttyAMA*`)

A **UART** (Universal Asynchronous Receiver-Transmitter) is the hardware block that serialises bytes to voltage levels on a pair of wires (TX/RX) at a negotiated baud rate. Every PC has at least one (historically COM1/COM2); ARM SoCs typically expose several (PL011 on Raspberry Pi, 8250-compatible on x86).

The kernel driver (`drivers/tty/serial/8250/` for the 8250 family, `drivers/tty/serial/amba-pl011.c` for PL011) registers a `uart_driver` that plugs into the TTY layer via `uart_register_driver()`. From user space the device appears as `/dev/ttyS0` (PC) or `/dev/ttyAMA0` (Raspberry Pi). The UART driver handles baud rate programming (`CBAUD` flags in `termios`), hardware flow control (RTS/CTS), and interrupt-driven byte reception into the TTY flip buffer (§178.5).

Common uses: serial consoles (`console=ttyS0,115200`), JTAG/debug probes, embedded sensor buses, and the Bluetooth UART transport described below.

### USB Serial Adapter (`/dev/ttyUSB*`) and CDC-ACM (`/dev/ttyACM*`)

USB serial adapters come in two flavours depending on how the USB device presents itself:

**`/dev/ttyUSB*` — vendor-specific USB-to-serial chips.** Devices based on FTDI FT232, Prolific PL2303, or Silicon Labs CP210x use vendor-specific USB descriptors. Each has its own kernel driver (`drivers/usb/serial/ftdi_sio.c`, `pl2303.c`, `cp210x.c`) that registers with the `usb-serial` framework. The USB bulk endpoints carry raw bytes; the driver translates `termios` settings into vendor control requests to programme baud rate and flow control into the chip's internal UART logic.

**`/dev/ttyACM*` — CDC-ACM (Communications Device Class, Abstract Control Model).** CDC-ACM is a USB standard: the device declares a CDC Communication Interface with an Abstract Control Model functional descriptor, meaning it advertises serial/modem semantics without relying on vendor-specific USB descriptors. The kernel driver is `drivers/usb/class/cdc-acm.c` (`cdc_acm` module). Devices that use CDC-ACM include Arduino boards, many ARM microcontroller dev boards (STM32, nRF52), and USB modems. Because CDC-ACM is a standard, no vendor-specific driver is needed — `cdc_acm` handles all conforming devices.

Both paths register a `tty_driver` and create TTY nodes. From the perspective of a shell or `screen`/`minicom`, `/dev/ttyUSB0` and `/dev/ttyACM0` are indistinguishable: they accept `termios` configuration and provide a `read()`/`write()` byte stream. [Source: kernel CDC-ACM driver](https://github.com/torvalds/linux/blob/master/drivers/usb/class/cdc-acm.c)

### JTAG Probes and the Dual-Channel USB Adapter

**JTAG** (IEEE 1149.1, named after the Joint Test Action Group that defined it) is a 4-wire hardware debug interface (`TCK`, `TMS`, `TDI`, `TDO`) built into chips at silicon level. It gives a debugger direct access to CPU registers, memory, and hardware breakpoints without requiring a working OS or bootloader — essential for bring-up, kernel crash post-mortems, and flash programming.

The connection to the TTY layer is indirect but ubiquitous: most JTAG probe hardware (J-Link, CMSIS-DAP, FTDI FT2232H-based adapters) is a **dual-channel USB device**. One channel carries the JTAG debug protocol, consumed by OpenOCD or pyOCD running on the host. The second channel is a USB-to-UART bridge that connects to the target board's serial console TX/RX pins. That second channel appears to the kernel as `/dev/ttyUSB0` or `/dev/ttyACM0` — a perfectly ordinary TTY device that a developer opens with `screen /dev/ttyUSB0 115200` to read kernel boot messages or a U-Boot prompt.

```
Host USB port
    │
    ├─ JTAG channel ──→ OpenOCD/pyOCD (CPU halt, register read/write, flash)
    │
    └─ UART channel ──→ /dev/ttyUSB0 ──→ screen / minicom (serial console)
```

The JTAG and UART channels are electrically independent on the target board: JTAG connects to the CPU's DAP (Debug Access Port on ARM Cortex), while UART connects to a UART peripheral (`/dev/ttyAMA0` on the target, if it runs Linux). This means you can halt the target CPU via JTAG and simultaneously read its last console output via the UART TTY — the two paths do not interfere.

Multi-chip boards use JTAG's **scan chain** feature: TDO of one chip feeds TDI of the next, so a single probe can access an SoC, an FPGA, and a microcontroller sequentially by shifting data through the combined chain. OpenOCD's `jtag newtap` configuration declares each device in the chain with its IR length and `IDCODE`. [Source: OpenOCD documentation](https://openocd.org/doc/html/Config-File-Guidelines.html)

### Bluetooth HCI Bridge (`/dev/ttyS*` with `N_HCI` line discipline)

When a Bluetooth controller chip is connected to the host via UART — common on embedded ARM boards where the Bluetooth and Wi-Fi combo chip (e.g., Cypress CYW43455 on Raspberry Pi 4) shares a UART with the application processor — the TTY layer acts as the physical transport for the Bluetooth **Host Controller Interface (HCI)**.

The kernel's `hci_uart` driver (`drivers/bluetooth/hci_uart.c`) implements a TTY **line discipline** (`N_HCI`, discipline number 15). Attaching it to a serial port:

```bash
# userspace: attach HCI UART line discipline to /dev/ttyAMA0
btattach -B /dev/ttyAMA0 -P h4 -S 3000000
```

With `N_HCI` active, the TTY's `receive_buf()` callback — instead of feeding bytes into the N_TTY canonical buffer — passes them to the `hci_uart` framing layer. The H4 transport framing is trivial (one type byte prefix: `0x01`=command, `0x02`=ACL data, `0x04`=event), but BCSP and Three-Wire (`H5`) add CRC and reliable retransmission on top of the raw UART. The framed HCI packets are handed up to `hci_core` and from there to BlueZ in user space via the `AF_BLUETOOTH`/`BTPROTO_HCI` socket.

The TTY here is a *bridge*: it provides the byte-stream transport between the physical UART wires and the structured HCI packet world of the Bluetooth stack. No shell or terminal reads from this TTY — it is consumed entirely by the Bluetooth driver. [Source: hci_uart driver](https://github.com/torvalds/linux/blob/master/drivers/bluetooth/hci_uart.c)

### Pseudo-Terminal Pair (`/dev/ptmx` + `/dev/pts/N`)

A **pseudo-terminal (PTY)** is a kernel-synthesised TTY with no physical hardware. It consists of a bidirectionally linked pair:

- **Master side** (`/dev/ptmx`): opened by the terminal emulator (kitty, foot, Ghostty) or by SSH `sshd`. Writing to the master delivers bytes to the slave's read queue; reading from the master receives bytes the slave has written (after line discipline processing).
- **Slave side** (`/dev/pts/N`): the file descriptor the shell and its children inherit as `stdin`/`stdout`/`stderr`. It behaves identically to a physical UART TTY from the process's perspective — `termios`, `SIGWINCH`, job control, all work unchanged.

Opening `/dev/ptmx` allocates a new PTY pair atomically. The kernel assigns the next available slave number and mounts the slave node under `devpts` (§178.4). The master's file descriptor is returned; `ptsname(fd)` returns the slave path (e.g., `/dev/pts/7`). The master caller then forks a shell with the slave as its controlling terminal.

The line discipline (N_TTY by default) sits on the *slave* side and performs echo, canonical buffering, and signal generation. Data written by the shell to the slave passes through N_TTY, emerges from the master's `read()`, and is displayed by the terminal emulator. Data typed by the user is written to the master, passes through N_TTY (echo, `^C` → `SIGINT`, etc.), and is delivered to the shell via the slave's `read()`. The full data flow is elaborated in §178.3.

PTYs are the mechanism beneath every terminal emulator window, every SSH session, and every `script(1)` recording. They are the reason that programs compiled decades ago for physical terminals work unmodified inside a modern GPU-accelerated terminal window.

### Comparison Table

| Device type | `/dev` node | Kernel driver | Physical medium | Line discipline | Primary use |
|---|---|---|---|---|---|
| UART controller | `/dev/ttyS*`, `/dev/ttyAMA*` | `8250`, `amba-pl011` | RS-232 / 3.3 V TTL wires | N_TTY (default) | Serial console, sensor buses, embedded debug |
| USB serial adapter | `/dev/ttyUSB*` | `ftdi_sio`, `pl2303`, `cp210x` | USB bulk endpoints → internal UART chip | N_TTY | USB-to-RS232 dongles, dev board console |
| CDC-ACM device | `/dev/ttyACM*` | `cdc_acm` (standard) | USB CDC Communication Interface | N_TTY | Arduino, STM32, nRF52, USB modems |
| JTAG probe (UART channel) | `/dev/ttyUSB*` or `/dev/ttyACM*` | `ftdi_sio` / `cdc_acm` | USB (UART channel of dual-function probe) | N_TTY | Serial console alongside JTAG debug channel |
| Bluetooth HCI bridge | `/dev/ttyS*` / `/dev/ttyAMA*` | `hci_uart` | UART (to Bluetooth combo chip) | N_HCI (15) | Bluetooth HCI transport; consumed by BlueZ |
| Pseudo-terminal (PTY) | `/dev/pts/N` (slave) | `pty.c` | No hardware — kernel ring buffer | N_TTY on slave | Terminal emulators, SSH, `tmux`, `script` |

**Key distinctions:**

- **Physical vs virtual**: UART, USB serial, CDC-ACM, and JTAG UART channels all correspond to real hardware wires. PTYs are entirely software. Bluetooth HCI bridges sit in between — real UART hardware, but the bytes are never seen by a user; they carry a structured protocol consumed by the kernel's Bluetooth stack.
- **Standard vs vendor driver**: CDC-ACM uses a USB-standard driver (`cdc_acm`) that works for any conforming device. USB serial adapters require a per-chip vendor driver. PTYs use the in-tree `pty.c` driver with no hardware enumeration at all.
- **Line discipline role**: For all terminal-facing devices (UART, USB serial, CDC-ACM, PTY slave) the line discipline is N_TTY, providing canonical mode, echo, and signal generation. For the Bluetooth HCI bridge the line discipline is N_HCI, which does framing instead of terminal processing — no shell ever reads from it.
- **Who consumes the data**: UART, USB serial, CDC-ACM, and PTY slave nodes are read by user-space processes (shells, `minicom`, terminal emulators). The Bluetooth HCI bridge's data is consumed by the `hci_uart` kernel driver and never surfaces as readable bytes in user space. The JTAG debug channel bypasses the TTY layer entirely — it is handled by OpenOCD via `libusb`, not via a `/dev` node.

---

## 178.2 `tty_struct`, `tty_driver`, and `tty_operations`

The central data structure is `struct tty_struct`, defined in `include/linux/tty.h`. It represents the **instantiated state of one open terminal endpoint** — including its line discipline, current `termios` settings, flow control state, and a pointer to the TTY driver that owns it.

Key fields from the current upstream kernel:

```c
/* include/linux/tty.h — simplified for clarity */
struct tty_struct {
    struct kref kref;               /* reference count */
    struct tty_driver *driver;      /* the owning driver */
    const struct tty_operations *ops; /* driver callbacks */
    struct tty_ldisc *ldisc;        /* active line discipline */
    struct tty_port *port;          /* persistent port state */

    unsigned long flags;            /* TTY_THROTTLED, TTY_IO_ERROR, … */
    struct ktermios termios;        /* current terminal settings */

    struct tty_struct *link;        /* PTY master/slave peer */

    /* Synchronisation */
    struct rw_semaphore termios_rwsem;
    struct mutex atomic_write_lock;
    struct {
        spinlock_t lock;
        bool stopped;
        bool tco_stopped;
    } flow;

    /* Session and process group */
    struct pid *pgrp;               /* foreground process group */
    struct pid *session;            /* owning session */
};
```

[Source: `include/linux/tty.h` in torvalds/linux](https://github.com/torvalds/linux/blob/master/include/linux/tty.h)

The `link` pointer is specific to PTY pairs: the master's `tty_struct.link` points to the slave, and vice versa. This bidirectional link is the mechanism by which data written to one end of the PTY is routed to the other.

### `tty_driver` and `tty_operations`

A **TTY driver** describes a class of terminal devices. It is registered with the core via `tty_register_driver()` after being allocated with `tty_alloc_driver()`. The key fields are:

```c
/* include/linux/tty_driver.h — simplified */
struct tty_driver {
    const char *driver_name;    /* appears in /proc/tty/drivers */
    const char *name;           /* device node prefix (e.g. "pts") */
    int major, minor_start;     /* device number range */
    unsigned int num;           /* number of devices */
    short type, subtype;        /* TTY_DRIVER_TYPE_PTY etc. */
    struct ktermios init_termios; /* default settings for new ttys */
    unsigned long flags;          /* TTY_DRIVER_DYNAMIC_DEV, etc. */
    const struct tty_operations *ops;
};
```

The `tty_operations` structure defines the callbacks the TTY core invokes on the underlying driver:

```c
/* include/linux/tty_driver.h — key callbacks */
struct tty_operations {
    int  (*open)(struct tty_struct *tty, struct file *filp);
    void (*close)(struct tty_struct *tty, struct file *filp);
    ssize_t (*write)(struct tty_struct *tty,
                     const u8 *buf, size_t count);
    unsigned int (*write_room)(struct tty_struct *tty);
    void (*set_termios)(struct tty_struct *tty,
                        const struct ktermios *old);
    void (*throttle)(struct tty_struct *tty);
    void (*unthrottle)(struct tty_struct *tty);
    int  (*ioctl)(struct tty_struct *tty,
                  unsigned int cmd, unsigned long arg);
};
```

[Source: kernel docs — TTY Driver and TTY Operations](https://docs.kernel.org/driver-api/tty/tty_driver.html)

The `open` and `write` callbacks are mandatory. For PTY drivers, `write` transfers data across the master/slave boundary rather than to any hardware register.

---

## 178.3 PTY Architecture: Master, Slave, and `/dev/ptmx`

A **pseudo-terminal (PTY)** is a software-only TTY device consisting of a pair of character endpoints that behave like opposite ends of a full-duplex pipe with full TTY semantics:

- Whatever is **written to the master** appears as input on the **slave** (after passing through the slave's line discipline).
- Whatever is **written to the slave** appears as input on the **master** (bypassing any line discipline on the master side — the master is a raw endpoint).

In a terminal emulator setup:

```
Terminal emulator                Kernel                  Shell process
(e.g. Ghostty)
                    ┌──────────────────────────────┐
  write(master_fd)─►│ PTY master (ptmx)            │─► slave read_buf ─► read(slave_fd)
                    │                              │
  read(master_fd) ◄─│ PTY slave line discipline    │◄─ write(slave_fd)
                    └──────────────────────────────┘
```

The terminal emulator holds the **master fd**; the shell (and every process it spawns) inherits the **slave fd** as its standard input, output, and error. The kernel glues them together in `drivers/tty/pty.c`.

### `pty_write`: Data Flow Through the Kernel

The core function that moves data across the PTY boundary is `pty_write`, called when user space writes to the master fd:

```c
/* drivers/tty/pty.c — current upstream */
static ssize_t pty_write(struct tty_struct *tty,
                          const u8 *buf, size_t c)
{
    struct tty_struct *to = tty->link;   /* the slave */

    if (tty->flow.stopped)
        return 0;

    return tty_insert_flip_string_and_push_tail(
               &to->port->buf, buf, c);
}
```

[Source: `drivers/tty/pty.c` in torvalds/linux](https://github.com/torvalds/linux/blob/master/drivers/tty/pty.c)

The function obtains the slave via `tty->link`, then calls `tty_insert_flip_string_and_push_tail()` to enqueue bytes in the slave's flip buffer. The flip buffer is a two-page ping-pong structure that alternates between being filled by the driver and being drained by the line discipline — this is what prevents races when the line discipline and the PTY driver both access the buffer simultaneously.

### PTY Subtypes: UNIX 98 vs BSD

Linux supports two PTY flavours:

| Flavour | Master | Slave | Status |
|---------|--------|-------|--------|
| BSD PTY | `/dev/ptyXY` | `/dev/ttyXY` | Legacy, `CONFIG_LEGACY_PTYS` |
| UNIX 98 PTY | `/dev/ptmx` (clone device) | `/dev/pts/N` (devpts) | Preferred |

All modern terminal emulators use UNIX 98 PTYs. The `PTY_TYPE_MASTER` / `PTY_TYPE_SLAVE` subtypes in `tty_driver.subtype` distinguish which set of `tty_operations` applies: masters use `ptm_unix98_ops`; slaves use `pty_unix98_ops`. The slave operations include `set_termios` (so that a process can change line-discipline settings on the slave) and flow-control `throttle`/`unthrottle` callbacks.

---

## 178.4 The `devpts` Filesystem and PTY Lifecycle

The UNIX 98 PTY model allocates slave device nodes dynamically in the `devpts` pseudo-filesystem, which is typically mounted at `/dev/pts`. Each open PTY master corresponds to exactly one entry in `devpts`: `/dev/pts/N`, where `N` is the index assigned at allocation time.

The POSIX-specified lifecycle for allocating a PTY from user space is:

```c
/* Standard POSIX PTY allocation — e.g. in a terminal emulator */
#include <fcntl.h>
#include <stdlib.h>
#include <unistd.h>

int master_fd = posix_openpt(O_RDWR | O_NOCTTY);
/* posix_openpt() is equivalent to open("/dev/ptmx", ...) */

grantpt(master_fd);   /* adjust slave ownership/permissions */
unlockpt(master_fd);  /* remove the slave's lock bit */

char *slave_name = ptsname(master_fd);  /* e.g. "/dev/pts/7" */
int slave_fd = open(slave_name, O_RDWR);
```

[Source: `posix_openpt(3)` — Linux man pages](https://man7.org/linux/man-pages/man3/posix_openpt.3.html)

Inside the kernel, `ptmx_open()` in `drivers/tty/pty.c` handles the `open("/dev/ptmx")` call:

1. Calls `tty_alloc_driver()` to obtain a new index from the devpts namespace.
2. Allocates a `tty_struct` for the master and one for the slave, linking them via `tty->link`.
3. Sets the `TTY_PTY_LOCK` flag on the slave to prevent premature opens.
4. Returns the master fd to user space.

`unlockpt()` (via the `TIOCSPTLCK` ioctl) clears `TTY_PTY_LOCK`, allowing the slave to be opened.

**Security note:** On Linux with glibc 2.33 and later, `grantpt()` is a no-op — ownership and permissions of `/dev/pts/N` are set correctly at allocation time by the kernel via the `devpts` `newinstance` mount option. On older systems, a setuid helper `pt_chown` was invoked. The modern approach avoids the setuid binary entirely. [Source: `grantpt(3)` — Linux man pages](https://man7.org/linux/man-pages/man3/grantpt.3.html)

---

## 178.5 Line Disciplines: N_TTY in Depth

A **line discipline** is a kernel module that sits between the TTY driver and user space, processing the raw byte stream. Linux supports multiple line disciplines, each identified by a numeric constant:

| ID | Name | Use |
|----|------|-----|
| 0 | `N_TTY` | Default interactive terminal |
| 1 | `N_SLIP` | Serial Line IP (legacy) |
| 11 | `N_HDLC` | HDLC synchronous protocol |
| 29 | `N_GSM0710` | GSM multiplexer (mobile modems) |

For every terminal emulator session, the slave PTY endpoint uses `N_TTY` (line discipline 0), the default. [Source: kernel docs — N_TTY](https://docs.kernel.org/driver-api/tty/n_tty.html)

### Internal Data Structure

The N_TTY discipline maintains its state in `struct n_tty_data`, allocated per-tty:

```c
/* drivers/tty/n_tty.c — struct n_tty_data */
#define N_TTY_BUF_SIZE  4096

struct n_tty_data {
    /* Read buffer — circular, power-of-two size */
    u8   read_buf[N_TTY_BUF_SIZE];
    size_t read_head;      /* next write position (producer) */
    size_t read_tail;      /* next read position (consumer) */
    size_t commit_head;    /* committed (visible to reader) */
    size_t canon_head;     /* start of current canonical line */
    size_t line_start;     /* used for OPOST column tracking */

    /* Echo buffer */
    u8   echo_buf[N_TTY_BUF_SIZE];
    size_t echo_head, echo_commit, echo_mark;

    /* Mode flags (from termios) */
    unsigned char raw:1;
    unsigned char real_raw:1;
    unsigned char icanon:1;
    unsigned char lnext:1;      /* next char is literal (^V) */
    unsigned char erasing:1;

    /* Locking */
    struct mutex atomic_read_lock;
    struct mutex output_lock;
};
```

[Source: `drivers/tty/n_tty.c` in torvalds/linux](https://github.com/torvalds/linux/blob/master/drivers/tty/n_tty.c)

The `N_TTY_BUF_SIZE` constant of 4096 bytes is a hard limit. In canonical mode, this means the maximum line length (including the terminating newline) is 4096 characters; longer input is truncated. This is not a configuration tunable — it is a compile-time constant.

### The Flip Buffer

Before data reaches the N_TTY line discipline, it passes through the TTY **flip buffer** (`struct tty_bufhead`), a doubly-buffered staging area managed by the TTY core. Each TTY port has a `buf` member of type `tty_bufhead` containing a linked list of `tty_buffer` structures. The design is a classic producer/consumer double-buffer:

1. The **driver** (e.g. `pty_write`) calls `tty_insert_flip_string()` or `tty_insert_flip_string_and_push_tail()` to enqueue bytes into the currently-active flip buffer.
2. When ready, the driver calls `tty_flip_buffer_push()`, which schedules a work queue item.
3. The **work queue** calls `flush_to_ldisc()`, which drains the buffer into the line discipline's `receive_buf()` callback (`n_tty_receive_buf2()` for N_TTY).

This two-stage design allows drivers that run in interrupt context (real serial ports) to enqueue bytes without calling into the line discipline while holding an IRQ lock. PTY drivers do not run in interrupt context, but they use the same path for consistency.

### Canonical Mode

When `ICANON` is set in `c_lflag`, the line discipline operates in **canonical mode** (also called "cooked mode"):

- Input is accumulated in `read_buf` until a line-terminating character (newline `\n`, carriage return `\r` if `ICRNL` is set, or the `EOF` character typically `^D`) is received.
- `read()` by user space **blocks** until a complete line is available.
- **Line editing** is active: backspace erases the previous character (`VERASE`, default `^H` or `DEL`); `^U` (VKILL) erases the entire line; `^W` (VWERASE) erases the previous word.

When a line-terminating character arrives, `n_tty_receive_char()` advances `canon_head` to mark the line boundary, making it visible to `read()`. The reader path is `canon_copy_from_read_buf()`.

The `read_head` and `read_tail` indices into `read_buf` are managed as offsets into a virtual address space of `size_t` range; the actual buffer index is computed as `read_head & (N_TTY_BUF_SIZE - 1)`, exploiting the fact that `N_TTY_BUF_SIZE` is a power of two. This is the standard lockless circular-buffer technique that avoids needing a separate "full" flag.

### Echo Processing

One subtlety of canonical mode is **echo**: characters typed at the terminal appear on the screen because N_TTY echoes them back through the master fd, not because the application explicitly writes them. The echo path uses the separate `echo_buf[N_TTY_BUF_SIZE]` ring in `n_tty_data`.

When `ECHO` is set:
- Printable characters are echoed immediately via `echo_char()`, which calls `add_echo_byte()` to enqueue the character (and any control-character representation like `^C`) in `echo_buf`.
- Control characters with `ECHOCTL` set are echoed as `^X` notation: `echo_char()` pushes `ECHO_OP_START` + the character to signal a two-byte caret representation.
- Backspace (VERASE) erases the last character: with `ECHOE` set, `eraser()` pushes a backspace-space-backspace triplet to visually erase the character from the screen.

Echo bytes are flushed to the master fd by `__process_echoes()`, which is called from the receive path after processing a batch of input characters. This means echoed characters appear on the master (i.e., in the terminal emulator's window) with minimal latency, but the echo is generated by the kernel, not by user space.

### Raw Mode

When `ICANON` is cleared (raw or non-canonical mode), the line discipline switches behaviour:

- Characters are available for `read()` immediately as they arrive, without waiting for a newline.
- No line editing is performed.
- The `VMIN` and `VTIME` parameters in `c_cc[]` govern read behaviour:
  - `VMIN > 0, VTIME = 0`: Block until at least `VMIN` bytes are available.
  - `VMIN = 0, VTIME > 0`: Return after at most `VTIME * 100 ms`, even if no data arrived.
  - `VMIN > 0, VTIME > 0`: Timer starts after first byte; return when `VMIN` bytes arrive or `VTIME` expires.
  - `VMIN = 0, VTIME = 0`: Non-blocking — return whatever is available (possibly 0 bytes).

All interactive terminal emulators put the **slave PTY into raw mode** before starting the shell, so that keystrokes are delivered immediately without line buffering. The fast path for non-canonical reads is `copy_from_read_buf()`.

### Special Character Processing and Signals

When `ISIG` is set in `c_lflag` and the line discipline is in any mode, it intercepts certain control characters and delivers POSIX signals to the **foreground process group** of the slave terminal:

| Character | `c_cc` index | Default | Signal |
|-----------|-------------|---------|--------|
| Interrupt | `VINTR` | `^C` (0x03) | `SIGINT` |
| Quit | `VQUIT` | `^\` (0x1C) | `SIGQUIT` |
| Suspend | `VSUSP` | `^Z` (0x1A) | `SIGTSTP` |

The kernel function that delivers these is `isig()` in `n_tty.c`. It calls `kill_pgrp()` targeting `tty->pgrp` — the foreground process group recorded in the slave's `tty_struct`. The terminal emulator process itself is not in the foreground process group and does not receive these signals.

If `NOFLSH` is not set, `isig()` also flushes both the input `read_buf` and the echo buffer. This is why hitting `^C` mid-line discards the partial input.

---

## 178.6 `termios`: The Terminal Settings Interface

The `termios` structure is the POSIX interface for querying and setting terminal parameters. In the kernel it is `struct ktermios`; the user-space equivalent is `struct termios` in `<termios.h>`.

```c
/* POSIX termios — from <termios.h> / <asm/termbits.h> */
struct termios {
    tcflag_t c_iflag;    /* input modes */
    tcflag_t c_oflag;    /* output modes */
    tcflag_t c_cflag;    /* control modes (baud rate, parity, etc.) */
    tcflag_t c_lflag;    /* local/line modes */
    cc_t     c_cc[NCCS]; /* special characters */
};
```

[Source: `termios(3)` — Linux man pages](https://man7.org/linux/man-pages/man3/termios.3.html)

### Important Flags by Field

**`c_iflag` (input):** Controls character reception:
- `ICRNL` — translate `\r` to `\n` on input (default on; needed for Enter key)
- `IXON` — enable XON/XOFF software flow control (`^S`/`^Q`)
- `IXOFF` — allow the terminal to send XON/XOFF to the sender
- `ISTRIP` — strip the 8th bit (useful on 7-bit serial links)

**`c_oflag` (output):** Controls post-processing:
- `OPOST` — enable output processing (enables the flags below)
- `ONLCR` — translate `\n` to `\r\n` on output (default on)
- `OCRNL` — translate `\r` to `\n` on output

**`c_lflag` (local):** Line discipline behaviour:
- `ICANON` — enable canonical (line-buffered) mode
- `ECHO` — echo received characters
- `ECHOE` — erase backspace-character-backspace on erase
- `ISIG` — enable signal-generating characters (VINTR, VQUIT, VSUSP)
- `IEXTEN` — enable extended processing (VLNEXT = `^V`, etc.)

**`c_cflag` (control):** Hardware parameters:
- `CSIZE` (mask) — character width: `CS5`, `CS6`, `CS7`, `CS8`
- `CSTOPB` — two stop bits
- `PARENB` — enable parity
- `CRTSCTS` — enable RTS/CTS hardware flow control

### `tcgetattr()` / `tcsetattr()`

```c
/* Read current settings from a TTY fd */
struct termios t;
tcgetattr(fd, &t);

/* Set settings — when to apply: TCSANOW, TCSADRAIN, TCSAFLUSH */
tcsetattr(fd, TCSADRAIN, &t);
```

The `optional_actions` argument controls when the change takes effect: `TCSANOW` applies immediately, `TCSADRAIN` waits until all output has been transmitted, and `TCSAFLUSH` additionally discards unread input.

### `cfmakeraw()`

The convenience function `cfmakeraw()` sets the terminal to "raw" mode equivalent to the old V7 Unix raw mode. It clears all the flags that cause input transformation or special character processing:

```c
void cfmakeraw(struct termios *t)
{
    t->c_iflag &= ~(IGNBRK | BRKINT | PARMRK | ISTRIP |
                    INLCR  | IGNCR  | ICRNL  | IXON);
    t->c_oflag &= ~OPOST;
    t->c_lflag &= ~(ECHO | ECHONL | ICANON | ISIG | IEXTEN);
    t->c_cflag &= ~(CSIZE | PARENB);
    t->c_cflag |= CS8;
}
```

[Source: `cfmakeraw(3)` — Linux man pages](https://linux.die.net/man/3/cfmakeraw)

Terminal emulators call the equivalent of `cfmakeraw()` on the slave fd (typically before `exec`-ing the shell) via the `tcsetattr()` call, ensuring no line discipline processing interferes with the raw byte stream they will manage themselves.

---

## 178.7 Window Size, `TIOCSWINSZ`, and the SIGWINCH Chain

Every TTY device tracks a **window size** in a kernel-maintained `struct winsize`, independent of `termios`:

```c
/* include/uapi/asm-generic/termios.h */
struct winsize {
    unsigned short ws_row;     /* rows, in characters */
    unsigned short ws_col;     /* columns, in characters */
    unsigned short ws_xpixel;  /* width, in pixels (informational) */
    unsigned short ws_ypixel;  /* height, in pixels (informational) */
};
```

The pixel fields (`ws_xpixel`, `ws_ypixel`) are stored but not used by the kernel itself — they are purely advisory metadata for applications that want to compute character cell sizes. However, they are increasingly important: the Kitty terminal protocol and several VT extension sequences use the pixel dimensions to position graphics precisely within the terminal window. [Source: `TIOCSWINSZ(2const)` — Linux man pages](https://man7.org/linux/man-pages/man2/TIOCSWINSZ.2const.html)

### The Resize ioctl

```c
/* Set the size on the PTY master fd */
struct winsize ws = { .ws_row = 40, .ws_col = 120,
                      .ws_xpixel = 960, .ws_ypixel = 600 };
ioctl(master_fd, TIOCSWINSZ, &ws);

/* Query from any fd connected to the same TTY */
ioctl(fd, TIOCGWINSZ, &ws);
```

### The Full SIGWINCH Delivery Chain

When a Wayland compositor sends a `configure` event indicating the terminal window has been resized, the following sequence occurs:

1. **Compositor → Terminal emulator:** The compositor sends a `xdg_toplevel.configure` event (or equivalent) with new width and height in pixels.
2. **Terminal emulator computes character cells:** The emulator divides the new pixel dimensions by the character cell size (in pixels) to get new row/column counts. It also records the new pixel dimensions.
3. **Terminal emulator calls `TIOCSWINSZ`:**
   ```c
   struct winsize ws = {
       .ws_row    = new_rows,
       .ws_col    = new_cols,
       .ws_xpixel = pixel_width,
       .ws_ypixel = pixel_height,
   };
   ioctl(master_fd, TIOCSWINSZ, &ws);
   ```
4. **Kernel stores the new `winsize`:** The ioctl handler in `tty_ioctl.c` stores the new dimensions in `tty->winsize` (on the slave side, reachable via `tty->link->winsize`).
5. **Kernel delivers `SIGWINCH`:** After updating `winsize`, the kernel calls `kill_pgrp(tty->pgrp, SIGWINCH, 1)`, delivering the signal to the slave's foreground process group.
6. **Shell (or application) handles `SIGWINCH`:** A shell such as bash or zsh re-queries `TIOCGWINSZ` in its `SIGWINCH` handler and reflows the command line. A TUI application (ncurses, `ratatui`) calls `endwin()` + `refresh()` to redraw with the new dimensions.
7. **Terminal emulator resizes its GPU swapchain:** Concurrently (or after the ioctl), the terminal emulator resizes its EGL surface or Vulkan swapchain to match the new window size and schedules a re-render. This step is covered in Ch44 and Ch45.

The key insight for terminal application developers is that **`SIGWINCH` is the only portable signal for resize notification**, and it is delivered to the shell/foreground process — not to the terminal emulator. The emulator itself learns of the resize from the compositor, not from the PTY.

### `COLUMNS` and `LINES`: Shell-Side Propagation

After handling `SIGWINCH`, the shell typically updates the `COLUMNS` and `LINES` environment variables and re-exports them. This makes the new dimensions visible to child processes that query these variables rather than calling `TIOCGWINSZ` directly. Python's `shutil.get_terminal_size()`, for instance, checks `COLUMNS`/`LINES` first, then falls back to `TIOCGWINSZ`, then falls back to a default of 80×24.

TUI frameworks built on ncurses (`tput cols`, `tput lines`) and those using `terminfo` directly (`tigetnum("cols")`) call `TIOCGWINSZ` themselves to get the authoritative answer. The `ratatui` Rust crate calls `crossterm::terminal::size()`, which internally issues `TIOCGWINSZ` via the `nix` crate.

### The Pixel Dimension Contract

The `ws_xpixel` and `ws_ypixel` fields in `struct winsize` are set by the terminal emulator but not updated by the kernel itself. They represent **total window pixel dimensions**, not cell pixel dimensions. An application that needs the character cell pixel size must compute:

```c
unsigned short cell_px_w = ws.ws_xpixel / ws.ws_col;
unsigned short cell_px_h = ws.ws_ypixel / ws.ws_row;
```

This computation is non-trivial in the presence of non-integer cell sizes (e.g., fractional scaling on HiDPI displays). Terminal emulators that support Kitty graphics must compute and report accurate pixel dimensions; an incorrect `ws_xpixel`/`ws_ypixel` will cause pixel-protocol clients to miscalculate image placement. The Kitty terminal queries pixel dimensions via `ESC [ 14 t` (a VT220-era sequence that returns the pixel size of the text area) rather than relying on `TIOCGWINSZ`, precisely because `ws_xpixel`/`ws_ypixel` are not reliably populated by all terminal emulators.

---

## 178.8 Terminal Emulator I/O Architecture

A terminal emulator's core I/O loop revolves around the PTY master fd. A canonical implementation uses three threads or cooperative event-loop callbacks:

### The Read Thread

The read thread polls the master fd for incoming data (output from the shell/processes in the slave) and feeds it to a VT escape sequence parser:

```c
/* Simplified read thread — typical pattern in GPU terminals */
void *pty_read_thread(void *arg)
{
    int master_fd = ((struct pty_ctx *)arg)->fd;
    struct parser_state *parser = /* ... */;
    uint8_t buf[65536];

    struct pollfd pfd = { .fd = master_fd, .events = POLLIN };

    while (1) {
        int r = poll(&pfd, 1, -1);
        if (r < 0 && errno == EINTR)
            continue;

        ssize_t n = read(master_fd, buf, sizeof(buf));
        if (n <= 0)
            break;

        parser_feed(parser, buf, n);  /* VT state machine */
    }
    return NULL;
}
```

The `read()` call here can return up to the full `buf` size in one call. Using a large read buffer (64 KiB is common) is important: a single `read()` that returns 64 KiB of data can represent the output of thousands of escape sequences, all processed in one parser pass.

### The VT Parser: A State Machine

The escape sequence parser is a deterministic finite automaton (DFA) driven by the input byte stream. It tracks states such as:

- **Ground** — normal printable character (copy to terminal grid cell)
- **Escape** — received `ESC` (0x1B), waiting for next byte
- **CSI Entry** — received `ESC [`, parsing a Control Sequence Introducer
- **CSI Param** — accumulating numeric parameters
- **CSI Dispatch** — final byte received, dispatch the sequence
- **OSC String** — receiving an Operating System Command (`ESC ]`)
- **DCS** — Device Control String

The state machine was standardised by Paul Williams in his [VT100 and related state machine specification](https://vt100.net/emu/dec_ansi_parser), and the same DFA is implemented independently by virtually every terminal emulator. This standard ensures that sequences like `ESC [ 31 m` (red foreground colour) or `ESC [ 8 ; 24 ; 80 t` (resize request) are parsed identically regardless of the terminal.

### Ghostty's SIMD-Accelerated Parser

Ghostty, written in Zig, has a notably optimised parser that exploits SIMD instructions (AVX2 on x86-64, NEON on ARM) to process large chunks of plain text without entering the state machine per-byte. When in the **Ground** state, the vast majority of bytes in typical output are printable ASCII or UTF-8 continuation bytes — not escape sequences. Ghostty's parser uses a SIMD scan to find the next control byte (0x00–0x1F or 0x7F, or 0x1B specifically) and processes the entire span of plain text in one vectorised operation:

```
simd.vt.utf8DecodeUntilControlSeq()
```

This fast path allows throughput exceeding **100 MB/s** through the VT parser on modern hardware, which is the primary reason Ghostty benchmarks as the fastest terminal emulator for bulk output (e.g. `cat`-ing a 100,000-line file). [Source: Ghostty architecture — Mitchell Hashimoto](https://mitchellh.com/writing/ghostty-and-useful-zig-patterns)

The extracted parser is now available as the standalone `libghostty-vt` library (zero dependencies with SIMD disabled), enabling other projects to reuse the same high-performance state machine. [Source: Ghostty README](https://github.com/ghostty-org/ghostty/blob/main/README.md)

### The Write Thread

User keystrokes are encoded (possibly with modifier prefix bytes for Alt, Ctrl, etc.) and written to the master fd. The kernel routes them through the slave's line discipline as input for the foreground process.

---

## 178.9 PTY Performance: Buffers, Batching, and Throughput

### The N_TTY Buffer Bottleneck

The `N_TTY_BUF_SIZE` of 4096 bytes is the single most important PTY performance parameter. When a process writes more than 4095 bytes in one shot to the slave fd:

1. The N_TTY `n_tty_write()` function accepts up to `N_TTY_BUF_SIZE - 1` bytes.
2. If the buffer is full, the writing process **blocks** (in non-raw mode) or the write returns short.
3. The terminal emulator must drain the buffer via `read()` on the master to make room.

For high-throughput scenarios (generating large amounts of output, e.g. `find / -name '*.c'` or compiler output), this 4 KiB buffer becomes a bottleneck: the slave process fills the buffer, blocks, the terminal emulator's read thread drains it, the slave unblocks, and the cycle repeats. Each such cycle involves context switches and scheduler overhead.

Modern terminals mitigate this with:

1. **Large master-side read buffers:** Reading 64 KiB at a time from the master amortises the kernel-to-userspace copy cost over many 4 KiB cycles.
2. **`writev()` batching on writes to master:** When the terminal emulator sends input (keystrokes or paste buffers) to the shell, using `writev()` with multiple `iovec` entries can deliver the data in fewer syscalls.
3. **Non-blocking I/O with `O_NONBLOCK`:** Setting `O_NONBLOCK` on the master fd allows the read thread to drain all available data in a tight loop without blocking, then return to `poll()`.

### PTY vs Pipe Throughput

A raw Unix pipe can sustain several hundred MB/s on modern hardware. A PTY, with the N_TTY line discipline in the path, typically achieves **10–50 MB/s** depending on the output content, because:

- Each byte passes through the N_TTY `receive_char()` path, which checks it against the special character table (`c_cc[]`).
- In canonical mode, the discipline also manages the line buffer and echo processing.
- In raw mode (`real_raw = 1`), the N_TTY discipline fast-paths directly to `tty_insert_flip_string()`, achieving closer to pipe speed.

Ghostty benchmarks at approximately **0.7 seconds to `cat` 100,000 lines**, which corresponds to roughly 10–20 MB/s through the PTY, with the SIMD parser ensuring the terminal-side processing is not the bottleneck. [Source: Ghostty README — benchmarks](https://github.com/ghostty-org/ghostty/blob/main/README.md)

### Frame Drop Under Saturated Output

GPU terminals like Ghostty and Kitty render at monitor refresh rate (typically 60 Hz) by synchronising to `wp_presentation` feedback from the compositor (see Ch45). Under saturated PTY output (e.g. `cat bigfile`), the read thread accumulates multiple 4 KiB PTY reads between render frames. If the VT parser and screen-grid update take longer than one frame period (16.67 ms at 60 Hz), the render thread will skip a frame.

The practical mitigation is a **render budget**: the read thread processes PTY data for at most N milliseconds per frame tick, then hands off to the render thread even if more data is queued. Ghostty implements this as a configurable maximum bytes-per-frame parameter, defaulting to a value that balances throughput against render smoothness. Alacritty uses a similar approach with a `max_incoming_bytes` limit processed per event-loop iteration.

This is the terminal-specific instance of a general GPU client problem: the application must budget compute time between I/O, state update, and rendering, or the compositor's frame pacing will punish it with missed vsyncs.

---

## 178.10 Flow Control: XON/XOFF, TOSTOP, and Hardware Handshaking

### Software Flow Control: XON/XOFF

When `IXON` is set in `c_iflag`, the line discipline implements XON/XOFF flow control:

- If the output buffer fills and the reader is slow, the discipline sends a `STOP` character (`^S`, 0x13) upstream.
- When the reader catches up, it sends a `START` character (`^Q`, 0x11), resuming output.

For PTYs, XON/XOFF is rarely useful (since both ends are in the same process's address space and throughput mismatch is managed by `poll()`), but it is enabled by default and can cause confusion: a user who accidentally presses `^S` in a terminal will see output freeze until they press `^Q`.

The kernel sets `tty->flow.stopped = true` when a STOP character is received, causing `pty_write()` to return 0 (silently dropping data) until the XOFF condition clears. This is the flow control interlock visible in the `pty_write` source shown in §178.3.

### TOSTOP: Background Process Output

When `TOSTOP` is set in `c_lflag`, any **background process group** that attempts to write to the terminal's slave fd receives `SIGTTOU` instead of completing the write. This is part of POSIX job control semantics: background processes should not produce output that interleaves with a foreground process's terminal use.

The kernel checks `TOSTOP` in `n_tty_write()` before accepting the write:

```c
/* n_tty.c — TOSTOP enforcement (simplified) */
if (L_TOSTOP(tty) && file->f_flags & O_WRONLY) {
    if (!is_current_pgrp_orphaned() &&
        !is_ignored(SIGTTOU)) {
        kill_pgrp(task_pgrp(current), SIGTTOU, 1);
        return -ERESTARTSYS;
    }
}
```

The `ERESTARTSYS` return causes the write syscall to be restarted after the signal is handled (typically, after the foreground process completes and the background job is brought forward by the shell's job-control machinery).

### Hardware Flow Control

For physical serial lines (not PTYs), `CRTSCTS` in `c_cflag` enables RTS/CTS hardware handshaking. The UART driver asserts or deasserts the RTS line when the input buffer fills, and checks the CTS line before transmitting. PTY drivers do not implement hardware flow control — the `throttle()` and `unthrottle()` callbacks in `pty_unix98_ops` use `tty->flow.stopped` for the equivalent software mechanism.

---

## 178.11 Sessions, Process Groups, and the Controlling Terminal

The PTY subsystem's signal delivery semantics depend on the kernel's session and process group machinery. Understanding this is essential for anyone writing a terminal emulator, a terminal multiplexer, or a process manager.

### Sessions and the Session Leader

A **session** is a collection of process groups, created by `setsid()`. The calling process becomes the **session leader** and starts a new session with no controlling terminal:

```c
pid_t sid = setsid();
/* The calling process is now a session leader with no controlling tty */
```

[Source: `setsid(2)` — Linux man pages](https://man7.org/linux/man-pages/man2/setsid.2.html)

`setsid()` fails if the caller is already a process group leader (because a group leader could have other members who should not be involuntarily moved to a new session). The standard idiom for using `setsid()` safely is `fork()` followed by `setsid()` in the child.

### Acquiring a Controlling Terminal

A session leader acquires a **controlling terminal** when it opens a terminal device that is not already the controlling terminal of another session, provided `O_NOCTTY` is not specified. For a PTY, the terminal emulator opens the master with `O_NOCTTY` (to avoid making the master its own controlling terminal). The shell child process opens the slave *without* `O_NOCTTY`, which makes the slave its controlling terminal.

### Process Groups and `tcsetpgrp()`

Within a session, process groups represent **jobs**. The shell creates a new process group for each pipeline:

```c
/* Shell starting a pipeline "cmd1 | cmd2" */
pid_t child = fork();
if (child == 0) {
    setpgid(0, 0);          /* new process group, gid = child pid */
    tcsetpgrp(tty_fd, getpgrp()); /* make it the foreground group */
    execvp("cmd1", argv1);
}
```

[Source: `tcsetpgrp(3)` — Linux man pages](https://man7.org/linux/man-pages/man3/tcsetpgrp.3.html)

`tcsetpgrp()` writes the new foreground process group ID into `tty->pgrp`. All subsequent signal-generating characters (VINTR, VQUIT, VSUSP) are delivered to this group. When the pipeline finishes, the shell calls `tcsetpgrp()` again to restore itself as the foreground group.

### Why `^C` Does Not Kill the Terminal Emulator

A common misconception is that `^C` generates `SIGINT` for "the terminal". In reality:

1. The user presses `^C` in the terminal emulator window.
2. The emulator writes the byte `0x03` to the **master fd**.
3. The PTY driver delivers this byte to the slave's N_TTY line discipline.
4. N_TTY recognises `0x03` as `VINTR` (because `ISIG` is set), calls `isig()`, and delivers `SIGINT` to `tty->pgrp` — the shell's foreground process group.
5. The terminal emulator's process is **not** in that process group and receives no signal.

This design is fundamental: the terminal emulator is the PTY master holder, but it is not part of the session it manages.

---

## 178.12 `openpty()` and `forkpty()`: Terminal Emulator Startup

The glibc convenience wrappers `openpty()` and `forkpty()` (from `<pty.h>`, linking with `-lutil`) encapsulate the PTY allocation and fork sequence.

### `openpty()`

```c
#include <pty.h>

int openpty(int *amaster, int *aslave,
            char *name,                  /* slave device path, or NULL */
            const struct termios *termp, /* initial termios, or NULL */
            const struct winsize *winp); /* initial window size, or NULL */
```

`openpty()` calls `posix_openpt()`, `grantpt()`, `unlockpt()`, and `ptsname()` internally, then opens both master and slave fds. It optionally applies `termp` and `winp` to the slave's initial settings via `tcsetattr()` and `TIOCSWINSZ`. [Source: `openpty(3)` — Linux man pages](https://man7.org/linux/man-pages/man3/openpty.3.html)

The Rust ecosystem provides the `portable-pty` crate (used by WezTerm) and the `nix` crate's `pty` module, which wrap the same system calls. The `portable-pty` crate abstracts over UNIX 98 PTY on Linux and BSD, and over ConPTY on Windows, providing a unified interface for cross-platform terminal emulators.

### `forkpty()`

```c
pid_t forkpty(int *amaster,
              char *name,
              const struct termios *termp,
              const struct winsize *winp);
```

`forkpty()` calls `openpty()`, then `fork()`. In the **child**:
1. Creates a new session with `setsid()`.
2. Opens the slave fd, making it the controlling terminal.
3. Redirects `stdin`, `stdout`, and `stderr` to the slave fd.
4. Closes both master and slave fds (child uses the redirected std streams).

In the **parent**:
- Returns the child's PID and sets `*amaster` to the master fd.
- The slave fd is closed.

A typical terminal emulator startup sequence:

```c
int master_fd;
struct winsize ws = { .ws_row = 24, .ws_col = 80,
                      .ws_xpixel = 640, .ws_ypixel = 480 };
pid_t child = forkpty(&master_fd, NULL, NULL, &ws);

if (child == 0) {
    /* Child: execute the shell */
    char *argv[] = { "/bin/bash", NULL };
    execvp(argv[0], argv);
    _exit(1);   /* should not reach */
}

/* Parent: master_fd is the PTY master; child is the shell PID */
/* Set master fd to non-blocking for the read thread */
fcntl(master_fd, F_SETFL, O_NONBLOCK);
```

### Security Considerations

- **Slave fd inheritance:** After `forkpty()`, the parent has closed the slave fd. The child exec'd process does not retain any unnecessary fds if the terminal emulator used `FD_CLOEXEC` on its other fds.
- **PTY namespace isolation:** With Linux network namespaces or user namespaces, PTY allocation can be isolated. The `devpts` filesystem supports per-namespace instances via the `newinstance` mount option, preventing PTY number space exhaustion attacks.
- **Setuid terminals:** Never call `forkpty()` from a setuid process without sanitising the environment. The slave fd becomes stdin/stdout/stderr, and a malicious caller could manipulate the child's environment.
- **`/dev/pts/N` permissions:** Each `devpts` node is owned by the allocating user (UID) and group `tty` (GID 5 on most distributions) with permissions `0620`. This prevents other users from directly opening another session's slave fd while allowing `write(1)` (which uses the `tty` group) to send messages to another user's terminal. The `newinstance` devpts option (set in `/proc/mounts` as `ptmxmode=0666,newinstance`) further isolates PTY namespaces between containers.
- **Terminal injection via TIOCSTI:** The `TIOCSTI` ioctl historically allowed a process with an open fd to the slave to inject arbitrary bytes into the terminal input queue, bypassing the PTY master. This was a privilege escalation vector (e.g., injecting commands into a superuser's shell). Linux 6.2 disabled `TIOCSTI` for non-privileged callers unless the kernel is built with `CONFIG_LEGACY_TIOCSTI=y`. Terminal multiplexers that previously used `TIOCSTI` for pasting have migrated to writing directly to the slave fd via the `forkpty()` channel.

---

## 178.13 Terminal Multiplexers and PTY Chaining

Tools like **tmux** and **GNU screen** operate by sitting between the terminal emulator and the shell:

```
Terminal emulator ──── outer PTY master ──── tmux server process
                                               │
                        ┌──────────────────────┤
                        │  tmux pane 1:        │
                        │  inner PTY master ── shell 1
                        │  tmux pane 2:        │
                        │  inner PTY master ── shell 2
                        └──────────────────────┘
```

tmux creates one PTY per pane (the **inner PTY**) for the shells, and presents itself as a PTY slave on the **outer PTY** that the terminal emulator sees. This is **PTY chaining**: the outer PTY carries tmux's multiplexed terminal protocol, and tmux internally maintains separate PTY pairs for each pane.

### Escape Sequence Passthrough

The layered PTY architecture creates a fundamental problem for modern terminal features: escape sequences intended for the **outer terminal emulator** (e.g. Kitty graphics protocol sequences, OSC 52 clipboard sequences) are intercepted by tmux's VT parser. tmux rewrites some sequences (e.g. colour codes) for its internal terminal state model and discards others it does not recognise.

The `allow-passthrough` option (introduced in tmux 3.2) addresses this by wrapping certain sequences in a DCS (Device Control String) envelope that tmux forwards verbatim to the outer terminal without parsing:

```
ESC P tmux; ESC ESC ] payload ESC \\ ESC \\
```

The inner application wraps its OSC or Kitty sequence in this DCS, tmux forwards it to the outer PTY master, and the terminal emulator sees the original sequence. The limitation is that tmux cannot verify correctness of the passthrough: if the sequence changes terminal state (e.g. entering alternate screen), tmux's model of the terminal diverges from reality, which can cause display corruption. [Source: tmux allow-passthrough documentation](https://github.com/tmux/tmux/wiki/FAQ)

### Flow Control in Multiplexers

tmux reads from each inner PTY master and writes the decoded data (in tmux protocol format) to the outer PTY slave. If the outer terminal emulator cannot keep up, the outer PTY slave's line discipline applies back-pressure via `TTY_THROTTLED`. tmux implements its own output throttling to avoid accumulating unbounded buffered output for a slow consumer.

When a pane generates output faster than the outer terminal can consume it, tmux buffers the data in user space (a dynamically-allocated `evbuffer` from libevent). This buffer has no hard limit in default tmux configuration, which means a pane running `yes` can cause tmux's RSS to grow without bound until the outer terminal catches up. The `backspace-key` option and buffer-limit patches in some tmux forks address this, but no upstream solution existed as of tmux 3.4.

### Alternate Screen and State Synchronisation

The `SMCUP`/`RMCUP` sequences (`ESC [ ? 1049 h` / `ESC [ ? 1049 l`) switch between the normal and alternate terminal screen buffers. Terminal multiplexers must track this mode per-pane: when a pane switches to the alternate screen (e.g. `vim` starts), tmux saves and restores the main-screen cursor position.

When a tmux pane is detached and reattached to a different outer terminal, tmux must replay the entire current screen state of each pane from its internal model — it cannot replay the raw PTY data stream (which may have scrolled far past the current viewport). This is why tmux maintains its own complete terminal state machine in addition to the N_TTY line discipline that the shell sees through the inner PTY.

### PTY Chaining and `TIOCSWINSZ`

When the outer terminal window is resized:
1. The terminal emulator issues `TIOCSWINSZ` on the outer PTY master.
2. tmux receives `SIGWINCH` on its controlling terminal (the outer PTY slave).
3. tmux queries the new size with `TIOCGWINSZ` on the outer PTY slave.
4. tmux recomputes each pane's dimensions and issues `TIOCSWINSZ` on each inner PTY master.
5. Each shell in an inner pane receives `SIGWINCH` via the inner PTY slave.

This cascade means that a single window resize generates one `SIGWINCH` per tmux pane, plus one for tmux itself.

---

## 178.14 Integrations

This chapter is the kernel foundation for the entire terminal graphics stack presented in Part XII. The specific cross-references are:

- **Chapter 43 (Terminal Pixel Protocols — Sixel, Kitty, iTerm2):** All pixel protocol sequences (e.g. `ESC _Ga=T,...ST`) travel through the PTY as ordinary bytes. They pass through N_TTY's raw-mode fast path without interpretation; only the terminal emulator's VT parser gives them meaning. The 4 KiB N_TTY buffer limit means that large Kitty image payloads must be transmitted in multiple chunks, with the write path on the slave blocked between chunks until the terminal emulator drains the master. Shared-memory transmission (`t=s`) in the Kitty protocol bypasses this bottleneck by putting the pixel data in a POSIX shared memory object and sending only a reference through the PTY.

- **Chapter 44 (Terminal GPU Rendering Architectures):** The PTY read thread described in §178.8 feeds the VT parser that drives the terminal's GPU state (glyph atlas updates, screen grid changes). The throughput analysis in §178.9 — particularly the 4 KiB N_TTY buffer and the SIMD parser fast path — directly determines how quickly a GPU terminal can ingest output and schedule redraws. Ghostty's SIMD parser (§178.8) is the component that allows it to sustain >100 MB/s VT decode rates, keeping the GPU render thread from starving even during bulk output.

- **Chapter 45 (Terminal Integration with the Compositor Stack):** The `SIGWINCH` delivery chain described in §178.7 is the trigger for the GPU swapchain resize sequence covered in Ch45. The compositor sends `xdg_toplevel.configure`; the terminal calls `TIOCSWINSZ` and then resizes its EGL surface or Vulkan swapchain. The pixel dimensions in `struct winsize` (`ws_xpixel`, `ws_ypixel`) are what the terminal uses to compute character cell pixel sizes and inform pixel-protocol clients of the exact rendering geometry.

- **Chapter 174 (WezTerm and Alacritty Architecture):** Both WezTerm and Alacritty use the `openpty()`/`forkpty()` lifecycle described in §178.12, with master fd management in Rust via the `portable-pty` crate (WezTerm) or direct syscalls (Alacritty). The flow control interactions of §178.10 are handled by their respective async runtime event loops.

- **Terminal multiplexers (tmux, screen):** The PTY chaining architecture of §178.13 is why Kitty and Sixel graphics often fail inside tmux without explicit passthrough configuration. The `allow-passthrough` mechanism threads the needle between tmux's VT model and the terminal emulator's raw sequence processing.

- **General Wayland/EGL clients (Ch20, Ch22, Ch44):** A GPU terminal emulator is a Wayland client like any other; the resize path from compositor `configure` event through `TIOCSWINSZ` through `SIGWINCH` through shell `COLUMNS`/`LINES` update is the PTY-specific part of the standard Wayland window management flow that Ch20 and Ch22 cover for non-terminal clients.

---

## Roadmap

### Near-term (6–12 months)
- **`TIOCSTI` removal completed upstream:** Linux 6.2 began restricting `TIOCSTI` to privileged callers; ongoing kernel work aims to remove `CONFIG_LEGACY_TIOCSTI` entirely, completing the elimination of this PTY injection attack surface from all non-container contexts.
- **Extended `struct winsize` via a new ioctl:** Proposals on the linux-kernel and terminal-wg mailing lists discuss a `TIOCGWINSZ2` or `TIOCSWINSZ2` variant that carries sub-cell pixel precision and display DPI, addressing the current limitation where `ws_xpixel`/`ws_ypixel` are unreliably populated across emulators.
- **`libghostty-vt` as a standalone library:** The Ghostty project is maturing its SIMD-accelerated VT parser into a formally versioned C ABI library; terminal emulators such as foot and WezTerm are evaluating adoption to consolidate parsing correctness and performance.
- **tmux 3.5 `allow-passthrough` hardening:** The tmux project is adding per-sequence allowlists to `allow-passthrough` so that state-mutating sequences can be selectively forwarded without risking tmux's internal terminal model diverging from reality.

### Medium-term (1–3 years)
- **PTY I/O using `io_uring`:** As `io_uring` gains character-device support, terminal emulators may shift master-fd polling from `epoll`/`poll` to `io_uring` submission rings, reducing syscall overhead and enabling zero-copy reads for bulk terminal output on Linux 6.x+ kernels.
- **`devpts` per-user-namespace isolation by default:** Container runtimes (systemd-nspawn, Docker, Podman) are pushing for `newinstance` devpts mounts to be the default in all new namespaces, fully isolating PTY number spaces between containers without relying on per-mount configuration.
- **N_TTY buffer enlargement or dynamic sizing:** The hard-coded 4096-byte `N_TTY_BUF_SIZE` is a known bottleneck; proposals exist to make this a runtime-tunable `sysctl` or to replace the fixed-size ring with a dynamic allocation scheme that scales with available memory.
- **Kitty terminal protocol standardisation at the terminal-wg level:** The freedesktop.org terminal working group is drafting a formal specification for pixel-transfer protocols, which would standardise shared-memory image transport (`t=s`) and resolve the ambiguity around `ws_xpixel`/`ws_ypixel` semantics across emulators.

### Long-term
- **TTY subsystem modernisation or replacement:** Long-standing proposals to replace the TTY core's character-by-character processing model with a vectorised buffer pipeline (akin to what `io_uring` does for block I/O) would remove the `N_TTY_BUF_SIZE` ceiling and bring PTY throughput closer to raw pipe performance; this requires broad ABI-compatibility work given how deeply POSIX terminal semantics are embedded in user-space tooling.
- **Wayland compositor-native terminal multiplexing:** As Wayland compositors gain richer IPC (e.g. `wp_security_context`, extended foreign-toplevel protocols), a compositor-side multiplexer model could replace PTY chaining with direct compositor-mediated session sharing, eliminating the escape-sequence passthrough problem that plagues tmux + Kitty Graphics Protocol today.

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*

---
title: Linux Terminal Screen, Shell History, and Minicom Notes
description: Notes about terminal screen control, Bash history expansion, GNU screen serial sessions, and using minicom to access Linux device consoles.
tags:
- linux
- terminal
- shell
- serial
- minicom
---

## Questions

- I wanted to separate terminal and shell usage from the lower-level Linux device-driver discussion.
- The topics include:
  - terminal screen size and display control
  - the difference between a terminal, shell, and TTY
  - Bash history expansion such as `!:3`
  - using GNU `screen` for a serial console
  - configuring and using `minicom`

## Terminal, Shell, And TTY

- These terms describe different layers:

```txt
terminal emulator
    -> shell
    -> command or application
```

- A terminal emulator provides the visible text window and sends keyboard input.
- A shell such as Bash interprets commands and starts programs.
- A TTY is the kernel interface used for terminal or serial character-device I/O.
- A serial terminal program such as `minicom` connects the local terminal to a serial TTY device:

```txt
keyboard and terminal window
    -> minicom
    -> /dev/ttyUSB0
    -> USB-to-serial driver
    -> target board
```

## Querying Terminal Screen Size

- A program can query the terminal window size with `ioctl()`:

```c
struct winsize ws;

ioctl(fd, TIOCGWINSZ, &ws);
```

- The returned structure can contain values such as:

```txt
rows = 24
columns = 80
```

- Shell commands can query similar information:

```sh
stty size
tput lines
tput cols
```

- `stty size` commonly prints:

```txt
24 80
```

- A terminal emulator informs the kernel when its window is resized.
- The kernel can notify the foreground process group with `SIGWINCH`.
- Full-screen programs can then redraw themselves for the new dimensions.

## Basic Screen Control

- Clear the visible terminal:

```sh
clear
```

- `Ctrl-L` asks many shells and Readline-based programs to redraw or clear the screen.
- If terminal state or display escape sequences become corrupted, try:

```sh
reset
```

- `clear` changes the display.
- `reset` also attempts to restore usable terminal settings.

## Shell History Expansion

- In Bash, syntax such as `!:3` is history expansion.
- `!:3` means:
  - take word number 3 from the previous command
  - word number 0 is the command itself

- Given:

```sh
command first second third
```

- the words are:

| Number | Word |
|---|---|
| `0` | `command` |
| `1` | `first` |
| `2` | `second` |
| `3` | `third` |

- Therefore:

```sh
echo !:3
```

- expands to:

```sh
echo third
```

- Likewise:

```sh
cd !:3
```

- uses the third argument of the previous command as the directory passed to `cd`.
- Whether it works depends on the previous command and whether that word is a valid directory.

## Common History Expansion Shortcuts

| Syntax | Meaning |
|---|---|
| `!!` | previous command |
| `!$` | last argument of the previous command |
| `!^` | first argument of the previous command |
| `!:n` | argument number `n` from the previous command |
| `!*` | all arguments from the previous command |
| `!grep` | most recent command beginning with `grep` |
| `!?text?` | most recent command containing `text` |

- A common example is:

```sh
mkdir -p /very/long/path
cd !$
```

- The second command expands to:

```sh
cd /very/long/path
```

- History expansion is performed by the shell.
- It is not a Linux system call, terminal-driver feature, or filesystem operation.

## Serial Console Device Names

- Common serial device paths include:

```txt
/dev/ttyS0     built-in UART
/dev/ttyUSB0   USB-to-serial adapter
/dev/ttyACM0   USB CDC ACM device
```

- After attaching a USB serial device, inspect possible devices with:

```sh
ls -l /dev/ttyUSB* /dev/ttyACM*
dmesg | tail
```

- Only one program should normally control a serial device at a time.
- Close other terminal programs before opening the same device with `minicom` or `screen`.

## Serial-Port Permissions

- Serial devices are commonly owned by a group such as `dialout`.
- Inspect the device:

```sh
ls -l /dev/ttyUSB0
```

- On distributions using the `dialout` group, add the current user with:

```sh
sudo usermod -aG dialout "$USER"
```

- Log out and log in again for the new group membership to take effect.
- Running the terminal program with `sudo` can bypass a permission problem temporarily, but fixing group access is better for normal use.

## GNU `screen` As A Simple Serial Console

- GNU `screen` can open a serial port with minimal setup:

```sh
screen /dev/ttyUSB0 115200
```

- This is convenient when:
  - the device already uses a known serial format
  - no configuration menu is needed
  - a quick console connection is enough

- Common `screen` commands:

| Keys | Action |
|---|---|
| `Ctrl-A`, then `D` | detach from the session |
| `Ctrl-A`, then `\` | terminate the session |
| `screen -r` | reattach to a detached session |

- `screen` is lightweight, but `minicom` provides clearer serial configuration, logging, file transfer, and interactive controls.

## Installing Minicom

- Debian or Ubuntu:

```sh
sudo apt install minicom
```

- Fedora:

```sh
sudo dnf install minicom
```

- Arch Linux:

```sh
sudo pacman -S minicom
```

## Starting Minicom Directly

- Open a device and choose its baud rate from the command line:

```sh
minicom -D /dev/ttyUSB0 -b 115200
```

- A common embedded Linux serial-console configuration is:

```txt
device:       /dev/ttyUSB0
baud rate:    115200
data bits:    8
parity:       none
stop bits:    1
flow control: none
```

- This format is commonly abbreviated as:

```txt
115200 8N1
```

## Configuring Minicom

- Start the setup menu:

```sh
sudo minicom -s
```

- Open `Serial port setup`.
- Configure:
  - serial device
  - baud rate
  - parity
  - data bits
  - stop bits
  - hardware flow control
  - software flow control
- Embedded development boards commonly require both hardware and software flow control to be disabled.
- Save the configuration as the default if it should be reused.
- After the configuration is saved, normal sessions should usually run without `sudo`:

```sh
minicom
```

## Minicom Command Key

- Minicom normally uses `Ctrl-A` as its command prefix.
- Press and release `Ctrl-A`, then press the command key.
- Do not hold every key simultaneously.

| Keys | Action |
|---|---|
| `Ctrl-A`, then `Z` | show command help |
| `Ctrl-A`, then `O` | open configuration |
| `Ctrl-A`, then `X` | exit and reset |
| `Ctrl-A`, then `Q` | quit without reset |
| `Ctrl-A`, then `E` | toggle local echo |
| `Ctrl-A`, then `L` | toggle capture-file logging |
| `Ctrl-A`, then `W` | toggle line wrapping |
| `Ctrl-A`, then `S` | send a file |
| `Ctrl-A`, then `R` | receive a file |

## Local Echo

- Local echo controls whether Minicom displays characters typed locally.
- Enable it when:
  - typed characters are accepted by the target but do not appear on the screen
- Disable it when:
  - each typed character appears twice
- The target system may already echo received characters, so local echo is often disabled for a Linux serial console.

## Capturing A Serial Log

- Start Minicom with a capture file:

```sh
minicom -D /dev/ttyUSB0 -b 115200 -C serial.log
```

- Or toggle capture while running with:

```txt
Ctrl-A, then L
```

- Logging is useful for:
  - bootloader output
  - kernel boot messages
  - crash records
  - test evidence

## Common Troubleshooting

- Permission denied:
  - inspect device ownership
  - add the user to the appropriate serial-device group
  - log in again

- Device is busy:
  - close another `minicom`, `screen`, or serial-monitor process
  - check whether a service such as a serial getty owns the port

- Garbled characters:
  - verify the baud rate
  - verify data bits, parity, and stop bits
  - confirm both sides use the same settings

- No output:
  - verify the correct device path
  - check the cable and TX/RX connection
  - press Enter
  - reset or power-cycle the target
  - disable flow control if the target does not use it

- Typed characters are invisible:
  - toggle local echo with `Ctrl-A`, then `E`

- Typed characters appear twice:
  - disable local echo

## Relationship To Raw And Canonical Modes

- `minicom` configures the local serial TTY for interactive byte-oriented communication.
- Raw and canonical processing belong to the TTY/line-discipline layer.
- Their driver-oriented behavior remains documented in:
  - [How Linux Device Nodes Reach A Driver](../../_posts/how-linux-device-nodes-reach-a-driver)

- This note focuses on operating terminal tools rather than implementing the underlying character driver.

## Summary

- A terminal emulator displays text, while a shell interprets commands.
- `TIOCGWINSZ`, `stty`, and `tput` can report terminal dimensions.
- `!:3` is Bash history expansion for the third argument of the previous command.
- GNU `screen` is useful for a quick serial connection.
- `minicom` provides serial configuration, logging, file transfer, local echo, and interactive command controls.
- Match the serial device, baud rate, framing, and flow-control settings on both sides.
- Fix serial-device permissions through the appropriate user group instead of relying on `sudo` for every session.

## References

- [Let's code a Linux Driver YouTube playlist by Johannes 4GNU_Linux](https://www.youtube.com/watch?v=DZrb9oSEzlU&list=PLCGpd0Do5-I3b5TtyqeF1UdyD4C-S-dMa&index=2)
- [Minicom manual page](https://man7.org/linux/man-pages/man1/minicom.1.html)
- [Linux TTY ioctl documentation](https://docs.kernel.org/userspace-api/ioctl/tty.html)
- [Bash history interaction](https://www.gnu.org/software/bash/manual/html_node/Using-History-Interactively.html)

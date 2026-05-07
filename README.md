# xv6 Operating System - Enhanced Edition

[![License](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Build](https://img.shields.io/badge/Build-Passing-brightgreen.svg)]()
[![Platform](https://img.shields.io/badge/Platform-x86-lightgrey.svg)]()

A modified version of the xv6 operating system with advanced scheduling and inter-process communication features. This project implements **Lottery Scheduling** and **Mailbox IPC** mechanisms, enhancing xv6's capabilities for educational and research purposes.

---

## 📋 Table of Contents

- [About xv6](#about-xv6)
- [Project Overview](#project-overview)
- [Key Features](#key-features)
- [Modifications](#modifications)
- [Building and Running](#building-and-running)
- [Testing](#testing)
- [Test Results](#test-results)
- [Documentation](#documentation)
- [Acknowledgments](#acknowledgments)

---

## 🔍 About xv6

xv6 is a re-implementation of Dennis Ritchie's and Ken Thompson's Unix Version 6 (v6). It loosely follows the structure and style of v6 but is implemented for a modern x86-based multiprocessor using ANSI C. This educational operating system is used to teach operating system concepts at MIT and other universities worldwide.

**Note:** The original xv6 team has shifted focus to the RISC-V version. This project is based on the x86 version.

---

## 🎯 Project Overview

This project extends xv6 with two major enhancements:

### 1. **Lottery Scheduler**
Replaces xv6's default round-robin scheduler with a **proportional-share CPU scheduling algorithm**. Processes are assigned "lottery tickets," and the scheduler randomly selects a winning ticket at each scheduling decision. Processes with more tickets receive proportionally more CPU time.

### 2. **Mailbox IPC**
Introduces a **message-oriented inter-process communication** mechanism that provides atomic message passing between processes through dedicated channels. This offers a lightweight alternative to traditional pipe-based communication.

---

## ✨ Key Features

### Lottery Scheduler Features
- ✅ **Proportional-share scheduling** - CPU time allocated based on ticket count
- ✅ **Dynamic priority support** - Processes can adjust their ticket allocation
- ✅ **Starvation prevention** - Even low-ticket processes get scheduled
- ✅ **Custom PRNG** - Linear Feedback Shift Register (LFSR) for efficient randomization
- ✅ **Performance tracking** - `getrunticks()` system call for CPU time measurement

### Mailbox IPC Features
- ✅ **16 independent channels** - Mailboxes 0-15 for multiplexed communication
- ✅ **Atomic message delivery** - Guaranteed discrete message boundaries
- ✅ **Blocking semantics** - Automatic sleep/wakeup synchronization
- ✅ **128-byte messages** - Fixed-size message buffer per mailbox
- ✅ **Thread-safe** - Spinlock protection for concurrent access
- ✅ **Simple API** - `ksend()` and `krecv()` system calls

---

## 🔧 Modifications

### System Calls Added

| System Call | Description | Parameters |
|------------|-------------|------------|
| `settickets(pid, tickets)` | Assign lottery tickets to a process | `pid`: Process ID, `tickets`: Number of tickets |
| `getrunticks()` | Get cumulative CPU time in ticks | None |
| `yield()` | Voluntarily relinquish CPU | None |
| `ksend(channel, msg, len)` | Send message on mailbox channel | `channel`: 0-15, `msg`: Message buffer, `len`: 1-128 bytes |
| `krecv(channel, buf, maxlen)` | Receive message from mailbox | `channel`: 0-15, `buf`: Receive buffer, `maxlen`: Buffer size |

### Files Modified

#### Core Kernel Files
- **`proc.h`** - Added `tickets`, `preempted`, and `runticks` fields to `struct proc`
- **`proc.c`** - Implemented lottery scheduler algorithm and `settickets()` function
- **`trap.c`** - Modified timer interrupt to track CPU time and preemption
- **`main.c`** - Added mailbox initialization at boot

#### System Call Infrastructure
- **`syscall.h`** - Added system call numbers (22-26)
- **`syscall.c`** - Registered new system calls in dispatch table
- **`sysproc.c`** - Implemented system call wrappers
- **`usys.S`** - Added assembly stubs for user-space calls
- **`user.h`** - Added user-space function declarations

#### New Files Created
- **`rand.c`** / **`rand.h`** - Pseudo-random number generator (LFSR)
- **`mailbox.c`** / **`mailbox.h`** - Mailbox IPC implementation
- **`schedtest.c`** - Comprehensive lottery scheduler test suite
- **`testmailbox.c`** - Mailbox IPC test suite

### Lottery Scheduler Algorithm

```c
// Count total tickets
int total_tickets = 0;
for(p = ptable.proc; p < &ptable.proc[NPROC]; p++) {
  if(p->state == RUNNABLE)
    total_tickets += p->tickets;
}

// Pick winning ticket
long winner = random_at_most(total_tickets - 1);

// Find winner
int counter = 0;
for(p = ptable.proc; p < &ptable.proc[NPROC]; p++) {
  if(p->state != RUNNABLE) continue;
  counter += p->tickets;
  if(counter > winner) {
    // This process won the lottery
    run_process(p);
    break;
  }
}
```

### Mailbox Synchronization

```c
// ksend - Blocking send
acquire(&mailbox->lock);
while(mailbox->full)
  sleep(mailbox, &mailbox->lock);  // Block until empty
memmove(mailbox->msg, msg, len);
mailbox->full = 1;
wakeup(mailbox);  // Wake receivers
release(&mailbox->lock);

// krecv - Blocking receive
acquire(&mailbox->lock);
while(!mailbox->full)
  sleep(mailbox, &mailbox->lock);  // Block until message arrives
memmove(buf, mailbox->msg, mailbox->msglen);
mailbox->full = 0;
wakeup(mailbox);  // Wake senders
release(&mailbox->lock);
```

---

## 🚀 Building and Running

### Prerequisites
- x86 ELF toolchain (GCC)
- QEMU PC simulator
- Make

### Build Instructions

```bash
# Clone the repository
git clone <your-repo-url>
cd xv6-enhanced

# Build xv6
make

# Run in QEMU
make qemu

# Run in QEMU with no display (terminal only)
make qemu-nox
```

### For Non-x86 or Non-ELF Systems (e.g., macOS)

Install a cross-compiler GCC suite capable of producing x86 ELF binaries:

```bash
# Then build with:
make TOOLPREFIX=i386-jos-elf-
```

---

## 🧪 Testing

### Lottery Scheduler Tests

Run the comprehensive scheduler test suite:

```bash
# In xv6 shell
$ schedtest
```

**Test Suite Includes:**
- **Experiment 1:** Equal tickets (fairness test)
- **Experiment 2:** Weighted tickets (priority test)
- **Experiment 3:** CPU vs I/O vs Yield behavior
- **Experiment 4:** Starvation prevention test

### Mailbox IPC Tests

Run the mailbox test suite:

```bash
# In xv6 shell
$ testmailbox
```

**Test Suite Includes:**
- **Test 1:** Basic send/receive
- **Test 2:** Blocking receive behavior
- **Test 3:** Multiple independent channels
- **Test 4:** Ping-pong bidirectional communication
- **Test 5:** Invalid argument handling
- **Test 6:** Maximum message size (128 bytes)

---

## 📊 Test Results

### Lottery Scheduler Results

#### Experiment 1: Equal Tickets (Fairness)
8 processes with 10 tickets each - demonstrates fair CPU distribution

![Lottery Test Comparison](Test%20Results/Lottery%20Test%20Comparison%20Table%20with%20default.jpeg)

#### Experiment 2-4: Scheduler Behavior
Various configurations testing priority, mixed workloads, and starvation prevention

<table>
<tr>
<td><img src="Test Results/Schedtest Exp1.jpeg" alt="Schedtest Experiment 1" width="400"/></td>
<td><img src="Test Results/Schedtest Exp2.jpeg" alt="Schedtest Experiment 2" width="400"/></td>
</tr>
<tr>
<td><img src="Test Results/Schedtest Exp3.jpeg" alt="Schedtest Experiment 3" width="400"/></td>
<td><img src="Test Results/Schedtest Exp4.jpeg" alt="Schedtest Experiment 4" width="400"/></td>
</tr>
</table>

### Mailbox IPC Results

Comprehensive testing of all mailbox features including blocking, channels, and edge cases

<table>
<tr>
<td><img src="Test Results/Test MailBox Exp1.png" alt="Mailbox Test 1" width="400"/></td>
<td><img src="Test Results/Test MailBox Exp2.png" alt="Mailbox Test 2" width="400"/></td>
</tr>
<tr>
<td><img src="Test Results/Test MailBox Exp3.png" alt="Mailbox Test 3" width="400"/></td>
<td><img src="Test Results/Test MailBox Exp4.png" alt="Mailbox Test 4" width="400"/></td>
</tr>
<tr>
<td><img src="Test Results/Test MailBox Exp5.png" alt="Mailbox Test 5" width="400"/></td>
<td><img src="Test Results/Test MailBox Exp6.png" alt="Mailbox Test 6" width="400"/></td>
</tr>
</table>

---

## 📚 Documentation

Comprehensive documentation is available in the repository:

- **[AUDIT_REPORT.md](AUDIT_REPORT.md)** - Complete audit of implementation integrity
- **[PROJECT_REPORT.md](PROJECT_REPORT.md)** - Detailed project report with evaluation
- **[lottery_scheduling_modifications.md](lottery_scheduling_modifications.md)** - Lottery scheduler deep dive
- **[mailbox_ipc_explanation.md](mailbox_ipc_explanation.md)** - Mailbox IPC concept and implementation
- **[TESTING_GUIDE.md](TESTING_GUIDE.md)** - Testing procedures and guidelines

---

## 🎓 Acknowledgments

### Original xv6 Team
xv6 is inspired by John Lions's Commentary on UNIX 6th Edition and developed by:
- Frans Kaashoek
- Robert Morris
- Russ Cox

### Code Sources
xv6 borrows code from:
- JOS (asm.h, elf.h, mmu.h, bootasm.S, ide.c, console.c, and others)
- Plan 9 (entryother.S, mp.h, mp.c, lapic.c)
- FreeBSD (ioapic.c)
- NetBSD (console.c)

### Contributors
Special thanks to the numerous contributors who have provided bug reports and patches to the xv6 project.

---

## 📄 License

The code in the files that constitute xv6 is Copyright 2006-2018 Frans Kaashoek, Robert Morris, and Russ Cox.

Modifications for lottery scheduling and mailbox IPC are provided for educational purposes.

---

## 🔗 Resources

- [Original xv6 Repository](https://github.com/mit-pdos/xv6-public)
- [xv6 RISC-V Version](https://github.com/mit-pdos/xv6-riscv)
- [MIT 6.828 Course](https://pdos.csail.mit.edu/6.828/)
- [Lions' Commentary on UNIX](https://en.wikipedia.org/wiki/Lions%27_Commentary_on_UNIX_6th_Edition)

---

## 📧 Contact

For questions or issues related to the lottery scheduler and mailbox IPC implementations, please open an issue in this repository.

---

**Built with ❤️ for Operating Systems Education**

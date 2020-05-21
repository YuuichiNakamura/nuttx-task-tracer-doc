NuttX Task Tracer
=================

# Overview

NuttX Task Tracer is the tool to collect the various events in the NuttX kernel and display the result graphically.

It can collect the following events.

- Task execution, termination, switching
- Entering to and leaving from the system call
- Entering to and leaving from the interrupt handler
- User defined message embedded in the user application

# Preparation

## Install Trace Compass

Task Tracer uses the external tool "Trace Compass" to display the trace result.

Download it from https://www.eclipse.org/tracecompass/ and install into the host environment.

After the installation, execute it and choose `Tools` -> `add-ons` menu, then select `Install Extensions` to install the extension named "Trace Compass ftrace (Incubation)".

## NuttX kernel configuration

To enable the task trace function, NuttX kernel configuration needs to be modified.
Enable the following options by menuconfig.

- `RTOS Features  --->`
  - `Performance Monitoring  --->`
    - Set check `[*]` to the following items.
      - `System performance monitor hooks`
      - `System call monitor hooks`
      - `Interrupt handler monitor hooks`
      - `Scheduler tracer`
    - Specify adequate trace buffer size to `Task trace buffer size`.
    - Specify task name buffer size to `Task trace name buffer size`.
- `Device Drivers  --->`
  - `System Logging  --->`
    - Set check `[*]` to `Scheduler tracer driver`.

After the configuration, rebuild the NuttX kernel and application.

If the trace function is enabled, "trace" command is available in NuttShell.

# How to get trace data

The trace function can be controlled by NuttShell "trace" command.

## Quick Guide

Trace can be started by the following command.
```
nsh> trace start
```

Trace can be stopped by the following command.
```
nsh> trace stop
```

The trace result is accumulated in the memory.
After getting the trace, the following command can display the accumulated trace data to the console.
```
nsh> trace dump
```
By using the logging function of your terminal software, the trace result can be saved into the host environment and it can be used as the input for "Trace Compass".

The trace result can be stored into the file by using the following command.
```
nsh> trace dump <file name>
```
If the target has a storage and has a way to transfer the file to the host by using such as USB MSC, the trace result file also can be used as the input for "Trace Compass".

## Trace command description

### **trace start**
Start task tracing
```
trace start [<duration>]
```
- `<duration>` : Specify the duration to trace by seconds.
Task tracing is stopped after the specified period.
If not specified, the tracing continues until stopped by the command.

### **trace stop**
Stop task tracing
```
trace stop
```

### **trace dump**
Output the trace result.
If running the task trace, it is stopped before the output.
```
trace dump [<filename>]
```
- `<filename>` : Specify the filename to save the trace result.
If not specified, the trace result is displayed to console.

### **trace cmd**
Get the trace while running the specified command.
After the termination of the command, task tracing is stopped.
```
trace cmd "<command>"
```
- `<command>` : Specify nsh command line to get the task trace.
It must be enclosed by double quotation if the command has arguments.

Example:
```
nsh> trace cmd "sleep 1"
```

### **trace mode**
Set the task trace mode options.
By default, all options are disabled.
```
trace mode [{+|-}{o|s|a|i}...]
```
- `+o` : Enable one-shot mode.
The trace buffer is used as the ring buffer by default, and the old data is overwritten when whole trace buffer is used.
If one-shot mode is enabled, the task trace is stoppedn whe whole buffer is used.
It can be used to get the trace of application boot.

- `-o` : Disable one-shot mode.

- `+s` : Enable system call trace.
It records the event of entering to and leaving from the system call which is issued by the application.
All system calls are recorded by default. `trace syscall` command can filter the system calls to be recorded.

- `-s` : Disable system call trace.

- `+a` : Enable recording the system call arguments.
It records the arguments passed to the issued system call to the trace data.

- `-a` : Disable recoding the system call arguments.

- `+i` : Enable interrupt trace.
It records the event of entering to and leaving from the interrupt handler which is occured while the tracing.
All IRQs are recorded by default. `trace irq` command can filter the IRQs to be recorded.

- `-i` : Disable interrupt trace.

If no command parameters are given, display the current mode as the follows.

Example:
```
nsh> trace mode
Task trace mode:
 Oneshot                 : off (-o)
 Syscall trace           : on  (+s)
  Filtered Syscalls      : 16
 Syscall trace with args : on  (+a)
 IRQ trace               : on  (+i)
  Filtered IRQs          : 2
```

### **trace syscall**
Configure the filter of system call trace.
```
trace syscall [{+|-}<syscallname>...]
```
- `+<syscallname>` : Add the specified system call name to the filter.
The execution of the filtered system call is not recorded into the trace data.<p>
Wildcard "`*`" can be used to specify the system call name.
For example, "`trace syscall +sem_*`" filters the system calls begin with "`sem_`", such as `sem_post()` and `sem_wait()`.

- `-<syscallname>` : Remove the specified system call name from the filter.
Wildcard "`*`" can be used to specify the system call name.

If the parameter is not specified, display the current filter settings as the follows.

Example:
```
nsh> trace syscall
Filtered Syscalls: 16
  getpid
  sem_destroy
  sem_post
  sem_timedwait
  sem_trywait
  sem_wait
  mq_close
  mq_getattr
  mq_notify
  mq_open
  mq_receive
  mq_send
  mq_setattr
  mq_timedreceive
  mq_timedsend
  mq_unlink
```

### **trace irq**
Configure the filter of interrupt trace.
```
trace irq [{+|-}<irqnum>...]
```
- `+<irqnum>` : Add the specified IRQ number to the filter.
The execution of the filtered IRQ handler is not recorded into the trace data.

- `-<irqnum>` : Remove the specified IRQ number from the filter.

If the parameter is not specified, display the current filter settings as the follows.

Example:
```
nsh> trace irq
Filtered IRQs: 2
  11
  15
```

### **Recording user defined message**

User message can be recorded into the trace data by writing the message to `/dev/tracer/`.

Example:
```
nsh> echo "message" >/dev/tracer
```

# Display the trace result

To display the trace result by "Trace Compass", choose `File` -> `Open Trace` menu to specify the trace data file name.

# Trace Control APIs

Application can control the trace features directly by using the trace control APIs.

To use the APIs, include the following file.
```
#include <nuttx/sched_tracer.h>
```

## API description

### **sched_tracer_start**
Start task tracing.
(Same as TRIOC_START ioctl)
```
#include <nuttx/sched_tracer.h>

void sched_tracer_start(void);
```
#### Input Parameters
- None

#### Returned Value
- None

### **sched_tracer_stop**
Stop task tracing.
(Same as TRIOC_STOP ioctl)
```
#include <nuttx/sched_tracer.h>

void sched_tracer_stop(void);
```
#### Input Parameters
- None

#### Returned Value
- None

### **sched_tracer_mode**
Set and get task trace mode.
(Same as TRIOC_GETMODE / TRIOC_SETMODE ioctls)
```
#include <nuttx/sched_tracer.h>

struct tracer_mode_s *sched_tracer_mode(struct tracer_mode_s *newm);
```

#### Input Parameters
- `newm` : A read-only pointer to `struct tracer_mode_s` which holds the new trace mode.
```
struct tracer_mode_s
{
  unsigned int flag;

#define TRIOC_MODE_FLAG_ONESHOT      (1 << 0)
#define TRIOC_MODE_FLAG_SYSCALL      (1 << 1)
#define TRIOC_MODE_FLAG_SYSCALL_ARGS (1 << 2)
#define TRIOC_MODE_FLAG_IRQ          (1 << 3)
};
```

#### Returned Value
- A pointer to `struct tracer_mode_s` of the current trace mode.

### **sched_tracer_syscallfilter**
Set and get syscall trace filter setting.
(Same as TRIOC_GETSYSCALLFILTER / TRIOC_SETSYSCALLFILTER ioctls)
```
#include <nuttx/sched_tracer.h>

ssize_t sched_tracer_syscallfilter(struct tracer_syscallfilter_s *oldf,
                                   struct tracer_syscallfilter_s *newf);
```
#### Input Parameters
- `oldf` : A writable pointer to `struct tracer_syscallfilter_s` to get current syscall trace filter setting. If 0, no data is written.
- `newf` : A read-only pointer to `struct tracer_syscallfilter_s` of the new syscall trace filter setting. If 0, the setting is not updated.
```
struct tracer_syscallfilter_s
{
  int nr_syscalls;
  int syscall[0];
};
```

#### Returned Value
- The API returns required size of `struct tracer_syscallfilter_s` to get current setting.
- Because the required size depends on the number of filtererd system calls, please follow the steps below to obtain the settings.
  1. Get the required size by calling the API with setting 0 to `oldf`.
  2. Allocate the required size memory.
  3. Call the API by setting the allocated memory to `oldf`.

### **sched_tracer_irqfilter**
Set and get IRQ trace filter setting.
(Same as TRIOC_GETIRQFILTER / TRIOC_SETIRQFILTER ioctls)
```
#include <nuttx/sched_tracer.h>

ssize_t sched_tracer_irqfilter(struct tracer_irqfilter_s *oldf,
                               struct tracer_irqfilter_s *newf);
```
#### Input Parameters
- `oldf` : A writable pointer to `struct tracer_irqfilter_s` to get current IRQ trace filter setting. If 0, no data is written.
- `newf` : A read-only pointer to `struct tracer_irqfilter_s` of the new IRQ trace filter setting. If 0, the setting is not updated.
```
struct tracer_irqfilter_s
{
  int nr_irqs;
  int irq[0];
};
```

#### Returned Value
- The API returns required size of `struct tracer_irqfilter_s` to get current setting.
- Because the required size depends on the number of filtererd IRQs, please follow the steps below to obtain the settings.
  1. Get the required size by calling the API with setting 0 to `oldf`.
  2. Allocate the required size memory.
  3. Call the API by setting the allocated memory to `oldf`.

### **sched_tracer_message**
Record the user provided message as the trace event.
```
#include <nuttx/sched_tracer.h>

void sched_tracer_message(FAR const char *fmt, ...);
```
#### Input Parameters
- `fmt` : printf() style format string. The following arguments are recorded by printf() style.

#### Returned Value
- None

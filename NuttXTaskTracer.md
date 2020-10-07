NuttX Task Tracer User Guide
============================

# Installation

## Install Trace Compass

Task Tracer uses the external tool "Trace Compass" to display the trace result.

Download it from https://www.eclipse.org/tracecompass/ and install into the host environment.

After the installation, execute it and choose `Tools` -> `add-ons` menu, then select `Install Extensions` to install the extension named "Trace Compass ftrace (Incubation)".

## NuttX kernel configuration

To enable the task trace function, the NuttX kernel configuration needs to be modified.

The following configurations must be enabled.

- `CONFIG_SCHED_INSTRUMENTATION`
- `CONFIG_SCHED_INSTRUMENTATION_FILTER`
- `CONFIG_SCHED_INSTRUMENTATION_SYSCALL`
- `CONFIG_SCHED_INSTRUMENTATION_IRQHANDLER`
- `CONFIG_DRIVER_NOTE`
- `CONFIG_DRIVER_NOTERAM`
- `CONFIG_DRIVER_NOTECTL`
- `CONFIG_SYSTEM_TRACE`
- `CONFIG_SYSTEM_SYSTEM`

The following configurations are configurable parameters for trace. 

- `CONFIG_SCHED_INSTRUMENTATION_FILTER_DEFAULT_MODE`
  - Specify the default filter mode.
    If the following bits are set, the corresponding instrumentations are enabled on boot.
    - Bit 0 = Enable instrumentation
    - Bit 1 = Enable syscall instrumentation
    - Bit 2 = Enable IRQ instrumentation

- `CONFIG_SCHED_INSTRUMENTATION_NOTERAM_BUFSIZE`
  - Specify the note buffer size in bytes.
    Higher value can hold more note records, but consumes more kernel memory.

- `CONFIG_SCHED_INSTRUMENTATION_NOTERAM_DEFAULT_NOOVERWRITE`
  - If enabled, stop overwriting old notes in the circular buffer when the buffer is full by default.
    This is useful to keep instrumentation data of the beginning of a system boot.

After the configuration, rebuild the NuttX kernel and application.

If the trace function is enabled, "`trace`" built-in command is available.

# How to get trace data

The trace function can be controlled by "`trace`" command.

## Quick Guide

### Getting the trace

Trace is started by the following command.
```
nsh> trace start
```

Trace is stopped by the following command.
```
nsh> trace stop
```

If you want to get the trace while executing some command, the following command can be used.
```
nsh> trace cmd <command> [<args>...]
```

### Displaying the trace result

The trace result is accumulated in the memory.
After getting the trace, the following command displays the accumulated trace data to the console.
```
nsh> trace dump
```
This will be get the trace results like the followings:
```
<noname>-1   [0]   7.640000000: sys_close()
<noname>-1   [0]   7.640000000: sys_close -> 0
<noname>-1   [0]   7.640000000: sys_sched_lock()
<noname>-1   [0]   7.640000000: sys_sched_lock -> 0
<noname>-1   [0]   7.640000000: sys_nxsched_get_stackinfo()
<noname>-1   [0]   7.640000000: sys_nxsched_get_stackinfo -> 0
<noname>-1   [0]   7.640000000: sys_sched_unlock()
<noname>-1   [0]   7.640000000: sys_sched_unlock -> 0
<noname>-1   [0]   7.640000000: sys_clock_nanosleep()
<noname>-1   [0]   7.640000000: sched_switch: prev_comm=<noname> prev_pid=1 prev_state=S ==> next_comm=<noname> next_pid=0
<noname>-0   [0]   7.640000000: irq_handler_entry: irq=11
<noname>-0   [0]   7.640000000: irq_handler_exit: irq=11
<noname>-0   [0]   7.640000000: irq_handler_entry: irq=15
<noname>-0   [0]   7.650000000: irq_handler_exit: irq=15
<noname>-0   [0]   7.650000000: irq_handler_entry: irq=15
    :
```

By using the logging function of your terminal software, the trace result can be saved into the host environment and it can be used as the input for "Trace Compass".

If the target has a storage, the trace result can be stored into the file by using the following command.
It also can be used as the input for "Trace Compass" by transferring the file in the target device to the host.
```
nsh> trace dump <file name>
```

To display the trace result by "Trace Compass", choose `File` -> `Open Trace` menu to specify the trace data file name.



![Trace Compass screenshot](image/trace-compass-screenshot.png)


## Trace command description

### **trace start**
Start task tracing
```
trace start [<duration>]
```
- `<duration>` : Specify the duration of the trace by seconds.
Task tracing is stopped after the specified period.
If not specified, the tracing continues until stopped by the command.

### **trace stop**
Stop task tracing
```
trace stop
```

### **trace cmd**
Get the trace while running the specified command.
After the termination of the command, task tracing is stopped.
To use this command, `CONFIG_SYSTEM_SYSTEM` needs to be enabled.

```
trace cmd <command> [<args>...]
```
- `<command>` : Specify the command to get the task trace.
- `<args>` : Arguments for the command.

Example:
```
nsh> trace cmd sleep 1
```

### **trace dump**
Output the trace result.
If the task trace is running, it is stopped before the output.
```
trace dump [<filename>]
```
- `<filename>` : Specify the filename to save the trace result.
If not specified, the trace result is displayed to console.


### **trace mode**
Set the task trace mode options.
The default value is given by the kernel configuration `CONFIG_SCHED_INSTRUMENTATION_FILTER_DEFAULT_MODE`.
```
trace mode [{+|-}{o|s|i}...]
```
- `+o` : Enable overwrite mode.
The trace buffer is a ring buffer and it can overwrite old data if no free space is available in the buffer.
Enables this behavior.

- `-o` : Disable overwrite mode.
The new trace data will be disposed when the buffer is full.
This is useful to keep the data of the beginning of the trace.

- `+s` : Enable system call trace.
It records the event of enter/leave system call which is issued by the application.
All system calls are recorded by default. `trace syscall` command can filter the system calls to be recorded.

- `-s` : Disable system call trace.

- `+i` : Enable interrupt trace.
It records the event of enter/leave interrupt handler which is occured while the tracing.
All IRQs are recorded by default. `trace irq` command can filter the IRQs to be recorded.

- `-i` : Disable interrupt trace.

If no command parameters are specified, display the current mode as the follows.

Example:
```
nsh> trace mode
Task trace mode:
 Trace                   : enabled
 Overwrite               : on  (+o)
 Syscall trace           : on  (+s)
  Filtered Syscalls      : 16
 IRQ trace               : on  (+i)
  Filtered IRQs          : 2
```

### **trace syscall**
Configure the filter of the system call trace.
```
trace syscall [{+|-}<syscallname>...]
```
- `+<syscallname>` : Add the specified system call name to the filter.
The execution of the filtered system call is not recorded into the trace data.<p>

- `-<syscallname>` : Remove the specified system call name from the filter.

Wildcard "`*`" can be used to specify the system call name.
For example, "`trace syscall +sem_*`" filters the system calls begin with "`sem_`", such as `sem_post()`, `sem_wait()`,...

If no command parameters are specified, display the current filter settings as the follows.

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
Configure the filter of the interrupt trace.
```
trace irq [{+|-}<irqnum>...]
```
- `+<irqnum>` : Add the specified IRQ number to the filter.
The execution of the filtered IRQ handler is not recorded into the trace data.

- `-<irqnum>` : Remove the specified IRQ number from the filter.

Wildcard "`*`" can be used to specify all IRQs.

If no command parameters are specified, display the current filter settings as the follows.

Example:
```
nsh> trace irq
Filtered IRQs: 2
  11
  15
```

# Related devices and ioctls

## Devices

The following devices are used for the task trace.

- `/dev/notectl`
  - The device to control an instrumentation filter in NuttX kernel.
  - The device has only ioctl function to control the filter.

- `/dev/note`
  - The device to get the trace (instrumentation) data.
  - The device has read function to get the data and ioctl function to control the buffer mode.

## /dev/notectl Ioctls

The header file `<nuttx/note/notectl_driver.h>` is needed to use the following ioctls.

### **NOTECTL_GETMODE**
Get note filter mode

#### Argument
- A writable pointer to `struct note_filter_mode_s`
```
struct note_filter_mode_s
{
  unsigned int flag;          /* Filter mode flag */
#ifdef CONFIG_SMP
  unsigned int cpuset;        /* The set of monitored CPUs */
#endif  
};
```
- `flag` : Filter mode flag. The following defines are available.
  - `NOTE_FILTER_MODE_FLAG_ENABLE`
    - Enable instrumentation
  - `NOTE_FILTER_MODE_FLAG_SYSCALL`
    - Enable syscall instrumentation
  - `NOTE_FILTER_MODE_FLAG_IRQ`
    - Enable IRQ instrumentaiton
- `cpuset` : (SMP only) Monitor only CPUs in the bitset. Bit 0=CPU0, Bit1=CPU1, etc.

#### Return value
- If success, 0 is returned and current note filter mode is stored into the given pointer.
- If failed, the negative errno value is returned.

### **NOTECTL_SETMODE**
Set note filter mode

#### Argument
- A read-only pointer to `struct note_filter_mode_s`

#### Return value
- If success, 0 is returned and the given filter mode is set as the current settings.
- If failed, the negative errno value is returned.

### **NOTECTL_GETSYSCALLFILTER**
Get syscall filter setting

#### Argument
- A writable pointer to `struct note_filter_syscall_s`
```
struct note_filter_syscall_s
{
  uint8_t syscall_mask[];
};
```
- `syscall_mask` : A bitmap array of the syscall filter. If a bit is set, the corresponding syscall is not recorded.
  The following helper macros are available:
  - `NOTE_FILTER_SYSCALLMASK_SET(nr, s)`
    - Set syscall number `nr` as masked. `s` specifies the variable of `struct note_filter_syscall_s`
  - `NOTE_FILTER_SYSCALLMASK_CLR(nr, s)`
    - Set syscall number `nr` as unmasked.
  - `NOTE_FILTER_SYSCALLMASK_ISSET(nr, s)`
    - Check whether syscall number `nr` is masked or not. True if masked.
  - `NOTE_FILTER_SYSCALLMASK_ZERO(s)`
    - Clear all masks.

#### Return value
- If success, 0 is returned and current syscall filter mode is stored into the given pointer.
- If failed, the negative errno value is returned.

### **NOTECTL_SETSYSCALLFILTER**
Set syscall filter setting

#### Argument
- A read-only pointer to `struct note_filter_syscall_s`

#### Return value
- If success, 0 is returned and the given syscall filter mode is set as the current settings.
- If failed, the negative errno value is returned.

### **NOTECTL_GETIRQFILTER**
Get IRQ filter setting

#### Argument
- A writable pointer to `struct note_filter_irq_s`
```
struct note_filter_irq_s
{
  uint8_t irq_mask[];
};
```
- `irq_mask` : A bitmap array of the IRQ filter. If a bit is set, the corresponding IRQ is not recorded.
  The following helper macros are available:
  - `NOTE_FILTER_IRQMASK_SET(nr, s)`
    - Set IRQ number `nr` as masked. `s` specifies the variable of `struct note_filter_irq_s`
  - `NOTE_FILTER_IRQMASK_CLR(nr, s)`
    - Set IRQ number `nr` as unmasked.
  - `NOTE_FILTER_IRQMASK_ISSET(nr, s)`
    - Check whether IRQ number `nr` is masked or not. True if masked.
  - `NOTE_FILTER_IRQMASK_ZERO(s)`
    - Clear all masks.

#### Return value
- If success, 0 is returned and current IRQ filter mode is stored into the given pointer.
- If failed, the negative errno value is returned.

### **NOTECTL_SETIRQFILTER**
Set IRQ filter setting

#### Argument
- A read-only pointer to `struct note_filter_irq_s`

#### Return value
- If success, 0 is returned and the given IRQ filter mode is set as the current settings.
- If failed, the negative errno value is returned.


## /dev/note Ioctls

The header file `<nuttx/note/noteram_driver.h>` is needed to use the following ioctls.

### **NOTERAM_CLEAR**
Clear all contents of the circular buffer

#### Argument
- Ignored

#### Return value
- Always returns 0.

### **NOTERAM_GETMODE**
Get overwrite mode

#### Argument
- A writable pointer to `unsigned int`
- The overwrite mode takes one of the following values.
  - `NOTERAM_MODE_OVERWRITE_DISABLE`
    - Overwrite mode is disabled. When the buffer is full, accepting the data will be stopped.
  - `NOTERAM_MODE_OVERWRITE_ENABLE`
    - Overwrite mode is enabled.
  - `NOTERAM_MODE_OVERWRITE_OVERFLOW`
    - Overwrite mode is disabled and the buffer is already full.

#### Return value
- If success, 0 is returned and current overwrite mode is stored into the given pointer.
settings.
- If failed, the negative errno value is returned.

### **NOTERAM_SETMODE**
Set overwrite mode

#### Argument
- A read-only pointer to `unsigned int`

#### Return value
- If success, 0 is returned and the given overwriter mode is set as the current settings.
- If failed, the negative errno value is returned.


# Filter control APIs

The following APIs are the functions to control note filters directly.
These are kernel APIs and application can use them only in FLAT build.

The header file `<nuttx/sched_note.h>` is needed to use the following APIs.

## API description

### **sched_note_filter_mode**
Set and get note filter mode.
(Same as `NOTECTL_GETMODE` / `NOTECTL_SETMODE` ioctls)
```
#include <nuttx/sched_note.h>

void sched_note_filter_mode(struct note_filter_mode_s *oldm,
                            struct note_filter_mode_s *newm);
```

#### Input Parameters
- `oldm` : A writable pointer to `struct note_filter_mode_s` to get current filter mode.
           If 0, no data is written.
- `newm` : A read-only pointer to `struct note_filter_mode_s` which holds the new filter mode.
           If 0, the filter mode is not updated.

#### Returned Value
- None

### **sched_note_filter_syscall**
Set and get syscall filter setting.
(Same as `NOTECTL_GETSYSCALLFILTER` / `NOTECTL_SETSYSCALLFILTER` ioctls)
```
#include <nuttx/sched_note.h>

void sched_note_filter_syscall(struct note_filter_syscall_s *oldf,
                               struct note_filter_syscall_s *newf);
```
#### Input Parameters
- `oldf` : A writable pointer to `struct note_filter_syscall_s` to get current syscall filter setting.
           If 0, no data is written.
- `newf` : A read-only pointer to `struct note_filter_syscall_s` of the new syscall filter setting.
           If 0, the setting is not updated.

#### Returned Value
- None

### **sched_note_filter_irq**
Set and get IRQ filter setting.
(Same as `NOTECTL_GETIRQFILTER` / `NOTECTL_SETIRQFILTER` ioctls)
```
#include <nuttx/sched_note.h>

void sched_note_filter_irq(struct note_filter_irq_s *oldf,
                           struct note_filter_irq_s *newf);
```
#### Input Parameters
- `oldf` : A writable pointer to `struct note_filter_irq_s` to get current IRQ filter setting.
           If 0, no data is written.
- `newf` : A read-only pointer to `struct note_filter_irq_s` of the new IRQ filter setting.
           If 0, the setting is not updated.

#### Returned Value
- None

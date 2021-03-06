# Chap 6 - Advanced Char Driver Operations
## Goal:
1. implement functionality more than synchronous read and write
2. handle concurrency issues
3. implement ```ioctl``` syscall - a common interface used for device control
4. synchronizing with user space
5. know how to put processes to sleep & to wake them up
6. implement nonblocking I/O
7. inform user space when your devices are available for reading or writing
8. implement a few different device access policies within drivers


## Device control with ioctl
### ioctl background
Most drivers need i) ability to read/write device and ii) perform various types of hardware control via device driver. User space must be able to request:
    
1. device lock its door
2. eject its media
3. report error info
4. change a baud rate
5. self destruct

The above operations are usually supported via ```ioctl``` method. In user space, it has the prototype of:
```
int ioctl(int fd, unsigned long cmd, ...);

// fd - file descriptor

/*
  The ioctl() system call manipulates the underlying device
       parameters of special files.  In particular, many operating
       characteristics of character special files (e.g., terminals) may
       be controlled with ioctl() requests.  
       
       The argument fd must be an open file descriptor.
       The second argument is a device-dependent request code.  
       The third argument is an untyped pointer to memory.  
       It's traditionally char *argp (from the days before void * was valid
       C), and will be so named for this discussion.

       An ioctl() request has encoded in it whether the argument is an
       in parameter or out parameter, and the size of the argument argp
       in bytes.  Macros and defines used in specifying an ioctl()
       request are located in the file <sys/ioctl.h>.
 */
```

Since a system call cannot have a variable number of arguments, the dots in the prototype represents a single **optional argument** - ```char *argp```. Such dots are used to prevent type checking during compilation.

The third argument depends on the **specific control command (2nd argument)** being issued. Such commands may take no arguments, or an integer value, or a pointer to other data. Using a pointer is the way to pass arbitrary data to the ioctl call; the device is then able to exchange any amount of data with user space.

As we can see, the ```ioctl```'s arguments are unstructured, which, makes it hard to make such arguments work identically on all systems. E.g. 64-bit systems with a userspace process running in 32-bit mode (haven't figured out why).

Hence, we need to implement miscellaneous control operations. There are two means:
1. embedding commands into the data stream (in this chapter)
2. using virtual filesystems, either sysfs or driverspecific filesystems (chapter 14).

### A small demonstration of ioctl
*Retrieved from https://stackoverflow.com/questions/15807846/ioctl-linux-device-driver#15856623*

A printer that has configuration options to check and set the font family. ```ioctl``` could be used to get the current font + set the font to a new one. A user application uses ```ioctl``` to send a code to a printer telling it to return the current font or to set the font to a new one.

Here's how a user program will call ```ioctl``` to interact with the printer (device):
```
int ioctl(int fd, int request, ...)
```
1. ```fd``` - file descriptor, returned by ```open```
2. ```request``` - request code, identify what action to do on such device
3. 3rd argument ```void *``` - depends on the 2nd argument, e.g. if the second argument is ```SETFONT```, the third argument can be the font name such as ```"Arial"```

A user application has to generate a **request code** and the **device driver module** to determine which configuration on device must be played with. In other words: 
1. the user application sends the **request code** using ```ioctl```
2. uses the **request code** in the device driver module to determine which action to perform.

As we have noticed, such **request code** is important, so we need to talk about it in details. Firstly, it has 4 main parts:
1. A Magic number - 8 bits:
    usually defined at the beginning, e.g. ```#define PRIN_MAGIC 'P'```
2. A sequence number - 8 bits:
    also called ordinal number
3. Argument type (typically 14 bits), if any:
    the magic number assiciated with the device
4. Direction of data transfer - 2 bits:
    e.g. if request code is ```SETFONT```, then the direction will be user application -> device driver module; if ```GETFONT```, then the direction will be reversed

*Now we have a brief understanding of the **request code***, but actually, we still need to figure out how to **generate** such code: *using predefined function-like macros in Linux* as follows:
1. ```_IO(type,nr)``` - for a command that has no argument
2. ```_IOR(type,nr,datatype)``` - for reading data from the driver
3. ```_IOW(type,nr,datatype)``` - for writing data
4. ```_IOWR(type,nr,datatype)``` - for bidirectional transfers

#### Example 1
Say, we would like to pause printer. Since pausing the printer does not require any data transfer, we could use ```_IO(type,nr)```, see the code below:
```
#define PRIN_MAGIC 'P' // define the magic number at the beginning
#define SEQ_NUM 0 // define the sequence number
#define PAUSE_PRIN _IO(PRIN_MAGIC, SEQ_NUM) // define the request code PAUSE_PRIN
                                            // using _IO(PRIN_MAGIC, NUM)
```
After we have done such things, user could use ```ioctl``` as:
```
ret_val = ioctl(fd, PAUSE_PRIN) // note that this ioctl takes two arguments
                                // because no extra arguments is needed here
```
Then, the corresponding syscall in the driver module will receive the code and pause the printer.

#### Example 2
Say, we would like to set the font to be Arial. The direction of data transfer will be: user application -> printer (device), and we would like to write to the device; hence, we have to use the predefined function-like macro - ```_IOW(type,nr,datatype)```. The code is as follows:
```
#define PRIN_MAGIC 'S' // define the magic number at the beginning (S = SET)
#define SEQ_NO 1 // define the sequence number as 1
#define SETFONT _IOW(PRIN_MAGIC, SEQ_NO, unsigned long) // define the request code
                                                        // SETFONT
```
```datatype``` here gives us the type of the **third argument** in ```ioctl```. Someone may be wondering why the type of which is ```unsigned long```. Recall that the prototype of ioctl is ```int ioctl(int fd, int request, ...)```, and the type of the optional 3rd argument is ```void *```, we need to interpret it to be an address, which is of type ```unsigned long```.

Now, the user could execute the following to interact with the printer (device):
```
char *font = "Arial";
ret_val = ioctl(fd, SETFONT, font); // font is a pointer, i.e. an address
```
The address of ```font``` is passed to corresponding syscall implemented in device driver module as ```unsigned long``` and we need to cast it to proper type before using it. Kernel space can access user space and hence this works. 


### ioctl driver method prototype
The ```ioctl``` driver method has a prototype of:
```
int (*ioctl) (struct inode *inode, struct file *filp,
                unsigned int cmd, unsigned long arg);

/*
 * @inode @filp: the values corresponding to the file descriptor fd passed on
 *               by the application and are the same parameters passed to the
 *               open method.
 * @cmd: passed from the user unchanged, same as ioctl() syscall
 *       type of which is int - a symbolic name, declared in header files
 * @arg: compiler will not warn user if this argument is not passed.
 */
```

### choosing the ioctl commands
Before writing the code for ```ioctl```, need to choose the numbers that correspond to commands.

The ioctl command numbers should be unique across the system in order to prevent errors caused by issuing the right command to the wrong device. Such a mismatch is not unlikely to happen, and a program might find itself trying to change the baud rate of a non-serial-port input stream, such as a FIFO or an audio device. If each ioctl number is unique, the application gets an EINVAL error rather than succeeding in doing something unintended. To help programmers create unique ioctl command codes, these codes have been split up into several bitfields.

Need to choose ```ioctl``` numbers for the driver according to the Linux kernel convention - first check *include/asm/ioctl.h* and *Documentation/ioctl-number.txt*. 

Such header defines the bitfields that will be using: 

1. ```type``` - 
    the magic number assiciated with the device

2. ```ordinal number``` - 
    eight bits (_IOC_NRBITS) wide

3. ```direction of transfer``` -
    possible values are _IOC_NONE (no data transfer), _IOC_READ, _IOC_WRITE, and _IOC_READ|_IOC_WRITE (data is transferred both ways)

4. ```size of argument``` - 
    the size of user data involved (can be found in the macro _IOC_SIZEBITS)

The *ioctl-number.txt* file lists the magic numbers used throughout the kernel, so we’ll be able to choose our own magic number and avoid overlaps. The text file also lists the reasons why the convention should be used.

The header file *<asm/ioctl.h>*, which is included by *<linux/ioctl.h>*, defines macros that help set up the **command numbers** as follows: 

1. ```_IO(type,nr)``` - for a command that has no argument
2. ```_IOR(type,nr,datatype)``` - for reading data from the driver
3. ```_IOW(type,nr,datatype)``` - for writing data
4. ```_IOWR(type,nr,datatype)``` - for bidirectional transfers

As we can see here, for the command that has no argument, there's no ```datatype``` field because we do not need to specify what type of data to read/write. 

The ```type``` and ```number (nr)``` fields are passed as arguments, and the size field is derived by applying sizeof to the datatype argument.

See *scull.h* in this directory for more detials of how *ioctl* commands are defined in scull. Such commands set and get the driver's configurable parameters.

### Using the ioctl argument
Remember that optional 3rd argument? If that is an integer, it would be easy to handle. However, if that is a pointer, be careful.

+ ```access_ok()``` - address verification
    + declared in *<asm/uaccess.h>*
    + ```int access_ok(int type, const void *addr, unsigned long size);```
    + ```type``` - either ```VERIFY_READ``` (reading the user-space memory area) or ```VERIFY_WRITE``` (writing to user-space memory area)
    + ```addr``` - user-space address
    + ```size``` - byte count. If ```ioctl``` needs to read an integer from user space, then ```size = sizeof(int)```. If both read and write are needed, use ```VERIFY_WRITE``` (superset of ```VERIFY_READ```).
    + return value: 1 on success, 0 on failure (then the driver should return ```-EFAULT```)
    + **only ensures that such address does not point to kernel-space memory.**

Now, let's see how ```access_ok()``` is used:
```
int err = 0;
int retval = 0;
int tmp;

/*
 * extract the type and number bitfields, and don't decode
 * wrong cmds: return ENOTTY (inappropriate ioctl) before access_ok()
 */
 if (_IOC_TYPE(cmd) != SCULL_IOC_MAGIC) return -ENOTTY;
 if (_IOC_NR(cmd) > SCULL_IOC_MAXNR) return -ENOTTY;

/*
 * the direction is a bitmask, and VERIFY_WRITE catches R/W transfers. 
 * `Type' is user-oriented, but access_ok is kernel-oriented, 
 * so the concept of "read" and "write" is reversed
 */
if (_IOC_DIR(cmd) & _IOC_READ)
    err = !access_ok(VERIFY_WRITE, (void __user *)arg, _IOC_SIZE(cmd));
else if (_IOC_DIR(cmd) & _IOC_WRITE)
    err = !access_ok(VERIFY_READ, (void __user *)arg, _IOC_SIZE(cmd));

if (err) return -EFAULT;
```

After calling ```access_ok```, the driver can safely perform the actual transfer. Besides ```copy_from_user``` and ```copy_to_user```, we could use other functions that are optimized for the most used data sizes (1,2,4,8 bytes). Such functions are defined in *<asm/uaccess.h>*. 
+ ```put_user(datum, ptr)``` and ```__put_user(datum, ptr)```
+ ```get_user(local, ptr)``` and ```__get_user(local, ptr)```

More details in p. 143 of LDD3

### Capabilities and Restricted Operations
This section mainly talks about permissions to access a device. Linux kernel provides a mechanism called *capabilities*, s.t. a user application can be empowered to perform a specific privileged operation without giving away the ability to perform other unrelated operations. Two syscalls used for permission management are ```capget```, and ```capset```. Hence, can be managed from user space.

Full set of capabilities can be found in *<linux/capability.h>*, including:
- CAP_DAC_OVERRIDE
- CAP_NET_ADMIN
- CAP_SYS_MODULE
- CAP_SYS_RAWIO
- CAP_SYS_ADMIN
- CAP_SYS_TTY_CONFIG
- See p. 144 for more details

Before performing a privileged operation, a device driver should check that the calling process has the appropriate capability. Capability checkes are performed with the ```capable``` function defined in *<linux/sched.h>*:
```
int capable(int capability); // capability has been discussed above
```
In *scull* driver, any user is allowed to query the quantum and quantum set sizes, but only privileged users may change those values. When needed, we could do as follows:
```
if(!capable(CAP_SYS_ADMIN))
    return -EPERM; // Error PERMission
```

### Implementation of ioctl Commands
Now, we could take a look at the implementation of ioctl Commands. The scull implementation of ioctl only transfers the configurable parameters of the device.
```
switch(cmd) {
    case SCULL_IOCRESET:
        scull_quantum = SCULL_QUANTUM;
        scull_qset = SCULL_QSET;
        break;

    /* Set: arg points to the value */
    case SCULL_IOCSQUANTUM: 
        if (! capable (CAP_SYS_ADMIN))
            return -EPERM;
        retval = __get_user(scull_quantum, (int __user *)arg);
        break;

    /* Tell: arg is the value */
    case SCULL_IOCTQUANTUM: 
        if (! capable (CAP_SYS_ADMIN))
            return -EPERM;
        scull_quantum = arg;
        break;

    /* Get: arg is pointer to result */
    case SCULL_IOCGQUANTUM: 
        retval = __put_user(scull_quantum, (int __user *)arg);
        break;

    /* Query: return it (it's positive) */
    case SCULL_IOCQQUANTUM: 
        return scull_quantum;

    /* eXchange: use arg as pointer */
    case SCULL_IOCXQUANTUM: 
        if (! capable (CAP_SYS_ADMIN))
            return -EPERM;
        tmp = scull_quantum;
        retval = __get_user(scull_quantum, (int __user *)arg);
        if (retval = = 0)
            retval = __put_user(tmp, (int __user *)arg);
        break;

    /* sHift: like Tell + Query */
    case SCULL_IOCHQUANTUM: 
        if (! capable (CAP_SYS_ADMIN))
            return -EPERM;
        tmp = scull_quantum;
        scull_quantum = arg;
        return tmp;

    /* redundant, as cmd was checked against MAXNR */
    default: 
        return -ENOTTY;
}
return retval;
```

In scull's implementation, it also includes 6 similar entries that act on ```scull_qset```.

*Now, let's take a look from the user's point of view: how to pass and recieve arguments?*
```
int quantum;

ioctl(fd,SCULL_IOCSQUANTUM, &quantum); /* Set by pointer */
ioctl(fd,SCULL_IOCTQUANTUM, quantum); /* Set by value */

ioctl(fd,SCULL_IOCGQUANTUM, &quantum); /* Get by pointer */
quantum = ioctl(fd,SCULL_IOCQQUANTUM); /* Get by return value */

ioctl(fd,SCULL_IOCXQUANTUM, &quantum); /* Exchange by pointer */
quantum = ioctl(fd,SCULL_IOCHQUANTUM, quantum); /* Exchange by value */
```

The above is a demonstration of all calling modes, but in the real-world implementation, data exchanges would be consistently performed, i.e. either through pointers OR by value. AVOID using mixed of two.


## Device control w/o ioctl
Sometimes controlling the device is better accomplished by writing control sequences to the device itself. E.g. the console driver, where so-called escape sequences are used to move the cursor, change the default color, or perform other configuration tasks. 

**Benefit**: user can control the device just by writing data, without needing to use (or sometimes write) programs built just for configuring the device. When devices can be controlled in this manner, the program issuing commands often need not even be running on the same system as the device it is controlling. E.g. *setterm* program.

**Drawback** of controlling by printing: it adds policy constraints to the device; for example, it is viable only if you are sure that the control sequence can’t appear in the data being written to the device during normal operation. This is only partly true for ttys. Although a text display is meant to display only ASCII characters, sometimes control characters can slip through in the data being written and can, therefore, affect the console setup. This can happen, for example, when you cat a binary file to the screen; the resulting mess can contain anything, and you often end up with the wrong font on your console.

More details, see p. 146-147

## Blocking I/O
How does a driver respond if it cannot immediately satisfy the request? 

A call to read may come when no data is available, or a process could attempt to write, but your device is not ready to accept the data, because your output buffer is full. The calling process usually does not care about such issues; so, in such cases, your driver should (by default) block the process, putting it to sleep until the request can proceed.

Let's take a look at how to put a process to sleep and wake it up again later on.

### Sleeping
When a process is put to sleep, it is 
1. marked as being in a special state
2. removed from the scheduler's run queue
3. not scheduled on any CPU until sth. comes to change such state

Need to follow the following rules (before sleep):
+ never sleep when you are running in an atomic context
    + atomic operation discussed in Chap 5
    + a state where multiple steps must be performed without any sort of concurrent access
+ the driver cannot sleep while holding a a spinlock, seqlock, or RCU lock
+ the driver cannot sleep if interrupts are disabled
+ the driver can sleep while holding a semaphore (be careful: not blocking the process that will eventually wake you up)
+ unaware of changes or how long the sleep was

**How could we find the sleeping process?**

**wait queue** - a list of processes, all waiting for a specific event. 
- Defined in *<linux/wait.h>*. 
- Managed by struct ```wait_queue_head_t```
Statically define and initialize a wait queue:
```
DECLARE_WAIT_QUEUE_HEAD(name);
```
Dynamically define and initialize a wait queue:
```
wait_queue_head_t q;
DECLARE_WAIT_QUEUE_HEAD(&q);
```

**How to sleep in Linux?**
The simplest way is to use ```wait_event```, which combines sleeping with a check on the **condition** a process is waiting for.
```
// queue - wait queue head, passed by value.
// condition - to be checked before sleep, bollean expression evaluated by macro
//             will be evaluated an arbitrary number of times
wait_event(queue, condition) 
wait_event_interruptible(queue, condition)
wait_event_timeout(queue, condition, timeout)
wait_event_interruptible_timeout(queue, condition, timeout)
```

+ ```wait_event```: 
    + put process into an uninterruptible sleep, not what we want
+ ```wait_event_interruptible```: 
    + can be interrupted by signals, is what we want. 
    + in LDD3's implementation, returning a nonzero value - sleep was interrupted by some signal, driver may return ```-ERESTARTSYS```.
+ ```wait_event_timeout```:
    + put process into an uninterruptible sleep for limited time
    + in LDD3's implementation, return 0 regardless of how condition evaluates
+ ```wait_event_interruptible_timeout```:
    + in LDD3's implementation, return 0 regardless of how condition evaluates

**How to wakeup?**
```wake_up``` the process that ```wait_event```.
```wake_up_interruptible``` the process that ```wait_event_interruptible```.
```
void wake_up(wait_queue_head_t *queue); // wakeup all processes on wait queue
void wake_up_interruptible(wait_queue_head_t *queue); // wake up an interruptible sleep
```

### Simple implementation of sleeping
A simple sleep that implements a device with simple behavior: any process that attempts to read from the device is put to sleep. See *sleepy.c* for more details.
```
// method for read (put current process to sleep)
static ssize_t sleepy_read(struct file *filp, char __user *buf, 
                           size_t count, loff_t *pos)
{
    /**
     * Kernel code can refer to the current process by accessing the global item 
     * current, defined in <asm/current.h>, which yields a pointer to struct 
     * task_struct, defined by <linux/sched.h>.
     */ 
    printk(KERN_DEBUG "process %i (%s) going to sleep\n", 
                    current->pid, current->comm); 
                    // comm - executable name of process, excluding path, 
                    // in <linux/sched.h>
    
    // wait current process to wait queue wq if flag != 0
    wait_event_interruptible(wq, flag != 0);
    flag = 0;
    printk(KERN_DEBUG "awoken %i (%s)\n", current->pid, current->comm);
}


static ssize_t sleepy_write(struct file *filp, const char __user *buf, 
                            size_t count, loff_t *pos)
{
    printk(KERN_DEBUG "process %i (%s) awakening the readers...\n", 
                    current->pid, current->comm);
    flag = 1;
    wake_up_interruptible(&wq);
    return count; /* succeed, to avoid retrial */
}
```

**What happens if two processes are waiting when ```sleepy_write``` is called?**
As we can see from *sleepy.c*, ```sleepy_read``` will put ```current``` (process) into wait queue - blocked. When ```current``` is waken up, it continues from where it got blocked, i.e. resets ```flag``` to 0 once it wakes up. Then, the second process to wake up would immediately go back to sleep.

However, such **only works on a single-processor system**. Actually, the ```wake_up_interruptible``` call will cause both sleeping processes to wake up. It is entirely possible that they will both note that flag is nonzero before either has the opportunity to reset it.

Therefore, the implementation **needs to be modified**. Before that, we need to cover other topics.

### Blocking and Nonblocking Operations
Before fully implementing ```read``` and ```write```, we need to decide **when to put a process to sleep**.
- Sometimes, it's required that an operation should not block.
- Sometimes, the calling process may inform you that it does not want to block.

We need to implement the following behavior:
+ If a process calls read but no data is (yet) available, the process must block.
    + Such process is awakened as soon as some data arrives, and returned to the caller
+ If a process calls write and there is no space in the buffer, the process must block.
    + Such process must be on a different wait queue from the one used for reading.
    + When space is available in the output buffer, process is awakened.

Every device has *input buffers* and *output buffers*. The input buffer is required to avoid losing data that arrives when nobody is reading. In contrast, data can’t be lost on write, because if the system call doesn’t accept data bytes, they remain in the user-space buffer. Even so, the output buffer is almost always useful for **squeezing more performance out of the hardware**, how?
- reduce number of context switches 
- reduce number of user-level/kernel-level transitions

You may have noticed that, the size of the output buffer is critical - too small -> lots of writes; too large -> wasteful; hence, the choice of a suitable size for the output buffer is device-specific. The implementation of buffers is in **Chapter 10**.

In this chapter, since we are discussing **scull**, data is already available when ```read``` is issued, no input buffer is used because data is simply copied to the memory area associated with the device - the device is a buffer.

Explicitly nonblocking I/O is indicated by the ```O_NONBLOCK``` flag (open-nonblock) in ```filp->f_flags```, defined in *<linux/fcntl.h>*, included by *<linux/fs.h>*. The reason it is called "open-nonblock" is that it can be specified at open time.

**TODO:** More details on p. 152 


### Example of Blocking I/O
In *scull/pipe.c*, a pipe-like device.

Usually, a process blocked in a ```read``` call is awakened when data arrives by the hardware issuing an interrupt to signal such an event; then the driver handle the interrupt by awakening the waiting processes. In *scullpipe* driver, it works differently - can be run without requiring any particular hardware or an interrupt handler; instead, using another process to generate the data and wake the reading process.

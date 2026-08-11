---
name: clang-kernel-modules
description: "Writing Linux kernel modules (LKMs) with Clang — Kbuild, module parameters, /proc, sysfs, character devices, KGDB debugging. Use when writing loadable kernel modules or debugging kernel code."
user-invocable: true
license: MIT
compatibility: Designed for Claude Code or similar AI coding agents. Requires clang, kernel headers, and git.
metadata:
  author: lobor
  version: "1.0.0"
  openclaw:
    emoji: "🔧"
    homepage: https://github.com/lobor/cc-skills-clang
    requires:
      bins:
        - clang
        - make
    install: []
allowed-tools: Read Edit Write Glob Grep Bash(clang:*) Bash(make:*) Bash(git:*) Agent
---

# Clang Kernel Modules

Writing and debugging loadable Linux kernel modules.

## Minimal Module

```c
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/init.h>

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Name");
MODULE_DESCRIPTION("Minimal module");

static int __init hello_init(void) {
    printk(KERN_INFO "hello: loaded\n");
    return 0;
}

static void __exit hello_exit(void) {
    printk(KERN_INFO "hello: unloaded\n");
}

module_init(hello_init);
module_exit(hello_exit);
```

## Kbuild Makefile

```makefile
obj-m := hello.o

KDIR := /lib/modules/$(shell uname -r)/build

all:
	$(MAKE) -C $(KDIR) M=$(PWD) modules

clean:
	$(MAKE) -C $(KDIR) M=$(PWD) clean
```

```bash
make                          # build
sudo insmod hello.ko          # load
lsmod | grep hello            # verify
dmesg | tail -5               # see printk
sudo rmmod hello              # unload
```

## Module Parameters

```c
#include <linux/moduleparam.h>

static int count = 1;
static char *name = "world";

module_param(count, int, S_IRUGO | S_IWUSR);
module_param(name, charp, S_IRUGO);
```

```bash
sudo insmod hello.ko count=3 name="kernel"
echo 5 > /sys/module/hello/parameters/count  # runtime (if S_IWUSR)
```

## /proc Interface

```c
#include <linux/proc_fs.h>
#include <linux/seq_file.h>

static int mymod_show(struct seq_file *m, void *v) {
    seq_printf(m, "counter: %d\n", my_counter);
    return 0;
}

static int mymod_open(struct inode *inode, struct file *file) {
    return single_open(file, mymod_show, NULL);
}

static const struct proc_ops mymod_fops = {
    .proc_open    = mymod_open,
    .proc_read    = seq_read,
    .proc_lseek   = seq_lseek,
    .proc_release = single_release,
};
```

## sysfs Interface

```c
#include <linux/kobject.h>
#include <linux/sysfs.h>

static ssize_t value_show(struct kobject *kobj,
                          struct kobj_attribute *attr, char *buf) {
    return sprintf(buf, "%d\n", mymod_value);
}

static ssize_t value_store(struct kobject *kobj,
                           struct kobj_attribute *attr,
                           const char *buf, size_t count) {
    sscanf(buf, "%d", &mymod_value);
    return count;
}

static struct kobj_attribute value_attr =
    __ATTR(value, 0664, value_show, value_store);
```

## Character Device

```c
#include <linux/cdev.h>
#include <linux/fs.h>
#include <linux/uaccess.h>

static int mydev_open(struct inode *inode, struct file *file) { return 0; }
static int mydev_release(struct inode *inode, struct file *file) { return 0; }

static ssize_t mydev_read(struct file *f, char __user *buf,
                          size_t len, loff_t *off) {
    if (copy_to_user(buf, kernel_buf, to_copy)) return -EFAULT;
    return to_copy;
}

static ssize_t mydev_write(struct file *f, const char __user *buf,
                           size_t len, loff_t *off) {
    if (copy_from_user(kernel_buf, buf, to_copy)) return -EFAULT;
    return to_copy;
}

static const struct file_operations mydev_fops = {
    .owner   = THIS_MODULE,
    .open    = mydev_open,
    .release = mydev_release,
    .read    = mydev_read,
    .write   = mydev_write,
};
```

## Debugging

```bash
# KGDB
gdb vmlinux
(gdb) target remote /dev/ttyS0

# ftrace
echo function > /sys/kernel/debug/tracing/current_tracer
echo mymod_write > /sys/kernel/debug/tracing/set_ftrace_filter
echo 1 > /sys/kernel/debug/tracing/tracing_on

# Dynamic debug
echo "module hello +p" > /sys/kernel/debug/dynamic_debug/control
```

## EXPORT_SYMBOL

```c
EXPORT_SYMBOL(my_helper);       // any module
EXPORT_SYMBOL_GPL(gpl_only_fn); // GPL-only modules
```

## Mental Model

1. `module_init` / `module_exit` are your entry/exit points
2. Cleanup in reverse order of initialization (goto cleanup pattern)
3. `printk` is your `printf` — check with `dmesg`
4. Never block in init, never leak in exit

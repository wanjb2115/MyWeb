---
title: "Linux Kernel开发常用节点片段"
date: 2024-02-11T21:42:38+08:00
draft: true
tags: ["Linux内核开发"]
categories: ["内核之旅"]
Summary: "Linux Kernel开发常用节点片段"
---

## 内核头文件定义

```c

#ifndef __EXAMPLE_H
#define __EXAMPLE_H

#endif /* __EXAMPLE_H */
```

## Log打印
```c
#include <linux/printk.h>

#undef pr_fmt
#define pr_fmt(fmt) "example" fmt
#define example_info(fmt, args...) pr_info(fmt, ##args)
#define example_warn(fmt, args...) pr_warn(fmt, ##args)
#define example_err(fmt, args...) pr_err(fmt, ##args)
#define example_debug(fmt, args...) pr_debug(fmt, ##args)
```

## 创建一个procfs节点

```c
#include <linux/proc_fs.h>
#include <linux/seq_file.h>
#include <linux/types.h>
#include <linux/module.h>

#define EXAMPLE_DIR_NAME "example"
#define EXAMPLE_NODE_NAME "example"
#define BUFFER_LEN 128


static struct proc_dir_entry *root_entry = NULL;

static const char id[] = "example";

static int example_proc_read(struct seq_file *m, void *v) {

    seq_printf(m, "%s\n", id);

    return 0;
}

static int example_proc_open(struct inode *inode, struct file *file) {
    return single_open(file, example_proc_read, NULL);
}

static ssize_t example_proc_write(struct file *fp, const char __user *user_buff, size_t count, loff_t *pos) {

    char tmp_buf[BUFFER_LEN];

    if(copy_from_user(tmp_buf, user_buff, count)) {
        pr_err("Error occured when conpy_from_user.");

        return -EINVAL;
    }

    if(strncmp(tmp_buf, id, count-1) == 0) {
        pr_info("Cmp success! Input str is same as the id.");
    } else {
        pr_info("Cmp failed! Input str is not same as the id.");
    }

    return count;
}

static const struct proc_ops example_proc_fops = {
    .proc_open = example_proc_open,
    .proc_read = seq_read,
    .proc_lseek = seq_lseek,
    .proc_write = example_proc_write,
    .proc_release = single_release,
};

int example_procfs_init(void)
{
    umode_t mode = 0666;
    struct proc_dir_entry *tmp_entry;

    root_entry = proc_mkdir(EXAMPLE_DIR_NAME, NULL);

    tmp_entry = proc_create_data(EXAMPLE_NODE_NAME, mode, root_entry, &example_proc_fops, NULL);
    if (tmp_entry == NULL) {
        pr_err("Create file error");

        goto fail;
    }

    return 0;

fail:
    return -ENOMEM;
}

void example_procfs_exit(void)
{
    pr_info("example_procfs_exit exit.\n");

    remove_proc_entry(EXAMPLE_NODE_NAME, root_entry);
    remove_proc_entry(EXAMPLE_DIR_NAME, NULL);
}

module_init(example_procfs_init);
module_exit(example_procfs_exit);
```

## 创建一个sysfs节点
```c
// SPDX-License-Identifier: GPL-2.0

#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/jiffies.h>
#include <linux/kobject.h>
#include <linux/spinlock.h>

#define EXAMPLE_DIR_NAME "example"

static const char id[] = "example";

static ssize_t id_show(struct kobject *kobj, struct kobj_attribute *attr,
            char *buffer);
static ssize_t id_store(struct kobject *kobj, struct kobj_attribute *attr,
            const char *buffer, size_t length);

static ssize_t jiffies_show(struct kobject *kobj, struct kobj_attribute *attr,
            char *buffer);

static ssize_t example_show(struct kobject *kobj, struct kobj_attribute *attr,
            char *buffer);
static ssize_t example_store(struct kobject *kobj, struct kobj_attribute *attr,
            const char *buffer, size_t length);

DEFINE_RWLOCK(example_lock);

static char example_msg[PAGE_SIZE];

static struct kobj_attribute id_attribute = __ATTR_RW_MODE(id, 0664);

static struct kobj_attribute jiffies_attribute = __ATTR_RO(jiffies);

static struct kobj_attribute example_attribute = __ATTR_RW(example);

static struct attribute *attrs[] = {
    &id_attribute.attr,
    &jiffies_attribute.attr,
    &example_attribute.attr,
    NULL,
};

static struct attribute_group example_group = {
    .attrs = attrs,
};

static struct kobject *example_kobj;

static ssize_t id_show(struct kobject *kobj, struct kobj_attribute *attr,
            char *buffer)
{
    return sysfs_emit(buffer, "%s\n", id);
}

static ssize_t id_store(struct kobject *kobj, struct kobj_attribute *attr,
            const char *buffer, size_t length)
{

    if (!strncmp(id, buffer, strlen(id)))
        return length;

    return -EINVAL;
}

static ssize_t jiffies_show(struct kobject *kobj, struct kobj_attribute *attr,
            char *buffer)
{
    return sysfs_emit(buffer, "%lu\n", jiffies);
}

static ssize_t example_show(struct kobject *kobj, struct kobj_attribute *attr,
            char *buffer)
{

    read_lock(&example_lock);
    strncpy(buffer, example_msg, strlen(example_msg));
    read_unlock(&example_lock);

    return strlen(example_msg);
}

static ssize_t example_store(struct kobject *kobj, struct kobj_attribute *attr,
            const char *buffer, size_t length)
{

    if (length >= PAGE_SIZE)
        return -EINVAL;

    write_lock(&example_lock);
    strncpy(example_msg, buffer, length);
    write_unlock(&example_lock);

    return length;
}

int example_sysfs_init(void)
{
    ssize_t ret;

    pr_info("Hello World, example_sysfs_init!\n");

    example_kobj = kobject_create_and_add(EXAMPLE_DIR_NAME, kernel_kobj);
    if (!example_kobj)
        return -ENOMEM;

    ret = sysfs_create_group(example_kobj, &example_group);
    if (ret)
        kobject_put(example_kobj);

    return 0;
}

void example_sysfs_exit(void)
{
    pr_info("example_sysfs_exit exit.\n");

    kobject_put(example_kobj);
}

module_init(example_sysfs_init);
module_exit(example_sysfs_exit);
```

## 创建一个drivers/misc节点

```c
// SPDX-License-Identifier: GPL-2.0
#include <linux/fs.h>
#include <linux/init.h>
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/miscdevice.h>

#define EXAMPLE_DIR_NAME "example"
#define EXAMPLE_NODE_NAME "example"
#define EXAMPLE_BUFFER_LEN 128


static const char example_id[] = "example";

static ssize_t example_read(struct file *filp, char *buffer,
        size_t length, loff_t *offset);

static ssize_t example_write(struct file *filp, const char *buffer,
        size_t length, loff_t *offset);
static const struct file_operations example_fops = {
    .owner = THIS_MODULE,
    .read = example_read,
    .write = example_write
};

static struct miscdevice example_dev = {
    .minor = MISC_DYNAMIC_MINOR,
    .name = EXAMPLE_NODE_NAME,
    .fops = &example_fops
};

static ssize_t example_read(struct file *filp, char *buffer, size_t length,
        loff_t *offset)
{
    return simple_read_from_buffer(buffer, length, offset, example_id, sizeof(example_id));
}

static ssize_t example_write(struct file *filp, const char *buff, size_t length,
        loff_t *offset)
{
    ssize_t ret;
    char msg[EXAMPLE_BUFFER_LEN] = {0};

    ret = simple_write_to_buffer(msg, sizeof(msg), offset, buff, length);
    if (ret < 0)
        return ret;

    if (!strncmp(msg, example_id, strlen(example_id)))
        return length;

    return -EINVAL;
}

int example_misc_init(void)
{
    int ret;

    ret = misc_register(&example_dev);
    if (ret)
        pr_debug("Miscdevice example_dev register failed");

    return ret;
}

void example_misc_exit(void)
{
    misc_deregister(&example_dev);
}


module_init(example_misc_init);
module_exit(example_misc_exit);
```

## 创建一个debugfs节点

```c
// SPDX-License-Identifier: GPL-2.0
#include <linux/fs.h>
#include <linux/init.h>
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/debugfs.h>

#define EXAMPLE_DIR_NAME "example"
#define EXAMPLE_NODE_NAME "example"
#define EXAMPLE_BUFFER_LEN 128

static struct dentry *example_debugfs_root;

static const char example_id[] = "example";

static ssize_t example_read(struct file *filp, char __user *buffer,
    size_t length, loff_t *offset);
static ssize_t example_write(struct file *filp, const char __user *buffer,
    size_t length, loff_t *offset);

static const struct file_operations example_fops = {
    .owner = THIS_MODULE,
    .read = example_read,
    .write = example_write
};

static ssize_t example_read(struct file *filp, char __user *buffer,
        size_t length, loff_t *offset)
{
    return simple_read_from_buffer(buffer, length, offset,
        example_id, sizeof(example_id));
}

static ssize_t example_write(struct file *filp, const char __user *buffer,
        size_t length, loff_t *offset)
{
    ssize_t ret;
    char msg[EXAMPLE_BUFFER_LEN] = {0};

    ret = simple_write_to_buffer(msg, sizeof(msg), offset,
        buffer, length);
    if (ret < 0)
        return ret;

    if (!strncmp(msg, example_id, strlen(example_id)))
        return length;

    return -EINVAL;
}

int example_debugfs_init(void)
{
    pr_info("Hello World, debugfs_init!");

    example_debugfs_root = debugfs_create_dir(EXAMPLE_DIR_NAME, NULL);

    debugfs_create_file(EXAMPLE_NODE_NAME, 0666, example_debugfs_root, NULL,
        &example_fops);

    return 0;
}

void example_debugfs_exit(void)
{
    pr_info("example debugfs exit.");

    debugfs_remove_recursive(example_debugfs_root);
}


module_init(example_debugfs_init);
module_exit(example_debugfs_exit);
```
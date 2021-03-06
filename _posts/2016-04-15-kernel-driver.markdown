---
layout: post
title:  "Writing a Kernel Driver"
date:   2016-04-15 10:00:00 -0400
categories: Linux,Kernel,Driver
---

# Intro

I have worked on a few kernel drivers before and every time I spend a couple of days setting up the build environment and figuring out how to do everything again. This is my attempt at helping myself out in the future. So to future Dave, you owe me a beer.

There are a lot of different types of modules to build, this one is aimed at presenting a file interface for a hardware device. Basically it allows userland access to your device using both file operations and sysfs. There are a lot of other ways to build modules. Perhaps in the future the git repo can be branched to accomodate these variations.

A high level view of the system:

![Driver and Userland]({{ site.baseurl }}/assets/image/posts/kernel-module/kernelmodule_driver.png)

This guide will walk through:

  * **Setup** The initial setup when building a module include preparing the host and downloading the code.
  * **Design** The design of the module
  * **Code-Build-Debug-Repeat** Debugging

To make things easy for this guide I'll declare the following things:

  * Module Name: **wallaby**
  * Module Location: **~/Projects/wallaby**

# Setup

The one time setup consists of the following steps:

  * Setting up the system
  * Cloning the github repo
  * Rename the module

## System Setup

Setup Ubuntu to build the kernel module. We'll need both build-essentials and the kernel headers.

{% highlight bash %}
sudo apt-get install linux-headers-$(uname -r)) build-essentials
{% endhighlight %}

## Getting the skeleton

I created a skeleton that can help get started with the build process

[Kernel Module Skeleton](https://github.com/CospanDesign/kernel-module)

Clone this repository to somewhere on your system.

{% highlight bash %}
git clone https://github.com/CospanDesign/kernel-module.git wallaby
{% endhighlight %}

## Modifying the Skeleton with your module name

There is a script that needs to be run one time that will rename the modules and all references to that module. It should probably be deleted after it's use or you may inadvertently rename things later on.

# Design

## ioctrl vs sysfs
As stated in the intro this is one type of driver focused on presenting a file like interface to a hardware device. File interfaces are useful becaues most programming languages can interact with files. There is one issue. Sometimes we need to interact with our hardware in a way that doesn't make sense using 'read' or 'write'. For example in order to configure a UART to operate at a specific baudrate users would need to use 'out of band' signals called 'Input Output Control' or ioctrl.

If you have ever used ioctl you can appreciate that it was frustrating to say the least. The main reason for this is because the user needed to know what specific ioctl corresponded to what function as well as what values were valid. The problem was exhaserbated by the fact that an ioctl for one driver could be completely different foa another driver. As can be imagined there were great attempts to figure this issue out with 'known ioctl numbers' as well as other such mechanisms.

A newer approach to controlling devices, drivers and general system functionality was introduced with sysfs. There are a lot of resources on the web describing the history and benefits but suffice to say we will be using sysfs to configure our device.

## Anatomy of a driver

A driver can be broken down into four regions.

  * Module Initialization/Remove.
  * Per Device Identification, Initialization and Removal.
  * User Interface.
  * General Driver Infrastructure.

### Kernel Module Installation and Removal

When a driver is loaded and removed the kernel uses the 'module_init' and 'module_exit' macros to identify which functions should be called. driver developers can assign a specific function as follows:

{% highlight c %}
static int wallaby_init(void)
{
}

static int wallaby_exit(void)
{
}

module_init ("wallaby_init");
modue_exit ("wallaby_exit");

{% endhighlight %}

The responsibility of these two functions are as follows:

  * **init**
    * Perform one time configuration for the driver.
    * Describe how the kernel can identify the device.

  * **exit**
    * Remove and cleanup any resources the driver declared.


### Per Device Probe/Disconnect

Identifying the hardware device is protocol dependent, some drivers can have devices installed and disconnect while the system is online. (Example: Adding or removing a USB Device)

For USB drivers, because devices can be inserted and disconnect while the system is online then drivers need a way to tell the kernel how to identify the devices that they work. In terms of drivers they 'register' themselves with the kernel.

_I like to think of the kernel managing a construction yard. A driver registers or tells the kernel that they can drive specific vehicles and can identify these vehicles using provided ID. If a vehicle matching any of the provided IDs the kernel will then notify the driver that there is a valid vehicle by calling the drivers 'probe' funcition. Similarly if the vehicle is disconnect the kernel will notify the driver by calling the 'disconnect' function_

The 'probe' and 'disconnect' functions are not declared using a macro instead the driver usually declares these two functions in a protocol specific way. For example USB uses a structure called **usb\_driver** that is initialized when the module's 'init' function is called.

The static structure is populated with usb specific functions a great reference to use is the 'usb-skeleton.c' driver found in kernel/driver/usb folder.


#### USB Device

In this module the usb\_driver structure looks like this:

{% highlight c %}

#define MODNAME "skeleton"

static int skel_probe(struct usb_interface *interface, const struct usb_device_id *id)
{
  return 0;
}

static void skel_disconnect(struct usb_interface *interface)
{
}

static struct usb_driver skel_driver = {
  .name =   MODNAME,
  .probe =  skel_probe,
  .disconnect = skel_disconnect,
  .suspend =  skel_suspend,
  .resume = skel_resume,
  .pre_reset =  skel_pre_reset,
  .post_reset = skel_post_reset,
  .id_table = skel_table,
  .supports_autosuspend = 1,
};

module_usb_driver(skel_driver);

{% endhighlight %}


#### PCIE Device


{% highlight c %}

#define MODNAME "skeleton"

static int skel_probe(struct pci_dev *pdev, const struct pci_device_id *id)
{
  return 0;
}

static void pcidriver_remove(struct pci_dev *pdev)
{
}

static struct pci_driver skel_driver = {
  .name = MODNAME,
  .id_table = skel_ids,
  .probe = skel_probe,
  .remove = skel_remove,
};

{% endhighlight %}


XXX: Remove this comment, it's here to fix VIM's over zealous highlighting
*/

# Interfacing with your driver

From the kernel's point of view the driver presents a _recognized interface_ to a device. From the device's point of view the driver translates the generic requests such as 'read' and 'write' to a device specific _native language_.

The term _recognized interface_ can mean many different things. One interpretation can mean a file like interface. Where the driver enables the user to interact the device in the same way they would interact with a file. Files allow the user to 'read' and 'write', some allow you to 'seek'. Most programming languages understand how to interact with a file so presenting your driver as a file allows users to interface with the device in an easy to use way.

Drivers do not need to interface with the user directly, instead the kernel may attach the driver to another module. As an example a USB camera may not interface with the user directly through a file but instead interface with a 'video4linux' module that transates data from the driver to the video subsystem.


## File Interface

We will use a file interface to communicate with the driver. As stated above most programming languages understand how to interact with a file. One issue with file interface is out of band signals or signals that are not specifically for reading and writing to a device may not be easy to do. We'll use sysfs in a later section. For now we'll talk about how to implement the file operations.

File oeprations is accomplished by:

  * Declaring a file\_operation structure that points to appropriate signals.
  * Instantiated an 'inode' or a file like entry into the /dev directory.


### Declaring a 'file\_operation' Structure

Inside the driver we'll include the file operation structure and populate it with functions that satisfy the required functionality.


{% highlight c++ %}
//-----------------------------------------------------------------------------
// File Operations
//-----------------------------------------------------------------------------

int wallaby_open(struct inode *inode, struct file *filp)
{
  return SUCCESS;
}

int wallaby_release(struct indoe *inode, struct file *filp)
{
  return SUCCESS;
}

ssize_t wallaby_write(struct file *filp, const char *buf, size_t count, loff_t = SUCCESS)
{
  return 0;
}

ssize_t wallaby_read(struct file *filp, char * buf, size_t count, loff_t *f_pos)
{
  return 0;
}


struct file_operations nysa_pcie_fops = {
  owner:    THIS_MODULE,
  read:     wallaby_read,
  write:    wallaby_write,
  open:     wallaby_open,
  release:  wallaby_release
}
{% endhighlight %}


There are more file operations to use but these are the minimum (I could tell). If you would like to implement more of the file\operations you can look them up here:

[File Operations Header](http://lxr.free-electrons.com/source/include/linux/fs.h)

### Declaring sysfs interface

As discussed above sysfs is the modern place where out of band signals can be implemented. Because sysfs exposes named interfaces users can more easily determine what the different signals mean.

There is a much better explanation of sysfs here:

[sysfs](https://www.kernel.org/pub/linux/kernel/people/mochel/doc/papers/ols-2005/mochel.pdf)

For brevity I'll only give a quick example of how to read and write values to/from the driver:

First create an attribute structure:


{% highlight c++ %}

const char wallaby_name[] = "attr1";

ssize_t show_wallaby_attr1 (struct device *dev, const char * buf){
  printk("User show: %s\n", wallaby_name);
  buf = wallaby_name;
  return 0;
}

ssize_t store_wallaby_attr1 (struct device * dev, char * buf, size_t count){
  struct attribute* wallaby_attr = dev->user->attr1;
  printk("User set: %s\n", buf);
  return 0;
}


/* Start Create */

int retval = 0;
//Create
struct attribute* wallaby_attr = (*struct attribute) kmalloc(sizeof(struct attribute));
struct device_attribute *wallaby_dev_attr = (*struct device_attribue) kmalloc(sizeof(struct device_attribute));

wallaby_attr->name = &wallaby_name;
wallaby_attr->owner = THIS_MODULE;
wallaby_attr->mode = 666; //Read and write

wallaby_dev_attr->attr = wallaby_attr;

retval = sysfs_create_file(/*Object */, &wallaby_dev_attr);
if (retval) {
  printk ("Failed to create sysfs file for attrubute: %s\n", wallaby_name);
}

/* End Create */


/* Start Tear Down */

//Tear down
sysfs_remove_file(/*Object*/, &wallaby_dev_attr);

/* End Tear Down */

{% endhighlight %}







# Code-Build-Debug-Repeat

Now that all the configuration is done it's time build and test your code


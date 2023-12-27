---
title: 雪夜杂谈——认识LKM
tags: [kernel,pwn]
index_img: /img/lkm.jpg
banner_img: /img/lkm.jpg
date: 2022-10-27 15:32:24
---
## 0、引子
先说点小遗憾吧：笔者水平所限，关于linux是如何实现设备文件化以及如何实现LKM的没能深入探究，本节着重于「如何使用」的实用角度来了解LKM。后续如果笔者水平有所提高，会从更加细致的角度来进行探究。  
<!--more-->
## 1、LKM是什么
LKM是Linux内核为了扩展其功能所使用的可加载内核模块。LKM的优点：动态加载，无须重新实现整个内核。基于此特性，LKM常被用作特殊设备的驱动程序（或文件系统），如声卡的驱动程序等等。  
LKM内核模块属于ELF目标文件，但又区别于一般的应用程序，属于系统级别的程序，用来扩展Linux内核功能。通常使用LKM加载一些设备驱动，可以捕获系统调用，功能十分强大。  
**注：LKM常用于做驱动程序，但是并非每个LKM都是驱动。** 驱动代表操作设备的方式和流程，而LKM仅仅是内核动态加载的模块。  
在Pwn过程中，出问题的地方大多都是LKM的代码，因为其运行在内核态，与内核，所以针对LKM的攻击可以在内核中实现很多结果，提权就是其中的一个目标。  
## 2、LKM撰写及动态加载
我们从撰写一个典型的linux开始：
```c
#include <linux/init.h>
#include <linux/module.h>
#include <linux/kernel.h>
MODULE_LICENSE("Dual BSD/GPL");
static int ko_test_init(void) 
{
    printk("This is a test ko!\n");
    return 0;
}
static void ko_test_exit(void) 
{
    printk("Bye Bye~\n");
}
module_init(ko_test_init);
module_exit(ko_test_exit);
```
其中module_init()是初始化函数，其在安装模块时被调用，所有的初始化工作可以在其中完成。module_exit()是清除函数，在卸载模块时调用。并且，我们用来输出字符产的函数是printk而不是printf。  
在LKM中，是无法依赖于我们平时使用的C库的，**模块仅仅被链接到内核**，只可以调用内核所导出的函数，不存在可链接的函数库；这也是内核编程与我们平时应用程序编程的不同之一。
```Makefile
KDIR =/usr/src/linux-headers-5.15.0-50-generic
obj-m += ko_test.o
all:
        @$(MAKE) -C $(KDIR) M=$(PWD) modules

clean:
        rm -rf *.o *.ko *.mod.* *.symvers *.order
```
这里的Makefile编写很有意思。  
-C 选项的作用是指将当前工作目录转移到你所指定的位置。“M=”选项的作用是，当用户需要以某个内核为基础编译一个外部模块的话，需要在make modules 命令中加入“M=dir”，程序会自动到你所指定的dir目录中查找模块源码，将其编译，生成KO文件。所以我们是在内核文件夹下执行了make M=$(PWD) modules。  
编译完成后我们用root权限执行insmod加载驱动，利用rmmod移除驱动。  
```
emiya@emiya-virtual-machine:~/桌面/KernelPwn/ko$ make clean
rm -rf *.o *.ko *.mod.* *.symvers *.order
emiya@emiya-virtual-machine:~/桌面/KernelPwn/ko$ make
make -C /usr/src/linux-headers-5.15.0-50-generic M=/home/emiya/桌面/KernelPwn/ko modules
make[1]: 进入目录“/usr/src/linux-headers-5.15.0-50-generic”
  CC [M]  /home/emiya/桌面/KernelPwn/ko/ko_test.o
  MODPOST /home/emiya/桌面/KernelPwn/ko/Module.symvers
  CC [M]  /home/emiya/桌面/KernelPwn/ko/ko_test.mod.o
  LD [M]  /home/emiya/桌面/KernelPwn/ko/ko_test.ko
  BTF [M] /home/emiya/桌面/KernelPwn/ko/ko_test.ko
Skipping BTF generation for /home/emiya/桌面/KernelPwn/ko/ko_test.ko due to unavailability of vmlinux
make[1]: 离开目录“/usr/src/linux-headers-5.15.0-50-generic”
emiya@emiya-virtual-machine:~/桌面/KernelPwn/ko$ sudo insmod ko_test.ko
emiya@emiya-virtual-machine:~/桌面/KernelPwn/ko$ sudo rmmod ko_test.ko
emiya@emiya-virtual-machine:~/桌面/KernelPwn/ko$ 
```
在另外的控制台利用`desmg -w`指令进行实时监视：  
```
[ 5818.816424] This is a test ko!
[ 5826.482328] Bye Bye~
```
## 3、为LKM传递参数
**模块参数**：简单来说模块参数允许用户再加载模块时通过命令行指定参数值，在模块的加载过程中，加载程序会得到命令行参数，并转换成相应类型的值，然后复制给对应的变量，这个过程发生在调用模块初始化函数之前。  
在上一节之中的`ko_test.c`中加入如下代码，并且在`ko_test_init`中将其输出。
```c
static int baudrate = 9600;
static int port[4] = {0,1,2,3};
static char *name = "vser";

module_param(baudrate,int,S_IRUGO);
module_param_array(port,int,NULL,S_IRUGO);
module_param(name,charp,S_IRUGO);
```
我们看到`module_param`和`module_param_array`被用来声明可以命令行传递的参数，参数从头到尾分别为变量名，变量（或者数组元素）类型，（数组长度），文件类型。  
我们可以在加载时直接用key=value的方式进行传参：
`sudo insmod ko_test.ko name="emiya" port=1,6,6,6 baudrate=6666`  
结果如下，对比发现如果没有在加载时传参则显示默认的变量，如果传参则显示传递的参数。
```
[ 9668.700714] baudrate:9600
[ 9668.700715] 0
[ 9668.700716] 1
[ 9668.700716] 2
[ 9668.700716] 3

[ 9668.700717] name: vser
[ 9877.239272] Bye Bye~
[ 9928.136959] init module:hello world!
                
[ 9928.136961] baudrate:6666
[ 9928.136962] 1
[ 9928.136963] 6
[ 9928.136963] 6
[ 9928.136963] 6

[ 9928.136964] name: emiya
```
## 4、接轨驱动
### 4.1 与内核交互
Linux就提供了一系列的系统调用，用来让开发者在用户态与系统沟通，进而与硬件沟通，其中就包括了我们常用的open、close、read、write和ioctl。  
以`ioctl`为例
```c
int ioctl(int fd, int cmd, ...)
```
而陷入内核之后，ioctl的函数原型为
```c
int (*ioctl)(struct inode *node, struct file *filp, unsigned int cmd, unsigned long arg)
```
ioctl变成了函数指针，fd被转换为两个结构体来标识节点和文件，用来标识操作的设备文件，cmd被原封不动的传入了驱动中。  
我们来扩充一下我们的内核模块，在这时，我们可以叫它「驱动」了，因为它确实可以给我们提供访问设备文件的接口了。  
首先是一些文件包含，以及我们要注册的设备类名，设备名，主设备号的声明。
```c
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/init.h>
#include <linux/device.h>
#include <linux/fs.h> 
MODULE_LICENSE("Dual BSD/GPL");

#define DEVICE_NAME "ko_test"
#define CLASS_NAME "ko_test"
static int majorNumber;
static struct   class*  ko_test_class = NULL;
static struct   device* ko_test_device = NULL;
```
接下来，是我们自己定义的用于ioctl交互的函数，当用户态用ioctl操作我们注册的驱动文件时，将会调用此函数进行响应。
```c
static long ko_test_ioctl(struct file *file,unsigned int cmd,unsigned long param)
{
    switch(cmd)
    {
        case 0:
        {
            printk("check ioctl 0\n");
            break;
        }
        default :
            printk("unknown ioctl\n");

    }
    return 0;
}
```
file_operation就是把系统调用和驱动程序关联起来的关键数据结构。这个结构的每一个成员都对应着一个系统调用。读取file_operation中相应的函数指针，接着把控制权转交给函数，从而完成了Linux设备驱动程序的工作。

在系统内部，I/O设备的存取操作通过特定的入口点来进行，而这组特定的入口点恰恰是由设备驱动程序提供的。通常这组设备驱动程序接口是由结构file_operations结构体向系统说明的，它定义在include/linux/fs.h中。
```c
static const struct file_operations ko_test_options = {
        .owner = THIS_MODULE,
        .unlocked_ioctl = ko_test_ioctl,
};
```
从以下的初始化设备函数中，我们看到初始化设备的过程主要是以下三个函数：
* register_chrdev
* class_create
* device_create  

`register_chrdev`向内核注册了一个字符设备。
第一个参数是主设备号，0代表动态分配,第二个参数是设备的名字，第三个参数是文件操作指针。  
`class_create`内核中定义了struct class结构体，顾名思义，一个struct class结构体类型变量对应一个类，内核同时提供了class_create函数，可以用它来创建一个类，这个类存放于sysfs下面，一旦创建好了这个类，再调用`device_create`函数来在/dev目录下创建相应的设备节点。这样，加载模块的时候，用户空间中的udev会自动响应device_create函数，去/sysfs下寻找对应的类从而创建设备节点。  

```c
static int ko_test_init(void) 
{
    printk(KERN_INFO "Entering test module. \n");
    majorNumber = register_chrdev(0, DEVICE_NAME, &ko_test_options);
    if(majorNumber < 0){
        printk(KERN_INFO "Failed to register a major number. \n");
        return majorNumber;
    }
    printk(KERN_INFO "Successful to register a major number %d. \n", majorNumber);
    ko_test_class = class_create(THIS_MODULE, CLASS_NAME);
    if(IS_ERR(ko_test_class))
    {
         unregister_chrdev(majorNumber, DEVICE_NAME);
         printk(KERN_INFO "Class device register failed!\n");
         return PTR_ERR(ko_test_class);
    }
    printk(KERN_INFO "Class device register success!\n");
    ko_test_device = device_create(ko_test_class, NULL, MKDEV(majorNumber, 0), NULL, "ko_test");
    if (IS_ERR(ko_test_device))
    {
        class_destroy(ko_test_class);
        unregister_chrdev(majorNumber, DEVICE_NAME);
        printk(KERN_ALERT "Failed to create the device\n");
        return PTR_ERR(ko_test_device);
    }
    printk(KERN_INFO "Test module register successful. \n");
    return 0;
}
static void ko_test_exit(void) 
{
    printk(KERN_INFO "Start to clean up module.\n");
    device_destroy(ko_test_class, MKDEV(majorNumber, 0));
    class_destroy(ko_test_class);
    unregister_chrdev(majorNumber, DEVICE_NAME);
    printk(KERN_INFO "Clean up successful. Bye.\n");

}
module_init(ko_test_init);
module_exit(ko_test_exit);
```
编写简单的test.c程序打开设备文件并与其进行交互。
```c
#include <stdlib.h>
#include <errno.h>
#include <fcntl.h>
#include <unistd.h>
#include <sys/ioctl.h>
int main(){
        int fd;
        fd = open("/dev/ko_test", O_RDWR); //我们的设备挂载在/dev/test处
        if (fd < 0){
                perror("Failed to open the device...");
                return errno;
        }else{
                printf("Open device successful!\n");
        }
        ioctl(fd, 0);
        printf("Called ioctl with parameter 0!\n");
}
```
结果如下：
```
emiya@emiya-virtual-machine:~/桌面/KernelPwn/ko$ make
make[1]: 进入目录“/usr/src/linux-headers-5.15.0-50-generic”
  CC [M]  /home/emiya/桌面/KernelPwn/ko/ko_test.o
  MODPOST /home/emiya/桌面/KernelPwn/ko/Module.symvers
  CC [M]  /home/emiya/桌面/KernelPwn/ko/ko_test.mod.o
  LD [M]  /home/emiya/桌面/KernelPwn/ko/ko_test.ko
  BTF [M] /home/emiya/桌面/KernelPwn/ko/ko_test.ko
Skipping BTF generation for /home/emiya/桌面/KernelPwn/ko/ko_test.ko due to unavailability of vmlinux
make[1]: 离开目录“/usr/src/linux-headers-5.15.0-50-generic”
emiya@emiya-virtual-machine:~/桌面/KernelPwn/ko$ sudo insmod ko_test.ko
emiya@emiya-virtual-machine:~/桌面/KernelPwn/ko$ sudo ./test
Open device successful!
Called ioctl with parameter 0!
emiya@emiya-virtual-machine:~/桌面/KernelPwn/ko$ sudo ./test
Open device successful!
Called ioctl with parameter 0!
emiya@emiya-virtual-machine:~/桌面/KernelPwn/ko$ sudo ./test
Open device successful!
Called ioctl with parameter 0!
emiya@emiya-virtual-machine:~/桌面/KernelPwn/ko$ sudo ./test
Open device successful!
Called ioctl with parameter 0!
```
同时desmg的消息如下，说明设备交互成功
```
[ 4303.207265] Entering test module. 
[ 4303.207270] Successful to register a major number 236. 
[ 4303.207323] Class device register success!
[ 4303.207929] Test module register successful. 
[ 4317.394714] check ioctl 0
[ 4319.625982] check ioctl 0
[ 4321.673908] check ioctl 0
[ 4322.504900] check ioctl 0
```
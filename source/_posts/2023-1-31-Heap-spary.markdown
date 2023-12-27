---
title: 雪夜杂谈——关于内核堆喷
categories: pwn
tags: [kernel,pwn]
index_img: /img/ori_slab.png
banner_img: /img/ori_slab.png
date: 2023-01-30 23:25:24
---

## 一、引言
  在用户态的单线程程序中，以ptmalloc为例，malloc函数分配的空间仅在该进程的用户空间中可见，堆块的申请和释放仅仅和本进程内部的malloc与free有关，只要进程对于堆块的操作和堆分配器的策略已知，那么在某一时刻堆分配的结构就是已知的。这样的性质对于用户态堆溢出和uaf的利用来说十分有帮助。  
<!--more-->
当然，不排除在很多复杂的数据结构下，堆布局也存在着不同的困难，但是总体来说，用户态的对利用存在着不言自明的便利，例如，对于这样的指令序列：  
```c
void* a = malloc(0x20);
void* b = malloc(0x20);
```
在进程的地址空间中，如果a，b分别是该进程申请的第一、第二个堆块，那么**a与b堆块是相邻的**，如果堆块a存在溢出，那么我们就会清楚地直到b堆块是被溢出堆块。而uaf的利用也是如此，例如对于以下序列：  
```c
void* a = malloc(0x20);
free(a);
……
void* b = malloc(0x20);
```
如果a，b分别是该进程申请的第一、第二个堆块，那么**a、b此时指向同一个堆块**，如果a在free之后并没有置空，对a“堆块”的操作也就是在操作b堆块。  
以上两种加粗的情况，就是CTF中堆利用最小、最简单普遍的构造布局，更加大型的布局例如heap overleap，house of 系列大都依赖着堆块分配策略的稳定性和确定性。然而，这点在内核pwn的过程却不能完全保证，存在以下几个原因：  
* exploit运行的过程中并不能覆盖全部申请堆块的进程
* linux内核堆分配机制可以将不同进程、相同大小的堆块分配在一个slub中
* 不同的时刻，申请堆块时堆块来源的内存页可能不同

所以在内核存在堆溢出和uaf时，我们需要提高运行exploit时达成特定堆布局的概率。以下引入内核堆喷射的方法。  
## 二、slab堆喷与堆溢出混合使用
既然需要在内核空间中达成特定的堆布局，我们就需要了解在内核`kmalloc/kfree`时的策略，该部分可以阅读源码或者在引用文献[1]中找到。  
在运行exploit时，我们假设slab中总是半满（partial）的，即：当前cpu对应的slab中可用对象（freed object）和已分配对象（used object）交替分布，如图：  
<center>
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" 
    src="/assets/ori_slab.png">
    <br>
    <div style="color:orange;
    display: inline-block;
    color: #999;
    padding: 15px;">随机的slab布局</div>
</center>  
  

这时，在申请堆块时，`kmalloc`等函数返回堆块的地址杂乱并且相对关系不明，所以，我们需要构建**新建空slab布局**。堆喷射与堆溢出一起使用时，旨在**大量申请目标大小堆块，将部分满（partial）的slab全部充满，迫使分配系统新建空slab，使得堆喷后，在空新建空slab上申请地址相对关系可预测。**  
首先，在一个slab初始化时，slab中的内存分为两部分，一部分是内存对齐的对象，另一部分则是索引数组，所有对象都可用对象（freed object），索引数组递增排列，slab的`freelist`指向第一个索引，表示该slab的下一个可用对象为第一个对象（obj 0）。
<center>
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" 
    src="/assets/empty_slab.png">
    <br>
    <div style="color:orange;
    display: inline-block;
    color: #999;
    padding: 15px;">新建空slab布局</div>
</center>  
  

当对象从slab中拿出时，调用的是`slab_get_obj`，代码如下，可以看出是取
```c
static void *slab_get_obj(struct kmem_cache *cachep, struct page *page){
	objp = index_to_obj(cachep, page, get_free_obj(page, page->active));
	page->active++;
	return objp;
}
```
在此时，如果拿出堆块a的地址为A_a，拿出堆块b的地址为A_b，则一定有：  A_a < A_b
  
然而，“申请”的堆块与“拿出”的堆块不同，slab被拿出的对象只有放进cpu_cache才能够被`kmalloc`等接口返回，该过程`alloc_block`中，可以发现，cpu_cache中entry数组为栈结构，`avail`所指向的第一个可用对象是最后一个入栈元素。  
```c
while (page->active < cachep->num && batchcount--) {
    //put objs into cpu cache
	ac->entry[ac->avail++] = slab_get_obj(cachep, page);
}
```  
所以上述不等关系变为:A_a > A_b
又因为对于所有**新建空slab的连续申请**，所以，只要在两次申请中间没有其它进程打断并申请同样大小的堆块，那么就有：A_a = sizeof(object)+A_b  
此时，如果堆块b存在堆溢出（一般堆喷结构体不存在主动溢出的情况），那么后申请的堆块b就可以溢出到堆喷所用的堆块a。  


### reference：  
>[1]深入理解Linux内存管理（八）slab，slob和slub介绍  https://zhuanlan.zhihu.com/p/490588193  
>[2]2021 InCTF 题目地址 https://github.com/teambi0s/InCTFi/tree/master/  
>[3]InCTF 内核Pwn之 Kqueue https://bbs.kanxue.com/thread-269031.htm  
>[4]linux内核源码阅读 https://elixir.bootlin.com/linux/v5.16/source

---
title: 雪夜杂谈——Ptmalloc堆管理器分析
categories: CTF
tags: [pwn]
index_img: /img/heap.png
banner_img: /img/heap.png
date: 2023-03-08 09:34:00
---

## 前言
glibc的ptmalloc堆管理器的利用一直是CTF Pwn中的重点，但是其复杂性和利用方法的多样性让人难以理清“对于一个漏洞，到底需要怎么样的利用方法来实现getshell？”这种问题。  
本文留作CTF赛前的复习资料，将会把`malloc`，`free`的过程以及检查点，漏洞利用点进行总结，以及近年以来的堆上保护措施的概览。为了更加清晰地展示`malloc`和`free`的变化，基础分析将从glibc-2.23开始。    
<!--more-->
## 一、malloc源码分析
`malloc`函数的重要调用层次结构如下:
```
__libc_malloc
            |----__malloc_hook
            |----_int_malloc
                        |----malloc_consolidate
                        |----sysmalloc
```
这种划分方式能让我们更加清楚地知道需要分析的函数对象，但是`malloc`的分析中为了预测分配器行为，光是知道这些接口还不够，需要对其时序进行更加详细的分析。 
### 1、__libc_malloc   
在顶层（`__libc_malloc`）中，时序很简单，先观察`__malloc_hook`是否有记录，在hook执行完毕后执行内存分配的核心函数`_int_malloc`。  
### 2、_int_malloc  
#### 2.1 fastbin分配  
`fastbin`是`malloc`分配内存的最快途径，分配器检查申请大小处于`fastbin`的请求，找到对应的fastbin链表头，然后遍历该链表找到第一个可用的`fastbin chunck`，所以说，`fastbin`是LIFO的。
```c
if ((unsigned long) (nb) <= (unsigned long) (get_max_fast ()))
    {
      idx = fastbin_index (nb);
      mfastbinptr *fb = &fastbin (av, idx);
      mchunkptr pp = *fb;
      do
        {
          victim = pp;
          if (victim == NULL)
            break;
        }
      while ((pp = catomic_compare_and_exchange_val_acq (fb, victim->fd, victim))//遍历fastbin链表
             != victim);
```
该操作过后，如果能够够找到这样的堆块，对其进行检查然后返回，这里的检查主要是对于得到堆块大小进行检查，是否在其被取出的`fastbin`区间内。  
```c
      if (victim != 0)
        {
          if (__builtin_expect (fastbin_index (chunksize (victim)) != idx, 0))//检查fastbin大小
            {
              //malloc printerr...
            }
          check_remalloced_chunk (av, victim, nb);
          void *p = chunk2mem (victim);
          alloc_perturb (p, bytes);
          return p;
        }
    }
```
而下方的`check_remalloced_chunk`只有开启了`MALLOC_DEBUG`才会存在，暂时不用管。  
#### 2.2 smallbin分配
如果`fastbin`中没有找到分配对象，则在`smallbin`中寻找，与`fastbin`不同，`smallbin`为双链表，所以从链表尾部直接取即可，然后做一个解链的操作，将其从双链表中拆下，拆下之前要检查链表完整性，因为下面的解链操作用的是bin，所以不检查victim->fd->bk是否为victim本身。  
```c
if (in_smallbin_range (nb))
    {
      idx = smallbin_index (nb);
      bin = bin_at (av, idx);

      if ((victim = last (bin)) != bin)//从smallbin的队尾取堆块
        {
          if (victim == 0)
            malloc_consolidate (av);
          else
            {
              bck = victim->bk;
	if (__glibc_unlikely (bck->fd != victim))
                {
                  errstr = "malloc(): smallbin double linked list corrupted";
                  goto errout;
                }
              set_inuse_bit_at_offset (victim, nb);
              bin->bk = bck;
              bck->fd = bin;

              if (av != &main_arena)
                victim->size |= NON_MAIN_ARENA;
              check_malloced_chunk (av, victim, nb);
              void *p = chunk2mem (victim);
              alloc_perturb (p, bytes);
              return p;
            }
        }
    }
```
在从`smallbin`中找寻堆块之后还没结束，有两种情况：一种是，申请的大小不是`smallbin`之内的；另一种是目标`smallbin`中没有空闲堆块供分配。  
这两种情况都会调用`malloc_consolidate`。  
该函数遍历所有的`fastbin`链表，将其中所有的堆块放入`unsorted bin`中，腾出空间，在这中间，还需要对每个堆块进行合并。  
对于前向（被合并的堆块在正在被分析的堆块的低地址处）合并，是以本堆块的`inuse bit`来判断的，`prev_size`来寻找的。  
对于后向（被合并的堆块在正在被分析的堆块的高地址处）合并，是以下一个堆块的`inuse bit`来判断的，本堆块的`size`来寻找的。  
如果有被合并的堆块，则需要调用`unlink`解链。  
```c
static void malloc_consolidate(mstate av)
{
    maxfb = &fastbin (av, NFASTBINS - 1);
    fb = &fastbin (av, 0);
    do {
      p = atomic_exchange_acq (fb, 0);
      if (p != 0) {
	do {
	  check_inuse_chunk(av, p);
	  nextp = p->fd;

	  size = p->size & ~(PREV_INUSE|NON_MAIN_ARENA);
	  nextchunk = chunk_at_offset(p, size);
	  nextsize = chunksize(nextchunk);
    //检查向前（低地址处）是否有相邻的自由堆块
	  if (!prev_inuse(p)) {
	    prevsize = p->prev_size;
	    size += prevsize;
	    p = chunk_at_offset(p, -((long) prevsize));
	    unlink(av, p, bck, fwd);
	  }

	  if (nextchunk != av->top) {
	    nextinuse = inuse_bit_at_offset(nextchunk, nextsize);
    //检查向后（高地址处）是否有相邻的自由堆块
	    if (!nextinuse) {
	      size += nextsize;
	      unlink(av, nextchunk, bck, fwd);
	    } else
	      clear_inuse_bit_at_offset(nextchunk, 0);

	    first_unsorted = unsorted_bin->fd;
	    unsorted_bin->fd = p;
	    first_unsorted->bk = p;

	    if (!in_smallbin_range (size)) {
	      p->fd_nextsize = NULL;
	      p->bk_nextsize = NULL;
	    }
      //将合成后的堆块插入unsortedbin
	    set_head(p, size | PREV_INUSE);
	    p->bk = unsorted_bin;
	    p->fd = first_unsorted;
	    set_foot(p, size);
	  }

	  else {
	    size += nextsize;
	    set_head(p, size | PREV_INUSE);
	    av->top = p;
	  }

	} while ( (p = nextp) != 0);

      }
    } while (fb++ != maxfb);
  }
  else {
    malloc_init_state(av);
    check_malloc_state(av);
  }

```
#### 2.3 unsorted bin分配
接下来是一个无限大循环，大循环中第一个while循环是对`unsorted bin`的遍历，对于每个`unsorted bin`的元素，首先检查其是否为是否可以切割（切割最后一个，且为`last remainder`的堆块）。  
```c
 if (in_smallbin_range (nb) &&
              bck == unsorted_chunks (av) &&
              victim == av->last_remainder &&
              (unsigned long) (size) > (unsigned long) (nb + MINSIZE))
            {
              /* split and reattach remainder */
              remainder_size = size - nb;
              remainder = chunk_at_offset (victim, nb);
              unsorted_chunks (av)->bk = unsorted_chunks (av)->fd = remainder;
              av->last_remainder = remainder;
              remainder->bk = remainder->fd = unsorted_chunks (av);
              if (!in_smallbin_range (remainder_size))
                {
                  remainder->fd_nextsize = NULL;
                  remainder->bk_nextsize = NULL;
                }

              set_head (victim, nb | PREV_INUSE |
                        (av != &main_arena ? NON_MAIN_ARENA : 0));
              set_head (remainder, remainder_size | PREV_INUSE);
              set_foot (remainder, remainder_size);

              check_malloced_chunk (av, victim, nb);
              void *p = chunk2mem (victim);
              alloc_perturb (p, bytes);
              return p;
            }
```
如果不能被切割，则从`unsorted bin`中解链，（这里的解链操作是存在问题的，没有校验完整性）
```c
unsorted_chunks (av)->bk = bck;
bck->fd = unsorted_chunks (av);
```
接下来，如果堆块大小满足就直接返回，如果不满足，则按照其大小插入`smallbin`或者`largebin`，然后在堆管理的位图中标识即可。（此处插入依然没有校验完整性），此时插入`largebin`时除了`fd`和`bk`指针还要对`nextsize`指针进行处理。  
### 2.4 largebin
现在终于到了在`largebin`中寻找的时候了，寻找适格堆块的方法就是按照size链表反向遍历。  
```c
while (((unsigned long) (size = chunksize (victim)) <
                      (unsigned long) (nb)))
                victim = victim->bk_nextsize;
```
然后将目标解链，如果剩余的size合适，则作为`last remainder`放入`unsorted bin`，如果不是则将整个堆块保留。  
```c
remainder_size = size - nb;
              unlink (av, victim, bck, fwd);

              /* Exhaust */
              if (remainder_size < MINSIZE)
                {
                  set_inuse_bit_at_offset (victim, size);
                  if (av != &main_arena)
                    victim->size |= NON_MAIN_ARENA;
                }
              /* Split */
              else
                {
                  remainder = chunk_at_offset (victim, nb);
                  /* We cannot assume the unsorted list is empty and therefore
                     have to perform a complete insert here.  */
                  bck = unsorted_chunks (av);
                  fwd = bck->fd;
	  if (__glibc_unlikely (fwd->bk != bck))
                    {
                      errstr = "malloc(): corrupted unsorted chunks";
                      goto errout;
                    }
                  remainder->bk = bck;
                  remainder->fd = fwd;
                  bck->fd = remainder;
                  fwd->bk = remainder;
                  if (!in_smallbin_range (remainder_size))
                    {
                      remainder->fd_nextsize = NULL;
                      remainder->bk_nextsize = NULL;
                    }
                  set_head (victim, nb | PREV_INUSE |
                            (av != &main_arena ? NON_MAIN_ARENA : 0));
                  set_head (remainder, remainder_size | PREV_INUSE);
                  set_foot (remainder, remainder_size);
                }
```
### 2.5 bitmap分配
如果此时还没能分配，那么只能说明：申请的堆块如对应的bin以及`unsorted`中没有free的堆块，并且`unsorted bin`中的`last remainder`不符合申请的要求。这种情况下只能从更加大的bin中发现有无新的堆块。  
```c
      ++idx;//更加大的index代表更大的bin，从第一个大于申请的bins开始找
      bin = bin_at (av, idx);
      block = idx2block (idx);//找到描述idx所处的block，block用来索引该区块bins的位图
      map = av->binmap[block];//得到描述目标bin所在区域的位图
      bit = idx2bit (idx);//得到目标bin对应的位图标识

      for (;; )
        {
          
          if (bit > map || bit == 0)//该区域没有合适的bin
            {
              do
                {
                  if (++block >= BINMAPSIZE) //转换到下个区域
                    goto use_top;
                }
              while ((map = av->binmap[block]) == 0);
              //存在bins的区域
              bin = bin_at (av, (block << BINMAPSHIFT));
              //从该区域最小的开始找
              bit = 1;
            }
          while ((bit & map) == 0)
            {
              bin = next_bin (bin);
              bit <<= 1;
              assert (bit != 0);
            }
          victim = last (bin);

          //记录错误，是空的bin
          if (victim == bin)
            {
              av->binmap[block] = map &= ~bit; /* Write through */
              bin = next_bin (bin);
              bit <<= 1;
            }
          else
            {
              size = chunksize (victim);

              /*  We know the first chunk in this bin is big enough to use. */
              assert ((unsigned long) (size) >= (unsigned long) (nb));

              remainder_size = size - nb;

              /* unlink */
              unlink (av, victim, bck, fwd);

              /* Exhaust */
              if (remainder_size < MINSIZE)
                {
                  set_inuse_bit_at_offset (victim, size);
                  if (av != &main_arena)
                    victim->size |= NON_MAIN_ARENA;
                }

              /* Split */
              else
                {
                  remainder = chunk_at_offset (victim, nb);

                  /* We cannot assume the unsorted list is empty and therefore
                     have to perform a complete insert here.  */
                  bck = unsorted_chunks (av);
                  fwd = bck->fd;
	  if (__glibc_unlikely (fwd->bk != bck))
                    {
                      errstr = "malloc(): corrupted unsorted chunks 2";
                      goto errout;
                    }
                  remainder->bk = bck;
                  remainder->fd = fwd;
                  bck->fd = remainder;
                  fwd->bk = remainder;

                  /* advertise as last remainder */
                  if (in_smallbin_range (nb))
                    av->last_remainder = remainder;
                  if (!in_smallbin_range (remainder_size))
                    {
                      remainder->fd_nextsize = NULL;
                      remainder->bk_nextsize = NULL;
                    }
                  set_head (victim, nb | PREV_INUSE |
                            (av != &main_arena ? NON_MAIN_ARENA : 0));
                  set_head (remainder, remainder_size | PREV_INUSE);
                  set_foot (remainder, remainder_size);
                }
              check_malloced_chunk (av, victim, nb);
              void *p = chunk2mem (victim);
              alloc_perturb (p, bytes);
              return p;
            }
        }
```

### 2.6 topchunk分配
运行至此，如果还是没能分配，就说明bin里没有这么大的堆块，只能从`topchunk`上分割进行分配了。如果`top chunk`还不够大那么就利用`sysmalloc`向系统申请内存了。  
## 二、free源码分析
`free`函数的重要调用层次结构如下:
```
__libc_free
            |----__free_hook
            |----_int_free
```
`free`的单线程活动部分比较简单，分为`fastbin`的释放和其它堆块释放两部分  
`fastbin`部分，将`fastbin chunk`放入对应bin的链表即可：
```c
if ((unsigned long)(size) <= (unsigned long)(get_max_fast ())
      && (chunk_at_offset(p, size) != av->top)) {

    unsigned int idx = fastbin_index(size);
    fb = &fastbin (av, idx);

    mchunkptr old = *fb, old2;
    unsigned int old_idx = ~0u;
    do
      {
	if (__builtin_expect (old == p, 0))
	  {
	    errstr = "double free or corruption (fasttop)";
	    goto errout;
	  }
	if (have_lock && old != NULL)
	  old_idx = fastbin_index(chunksize(old));
	p->fd = old2 = old;
      }
    while ((old = catomic_compare_and_exchange_val_rel (fb, p, old2)) != old2);

    if (have_lock && old != NULL && __builtin_expect (old_idx != idx, 0))
      {
	errstr = "invalid fastbin entry (free)";
	goto errout;
      }
  }
```
其他部分，查看被释放堆块的前后有没有相邻的释放堆块进行合并  
```c
if (!prev_inuse(p)) {
      prevsize = p->prev_size;
      size += prevsize;
      p = chunk_at_offset(p, -((long) prevsize));
      unlink(av, p, bck, fwd);
    }

    if (nextchunk != av->top) {
      /* get and clear inuse bit */
      nextinuse = inuse_bit_at_offset(nextchunk, nextsize);

      /* consolidate forward */
      if (!nextinuse) {
	unlink(av, nextchunk, bck, fwd);
	size += nextsize;
      } else
	clear_inuse_bit_at_offset(nextchunk, 0);
      bck = unsorted_chunks(av);
      fwd = bck->fd;
      if (__glibc_unlikely (fwd->bk != bck))
	{
	  errstr = "free(): corrupted unsorted chunks";
	  goto errout;
	}
      p->fd = fwd;
      p->bk = bck;
      if (!in_smallbin_range(size))
	{
	  p->fd_nextsize = NULL;
	  p->bk_nextsize = NULL;
	}
      bck->fd = p;
      fwd->bk = p;

      set_head(p, size | PREV_INUSE);
      set_foot(p, size);

      check_free_chunk(av, p);
    }
    else {
      size += nextsize;
      set_head(p, size | PREV_INUSE);
      av->top = p;
      check_chunk(av, p);
    }
``` 
上一篇文章[Android进程间通信（IPC）机制Binder简要介绍和学习计划](http://blog.csdn.net/luoshengyang/article/details/6618363)简要介绍了[Android](http://lib.csdn.net/base/android)系统进程间通信机制Binder的总体[架构](http://lib.csdn.net/base/architecture)，它由Client、Server、Service Manager和驱动程序Binder四个组件构成。本文着重介绍组件Service Manager，它是整个Binder机制的守护进程，用来管理开发者创建的各种Server，并且向Client提供查询Server远程接口的功能。

《[android](http://lib.csdn.net/base/android)系统源代码情景分析》一书正在进击的程序员网（[http://0xcc0xcd.com](http://0xcc0xcd.com/)）中连载，点击进入！

既然Service Manager组件是用来管理Server并且向Client提供查询Server远程接口的功能，那么，Service Manager就必然要和Server以及Client进行通信了。我们知道，Service Manger、Client和Server三者分别是运行在独立的进程当中，这样它们之间的通信也属于进程间通信了，而且也是采用Binder机制进行进程间通信，因此，Service Manager在充当Binder机制的守护进程的角色的同时，也在充当Server的角色，然而，它是一种特殊的Server，下面我们将会看到它的特殊之处。

与Service Manager相关的源代码较多，这里不会完整去分析每一行代码，主要是带着Service Manager是如何成为整个Binder机制中的守护进程这条主线来一步一步地深入分析相关源代码，包括从用户空间到内核空间的相关源代码。希望读者在阅读下面的内容之前，先阅读一下前一篇文章提到的两个参考资料[Android深入浅出之Binder机制](http://www.cnblogs.com/innost/archive/2011/01/09/1931456.html)和[Android Binder设计与实现](http://disanji.net/2011/02/28/android-bnder-design/)，熟悉相关概念和[数据结构](http://lib.csdn.net/base/datastructure)，这有助于理解下面要分析的源代码。

Service Manager在用户空间的源代码位于frameworks/base/cmds/servicemanager目录下，主要是由binder.h、binder.c和service_manager.c三个文件组成。Service Manager的入口位于service_manager.c文件中的main函数：

```c
int main(int argc, char **argv)  
{  
    struct binder_state *bs;  
    void *svcmgr = BINDER_SERVICE_MANAGER;  

    bs = binder_open(128*1024);  

    if (binder_become_context_manager(bs)) {  
LOGE("cannot become context manager (%s)\n", strerror(errno));  
return -1;  
    }  

    svcmgr_handle = svcmgr;  
    binder_loop(bs, svcmgr_handler);  
    return 0;  
}  
```
main函数主要有三个功能：一是打开Binder设备文件；二是告诉Binder驱动程序自己是Binder上下文管理者，即我们前面所说的守护进程；三是进入一个无穷循环，充当Server的角色，等待Client的请求。进入这三个功能之间，先来看一下这里用到的结构体`binder_state`、宏 `BINDER_SERVICE_MANAGER` 的定义：

struct binder_state定义在frameworks/base/cmds/servicemanager/binder.c文件中：

```c
struct binder_state  
{  
    int fd;  
    void *mapped;  
    unsigned mapsize;  
};  
```
fd是文件描述符，即表示打开的/dev/binder设备文件描述符；mapped是把设备文件/dev/binder映射到进程空间的起始地址；mapsize是上述内存映射空间的大小。

宏 `BINDER_SERVICE_MANAGER` 定义frameworks/base/cmds/servicemanager/binder.h文件中：

```c
/* the one magic object */  
#define BINDER_SERVICE_MANAGER ((void*) 0)  
```
它表示Service Manager的句柄为0。Binder通信机制使用句柄来代表远程接口，这个句柄的意义和Windows编程中用到的句柄是差不多的概念。前面说到，Service Manager在充当守护进程的同时，它充当Server的角色，当它作为远程接口使用时，它的句柄值便为0，这就是它的特殊之处，其余的Server的远程接口句柄值都是一个大于0 而且由Binder驱动程序自动进行分配的。

函数首先是执行打开Binder设备文件的操作`binder_open`，这个函数位于frameworks/base/cmds/servicemanager/binder.c文件中：

```c
struct binder_state *binder_open(unsigned mapsize)  
{  
    struct binder_state *bs;  

    bs = malloc(sizeof(*bs));  
    if (!bs) {  
errno = ENOMEM;  
return 0;  
    }  

    bs->fd = open("/dev/binder", O_RDWR);  
    if (bs->fd < 0) {  
fprintf(stderr,"binder: cannot open device (%s)\n",  
        strerror(errno));  
goto fail_open;  
    }  

    bs->mapsize = mapsize;  
    bs->mapped = mmap(NULL, mapsize, PROT_READ, MAP_PRIVATE, bs->fd, 0);  
    if (bs->mapped == MAP_FAILED) {  
fprintf(stderr,"binder: cannot map device (%s)\n",  
        strerror(errno));  
goto fail_map;  
    }  

/* TODO: check version */  

    return bs;  

fail_map:  
    close(bs->fd);  
fail_open:  
    free(bs);  
    return 0;  
}  
```
通过文件操作函数open来打开/dev/binder设备文件。设备文件/dev/binder是在Binder驱动程序模块初始化的时候创建的，我们先看一下这个设备文件的创建过程。进入到kernel/common/drivers/staging/android目录中，打开binder.c文件，可以看到模块初始化入口 `binder_init` ：

```c
static struct file_operations binder_fops = {  
    .owner = THIS_MODULE,  
    .poll = binder_poll,  
    .unlocked_ioctl = binder_ioctl,  
    .mmap = binder_mmap,  
    .open = binder_open,  
    .flush = binder_flush,  
    .release = binder_release,  
};  

static struct miscdevice binder_miscdev = {  
    .minor = MISC_DYNAMIC_MINOR,  
    .name = "binder",  
    .fops = &binder_fops  
};  

static int __init binder_init(void)  
{  
    int ret;  

    binder_proc_dir_entry_root = proc_mkdir("binder", NULL);  
    if (binder_proc_dir_entry_root)  
binder_proc_dir_entry_proc = proc_mkdir("proc", binder_proc_dir_entry_root);  
    ret = misc_register(&binder_miscdev);  
    if (binder_proc_dir_entry_root) {  
create_proc_read_entry("state", S_IRUGO, binder_proc_dir_entry_root, binder_read_proc_state, NULL);  
create_proc_read_entry("stats", S_IRUGO, binder_proc_dir_entry_root, binder_read_proc_stats, NULL);  
create_proc_read_entry("transactions", S_IRUGO, binder_proc_dir_entry_root, binder_read_proc_transactions, NULL);  
create_proc_read_entry("transaction_log", S_IRUGO, binder_proc_dir_entry_root, binder_read_proc_transaction_log, &binder_transaction_log);  
create_proc_read_entry("failed_transaction_log", S_IRUGO, binder_proc_dir_entry_root, binder_read_proc_transaction_log, &binder_transaction_log_failed);  
    }  
    return ret;  
}  

device_initcall(binder_init);  
```
创建设备文件的地方在 `misc_register` 函数里面，关于misc设备的注册，我们在[Android日志系统驱动程序Logger源代码分析](http://blog.csdn.net/luoshengyang/article/details/6595744)一文中有提到，有兴趣的读取不访去了解一下。其余的逻辑主要是在/proc目录创建各种Binder相关的文件，供用户访问。从设备文件的操作方法 `binder_fops` 可以看出，前面的 `binder_open` 函数执行语句：

```c
bs->fd = open("/dev/binder", O_RDWR);  
```
就进入到Binder驱动程序的binder_open函数了：

```c
static int binder_open(struct inode *nodp, struct file *filp)  
{  
    struct binder_proc *proc;  

    if (binder_debug_mask & BINDER_DEBUG_OPEN_CLOSE)  
printk(KERN_INFO "binder_open: %d:%d\n", current->group_leader->pid, current->pid);  

    proc = kzalloc(sizeof(*proc), GFP_KERNEL);  
    if (proc == NULL)  
return -ENOMEM;  
    get_task_struct(current);  
    proc->tsk = current;  
    INIT_LIST_HEAD(&proc->todo);  
    init_waitqueue_head(&proc->wait);  
    proc->default_priority = task_nice(current);  
    mutex_lock(&binder_lock);  
    binder_stats.obj_created[BINDER_STAT_PROC]++;  
    hlist_add_head(&proc->proc_node, &binder_procs);  
    proc->pid = current->group_leader->pid;  
    INIT_LIST_HEAD(&proc->delivered_death);  
    filp->private_data = proc;  
    mutex_unlock(&binder_lock);  

    if (binder_proc_dir_entry_proc) {  
char strbuf[11];  
snprintf(strbuf, sizeof(strbuf), "%u", proc->pid);  
remove_proc_entry(strbuf, binder_proc_dir_entry_proc);  
create_proc_read_entry(strbuf, S_IRUGO, binder_proc_dir_entry_proc, binder_read_proc_proc, proc);  
    }  

    return 0;  
}  
```
这个函数的主要作用是创建一个struct binder_proc数据结构来保存打开设备文件/dev/binder的进程的上下文信息，并且将这个进程上下文信息保存在打开文件结构struct file的私有数据成员变量 `private_data` 中，这样，在执行其它文件操作时，就通过打开文件结构struct file来取回这个进程上下文信息了。这个进程上下文信息同时还会保存在一个全局哈希表 `binder_procs` 中，驱动程序内部使用。 `binder_procs` 定义在文件的开头：

```c
static HLIST_HEAD(binder_procs);  
```
结构体struct binder_proc也是定义在kernel/common/drivers/staging/android/binder.c文件中：

```c
struct binder_proc {  
    struct hlist_node proc_node;  
    struct rb_root threads;  
    struct rb_root nodes;  
    struct rb_root refs_by_desc;  
    struct rb_root refs_by_node;  
    int pid;  
    struct vm_area_struct *vma;  
    struct task_struct *tsk;  
    struct files_struct *files;  
    struct hlist_node deferred_work_node;  
    int deferred_work;  
    void *buffer;  
    ptrdiff_t user_buffer_offset;  

    struct list_head buffers;  
    struct rb_root free_buffers;  
    struct rb_root allocated_buffers;  
    size_t free_async_space;  

    struct page **pages;  
    size_t buffer_size;  
    uint32_t buffer_free;  
    struct list_head todo;  
    wait_queue_head_t wait;  
    struct binder_stats stats;  
    struct list_head delivered_death;  
    int max_threads;  
    int requested_threads;  
    int requested_threads_started;  
    int ready_threads;  
    long default_priority;  
};  
```
这个结构体的成员比较多，这里就不一一说明了，简单解释一下四个成员变量threads、nodes、  `refs_by_desc` 和 `refs_by_node` ，其它的我们在遇到的时候再详细解释。这四个成员变量都是表示红黑树的节点，也就是说， `binder_proc` 分别挂会四个红黑树下。threads树用来保存binder_proc进程内用于处理用户请求的线程，它的最大数量由 `max_threads` 来决定；node树成用来保存binder_proc进程内的Binder实体； `refs_by_desc` 树和 `refs_by_node` 树用来保存 `binder_proc` 进程内的Binder引用，即引用的其它进程的Binder实体，它分别用两种方式来组织红黑树，一种是以句柄作来key值来组织，一种是以引用的实体节点的地址值作来key值来组织，它们都是表示同一样东西，只不过是为了内部查找方便而用两个红黑树来表示。

这样，打开设备文件/dev/binder的操作就完成了，接着是对打开的设备文件进行内存映射操作mmap：

```c
bs->mapped = mmap(NULL, mapsize, PROT_READ, MAP_PRIVATE, bs->fd, 0);  
```
对应Binder驱动程序的binder_mmap函数：

```c
static int binder_mmap(struct file *filp, struct vm_area_struct *vma)  
{  
    int ret;  
    struct vm_struct *area;  
    struct binder_proc *proc = filp->private_data;  
    const char *failure_string;  
    struct binder_buffer *buffer;  

    if ((vma->vm_end - vma->vm_start) > SZ_4M)  
vma->vm_end = vma->vm_start + SZ_4M;  

    if (binder_debug_mask & BINDER_DEBUG_OPEN_CLOSE)  
printk(KERN_INFO  
    "binder_mmap: %d %lx-%lx (%ld K) vma %lx pagep %lx\n",  
    proc->pid, vma->vm_start, vma->vm_end,  
    (vma->vm_end - vma->vm_start) / SZ_1K, vma->vm_flags,  
    (unsigned long)pgprot_val(vma->vm_page_prot));  

    if (vma->vm_flags & FORBIDDEN_MMAP_FLAGS) {  
ret = -EPERM;  
failure_string = "bad vm_flags";  
goto err_bad_arg;  
    }  
    vma->vm_flags = (vma->vm_flags | VM_DONTCOPY) & ~VM_MAYWRITE;  

    if (proc->buffer) {  
ret = -EBUSY;  
failure_string = "already mapped";  
goto err_already_mapped;  
    }  

    area = get_vm_area(vma->vm_end - vma->vm_start, VM_IOREMAP);  
    if (area == NULL) {  
ret = -ENOMEM;  
failure_string = "get_vm_area";  
goto err_get_vm_area_failed;  
    }  
    proc->buffer = area->addr;  
    proc->user_buffer_offset = vma->vm_start - (uintptr_t)proc->buffer;  

#ifdef CONFIG_CPU_CACHE_VIPT  
    if (cache_is_vipt_aliasing()) {  
while (CACHE_COLOUR((vma->vm_start ^ (uint32_t)proc->buffer))) {  
    printk(KERN_INFO "binder_mmap: %d %lx-%lx maps %p bad alignment\n", proc->pid, vma->vm_start, vma->vm_end, proc->buffer);  
    vma->vm_start += PAGE_SIZE;  
}  
    }  
#endif  
    proc->pages = kzalloc(sizeof(proc->pages[0]) * ((vma->vm_end - vma->vm_start) / PAGE_SIZE), GFP_KERNEL);  
    if (proc->pages == NULL) {  
ret = -ENOMEM;  
failure_string = "alloc page array";  
goto err_alloc_pages_failed;  
    }  
    proc->buffer_size = vma->vm_end - vma->vm_start;  

    vma->vm_ops = &binder_vm_ops;  
    vma->vm_private_data = proc;  

    if (binder_update_page_range(proc, 1, proc->buffer, proc->buffer + PAGE_SIZE, vma)) {  
ret = -ENOMEM;  
failure_string = "alloc small buf";  
goto err_alloc_small_buf_failed;  
    }  
    buffer = proc->buffer;  
    INIT_LIST_HEAD(&proc->buffers);  
    list_add(&buffer->entry, &proc->buffers);  
    buffer->free = 1;  
    binder_insert_free_buffer(proc, buffer);  
    proc->free_async_space = proc->buffer_size / 2;  
    barrier();  
    proc->files = get_files_struct(current);  
    proc->vma = vma;  

    /*printk(KERN_INFO "binder_mmap: %d %lx-%lx maps %p\n", proc->pid, vma->vm_start, vma->vm_end, proc->buffer);*/  
    return 0;  

err_alloc_small_buf_failed:  
    kfree(proc->pages);  
    proc->pages = NULL;  
err_alloc_pages_failed:  
    vfree(proc->buffer);  
    proc->buffer = NULL;  
err_get_vm_area_failed:  
err_already_mapped:  
err_bad_arg:  
    printk(KERN_ERR "binder_mmap: %d %lx-%lx %s failed %d\n", proc->pid, vma->vm_start, vma->vm_end, failure_string, ret);  
    return ret;  
}  
```
函数首先通过filp->private_data得到在打开设备文件/dev/binder时创建的struct binder_proc结构。内存映射信息放在vma参数中，注意，这里的vma的数据类型是struct vm_area_struct，它表示的是一块连续的虚拟地址空间区域，在函数变量声明的地方，我们还看到有一个类似的结构体struct vm_struct，这个数据结构也是表示一块连续的虚拟地址空间区域，那么，这两者的区别是什么呢？在[Linux](http://lib.csdn.net/base/linux)中，struct vm_area_struct表示的虚拟地址是给进程使用的，而struct vm_struct表示的虚拟地址是给内核使用的，它们对应的物理页面都可以是不连续的。struct vm_area_struct表示的地址空间范围是0~3G，而struct vm_struct表示的地址空间范围是(3G + 896M + 8M) ~ 4G。struct vm_struct表示的地址空间范围为什么不是3G~4G呢？原来，3G ~ (3G + 896M)范围的地址是用来映射连续的物理页面的，这个范围的虚拟地址和对应的实际物理地址有着简单的对应关系，即对应0~896M的物理地址空间，而(3G + 896M) ~ (3G + 896M + 8M)是安全保护区域（例如，所有指向这8M地址空间的指针都是非法的），因此struct vm_struct使用(3G + 896M + 8M) ~ 4G地址空间来映射非连续的物理页面。有关[linux](http://lib.csdn.net/base/linux)的内存管理知识，可以参考[Android学习启动篇](http://blog.csdn.net/luoshengyang/article/details/6557518)一文提到的《Understanding the Linux Kernel》一书中的第8章。

这里为什么会同时使用进程虚拟地址空间和内核虚拟地址空间来映射同一个物理页面呢？这就是Binder进程间通信机制的精髓所在了，同一个物理页面，一方映射到进程虚拟地址空间，一方面映射到内核虚拟地址空间，这样，进程和内核之间就可以减少一次内存拷贝了，提到了进程间通信效率。举个例子如，Client要将一块内存数据传递给Server，一般的做法是，Client将这块数据从它的进程空间拷贝到内核空间中，然后内核再将这个数据从内核空间拷贝到Server的进程空间，这样，Server就可以访问这个数据了。但是在这种方法中，执行了两次内存拷贝操作，而采用我们上面提到的方法，只需要把Client进程空间的数据拷贝一次到内核空间，然后Server与内核共享这个数据就可以了，整个过程只需要执行一次内存拷贝，提高了效率。

binder_mmap的原理讲完了，这个函数的逻辑就好理解了。不过，这里还是先要解释一下struct binder_proc结构体的几个成员变量。buffer成员变量是一个void*指针，它表示要映射的物理内存在内核空间中的起始位置；buffer_size成员变量是一个size_t类型的变量，表示要映射的内存的大小；pages成员变量是一个struct page*类型的数组，struct page是用来描述物理页面的数据结构； `user_buffer_offset` 成员变量是一个ptrdiff_t类型的变量，它表示的是内核使用的虚拟地址与进程使用的虚拟地址之间的差值，即如果某个物理页面在内核空间中对应的虚拟地址是addr的话，那么这个物理页面在进程空间对应的虚拟地址就为addr + user_buffer_offset。

再解释一下Binder驱动程序管理这个内存映射地址空间的方法，即是如何管理buffer ~ (buffer + buffer_size)这段地址空间的，这个地址空间被划分为一段一段来管理，每一段是结构体struct binder_buffer来描述：

```c
struct binder_buffer {  
    struct list_head entry; /* free and allocated entries by addesss */  
    struct rb_node rb_node; /* free entry by size or allocated entry */  
        /* by address */  
    unsigned free : 1;  
    unsigned allow_user_free : 1;  
    unsigned async_transaction : 1;  
    unsigned debug_id : 29;  

    struct binder_transaction *transaction;  

    struct binder_node *target_node;  
    size_t data_size;  
    size_t offsets_size;  
    uint8_t data[0];  
};  
```
每一个binder_buffer通过其成员entry按从低址到高地址连入到struct binder_proc中的buffers表示的链表中去，同时，每一个binder_buffer又分为正在使用的和空闲的，通过free成员变量来区分，空闲的binder_buffer通过成员变量rb_node连入到struct binder_proc中的free_buffers表示的红黑树中去，正在使用的binder_buffer通过成员变量rb_node连入到struct binder_proc中的allocated_buffers表示的红黑树中去。这样做当然是为了方便查询和维护这块地址空间了，这一点我们可以从其它的代码中看到，等遇到的时候我们再分析。

终于可以回到binder_mmap这个函数来了，首先是对参数作一些健康体检（sanity check），例如，要映射的内存大小不能超过SIZE_4M，即4M，回到service_manager.c中的main 函数，这里传进来的值是128 * 1024个字节，即128K，这个检查没有问题。通过健康体检后，调用 `get_vm_area` 函数获得一个空闲的vm_struct区间，并初始化proc结构体的buffer、 `user_buffer_offset` 、pages和buffer_size和成员变量，接着调用 `binder_update_page` _range来为虚拟地址空间proc->buffer ~ proc->buffer + PAGE_SIZE分配一个空闲的物理页面，同时这段地址空间使用一个binder_buffer来描述，分别插入到proc->buffers链表和proc->free_buffers红黑树中去，最后，还初始化了proc结构体的 `free_async_space` 、files和vma三个成员变量。

这里，我们继续进入到binder_update_page_range函数中去看一下Binder驱动程序是如何实现把一个物理页面同时映射到内核空间和进程空间去的：

```c
static int binder_update_page_range(struct binder_proc *proc, int allocate,  
    void *start, void *end, struct vm_area_struct *vma)  
{  
    void *page_addr;  
    unsigned long user_page_addr;  
    struct vm_struct tmp_area;  
    struct page **page;  
    struct mm_struct *mm;  

    if (binder_debug_mask & BINDER_DEBUG_BUFFER_ALLOC)  
printk(KERN_INFO "binder: %d: %s pages %p-%p\n",  
       proc->pid, allocate ? "allocate" : "free", start, end);  

    if (end <= start)  
return 0;  

    if (vma)  
mm = NULL;  
    else  
mm = get_task_mm(proc->tsk);  

    if (mm) {  
down_write(&mm->mmap_sem);  
vma = proc->vma;  
    }  

    if (allocate == 0)  
goto free_range;  

    if (vma == NULL) {  
printk(KERN_ERR "binder: %d: binder_alloc_buf failed to "  
       "map pages in userspace, no vma\n", proc->pid);  
goto err_no_vma;  
    }  

    for (page_addr = start; page_addr < end; page_addr += PAGE_SIZE) {  
int ret;  
struct page **page_array_ptr;  
page = &proc->pages[(page_addr - proc->buffer) / PAGE_SIZE];  

BUG_ON(*page);  
*page = alloc_page(GFP_KERNEL | __GFP_ZERO);  
if (*page == NULL) {  
    printk(KERN_ERR "binder: %d: binder_alloc_buf failed "  
           "for page at %p\n", proc->pid, page_addr);  
    goto err_alloc_page_failed;  
}  
tmp_area.addr = page_addr;  
tmp_area.size = PAGE_SIZE + PAGE_SIZE /* guard page? */;  
page_array_ptr = page;  
ret = map_vm_area(&tmp_area, PAGE_KERNEL, &page_array_ptr);  
if (ret) {  
    printk(KERN_ERR "binder: %d: binder_alloc_buf failed "  
           "to map page at %p in kernel\n",  
           proc->pid, page_addr);  
    goto err_map_kernel_failed;  
}  
user_page_addr =  
    (uintptr_t)page_addr + proc->user_buffer_offset;  
ret = vm_insert_page(vma, user_page_addr, page[0]);  
if (ret) {  
    printk(KERN_ERR "binder: %d: binder_alloc_buf failed "  
           "to map page at %lx in userspace\n",  
           proc->pid, user_page_addr);  
    goto err_vm_insert_page_failed;  
}  
/* vm_insert_page does not seem to increment the refcount */  
    }  
    if (mm) {  
up_write(&mm->mmap_sem);  
mmput(mm);  
    }  
    return 0;  

free_range:  
    for (page_addr = end - PAGE_SIZE; page_addr >= start;  
 page_addr -= PAGE_SIZE) {  
page = &proc->pages[(page_addr - proc->buffer) / PAGE_SIZE];  
if (vma)  
    zap_page_range(vma, (uintptr_t)page_addr +  
        proc->user_buffer_offset, PAGE_SIZE, NULL);  
err_vm_insert_page_failed:  
unmap_kernel_range((unsigned long)page_addr, PAGE_SIZE);  
err_map_kernel_failed:  
__free_page(*page);  
*page = NULL;  
err_alloc_page_failed:  
;  
    }  
err_no_vma:  
    if (mm) {  
up_write(&mm->mmap_sem);  
mmput(mm);  
    }  
    return -ENOMEM;  
}  
```
这个函数既可以分配物理页面，也可以用来释放物理页面，通过allocate参数来区别，这里我们只关注分配物理页面的情况。要分配物理页面的虚拟地址空间范围为(start ~ end)，函数前面的一些检查逻辑就不看了，直接看中间的for循环：

```c
for (page_addr = start; page_addr < end; page_addr += PAGE_SIZE) {  
    int ret;  
    struct page **page_array_ptr;  
    page = &proc->pages[(page_addr - proc->buffer) / PAGE_SIZE];  

    BUG_ON(*page);  
    *page = alloc_page(GFP_KERNEL | __GFP_ZERO);  
    if (*page == NULL) {  
printk(KERN_ERR "binder: %d: binder_alloc_buf failed "  
       "for page at %p\n", proc->pid, page_addr);  
goto err_alloc_page_failed;  
    }  
    tmp_area.addr = page_addr;  
    tmp_area.size = PAGE_SIZE + PAGE_SIZE /* guard page? */;  
    page_array_ptr = page;  
    ret = map_vm_area(&tmp_area, PAGE_KERNEL, &page_array_ptr);  
    if (ret) {  
printk(KERN_ERR "binder: %d: binder_alloc_buf failed "  
       "to map page at %p in kernel\n",  
       proc->pid, page_addr);  
goto err_map_kernel_failed;  
    }  
    user_page_addr =  
(uintptr_t)page_addr + proc->user_buffer_offset;  
    ret = vm_insert_page(vma, user_page_addr, page[0]);  
    if (ret) {  
printk(KERN_ERR "binder: %d: binder_alloc_buf failed "  
       "to map page at %lx in userspace\n",  
       proc->pid, user_page_addr);  
goto err_vm_insert_page_failed;  
    }  
    /* vm_insert_page does not seem to increment the refcount */  
}  
```
首先是调用alloc_page来分配一个物理页面，这个函数返回一个struct page物理页面描述符，根据这个描述的内容初始化好struct vm_struct tmp_area结构体，然后通过 `map_vm_area` 将这个物理页面插入到tmp_area描述的内核空间去，接着通过page_addr + proc->user_buffer_offset获得进程虚拟空间地址，并通过 `vm_insert_page` 函数将这个物理页面插入到进程地址空间去，参数vma代表了要插入的进程的地址空间。

这样，frameworks/base/cmds/servicemanager/binder.c文件中的binder_open函数就描述完了，回到frameworks/base/cmds/servicemanager/service_manager.c文件中的main函数，下一步就是调用binder_become_context_manager来通知Binder驱动程序自己是Binder机制的上下文管理者，即守护进程。binder_become_context_manager函数位于frameworks/base/cmds/servicemanager/binder.c文件中：

```c
int binder_become_context_manager(struct binder_state *bs)  
{  
    return ioctl(bs->fd, BINDER_SET_CONTEXT_MGR, 0);  
}  
```
这里通过调用ioctl文件操作函数来通知Binder驱动程序自己是守护进程，命令号是BINDER_SET_CONTEXT_MGR，没有参数。BINDER_SET_CONTEXT_MGR定义为：

```c
#define BINDER_SET_CONTEXT_MGR      _IOW('b', 7, int)  
```
这样就进入到Binder驱动程序的binder_ioctl函数，我们只关注BINDER_SET_CONTEXT_MGR命令：

```c
static long binder_ioctl(struct file *filp, unsigned int cmd, unsigned long arg)  
{  
    int ret;  
    struct binder_proc *proc = filp->private_data;  
    struct binder_thread *thread;  
    unsigned int size = _IOC_SIZE(cmd);  
    void __user *ubuf = (void __user *)arg;  

    /*printk(KERN_INFO "binder_ioctl: %d:%d %x %lx\n", proc->pid, current->pid, cmd, arg);*/  

    ret = wait_event_interruptible(binder_user_error_wait, binder_stop_on_user_error < 2);  
    if (ret)  
return ret;  

    mutex_lock(&binder_lock);  
    thread = binder_get_thread(proc);  
    if (thread == NULL) {  
ret = -ENOMEM;  
goto err;  
    }  

    switch (cmd) {  
......  
    case BINDER_SET_CONTEXT_MGR:  
if (binder_context_mgr_node != NULL) {  
    printk(KERN_ERR "binder: BINDER_SET_CONTEXT_MGR already set\n");  
    ret = -EBUSY;  
    goto err;  
}  
if (binder_context_mgr_uid != -1) {  
    if (binder_context_mgr_uid != current->cred->euid) {  
        printk(KERN_ERR "binder: BINDER_SET_"  
            "CONTEXT_MGR bad uid %d != %d\n",  
            current->cred->euid,  
            binder_context_mgr_uid);  
        ret = -EPERM;  
        goto err;  
    }  
} else  
    binder_context_mgr_uid = current->cred->euid;  
binder_context_mgr_node = binder_new_node(proc, NULL, NULL);  
if (binder_context_mgr_node == NULL) {  
    ret = -ENOMEM;  
    goto err;  
}  
binder_context_mgr_node->local_weak_refs++;  
binder_context_mgr_node->local_strong_refs++;  
binder_context_mgr_node->has_strong_ref = 1;  
binder_context_mgr_node->has_weak_ref = 1;  
break;  
......  
    default:  
ret = -EINVAL;  
goto err;  
    }  
    ret = 0;  
err:  
    if (thread)  
thread->looper &= ~BINDER_LOOPER_STATE_NEED_RETURN;  
    mutex_unlock(&binder_lock);  
    wait_event_interruptible(binder_user_error_wait, binder_stop_on_user_error < 2);  
    if (ret && ret != -ERESTARTSYS)  
printk(KERN_INFO "binder: %d:%d ioctl %x %lx returned %d\n", proc->pid, current->pid, cmd, arg, ret);  
    return ret;  
}  
```
继续分析这个函数之前，又要解释两个数据结构了，一个是struct binder_thread结构体，顾名思久，它表示一个线程，这里就是执行binder_become_context_manager函数的线程了。

```c
struct binder_thread {  
    struct binder_proc *proc;  
    struct rb_node rb_node;  
    int pid;  
    int looper;  
    struct binder_transaction *transaction_stack;  
    struct list_head todo;  
    uint32_t return_error; /* Write failed, return error code in read buf */  
    uint32_t return_error2; /* Write failed, return error code in read */  
/* buffer. Used when sending a reply to a dead process that */  
/* we are also waiting on */  
    wait_queue_head_t wait;  
    struct binder_stats stats;  
};  
```
proc表示这个线程所属的进程。struct binder_proc有一个成员变量threads，它的类型是rb_root，它表示一查红黑树，把属于这个进程的所有线程都组织起来，struct binder_thread的成员变量rb_node就是用来链入这棵红黑树的节点了。looper成员变量表示线程的状态，它可以取下面这几个值：

```c
enum {  
    BINDER_LOOPER_STATE_REGISTERED  = 0x01,  
    BINDER_LOOPER_STATE_ENTERED     = 0x02,  
    BINDER_LOOPER_STATE_EXITED      = 0x04,  
    BINDER_LOOPER_STATE_INVALID     = 0x08,  
    BINDER_LOOPER_STATE_WAITING     = 0x10,  
    BINDER_LOOPER_STATE_NEED_RETURN = 0x20  
};  
```
其余的成员变量，transaction_stack表示线程正在处理的事务，todo表示发往该线程的数据列表，return_error和return_error2表示操作结果返回码，wait用来阻塞线程等待某个事件的发生，stats用来保存一些统计信息。这些成员变量遇到的时候再分析它们的作用。

另外一个数据结构是struct binder_node，它表示一个binder实体：

```c
struct binder_node {  
    int debug_id;  
    struct binder_work work;  
    union {  
struct rb_node rb_node;  
struct hlist_node dead_node;  
    };  
    struct binder_proc *proc;  
    struct hlist_head refs;  
    int internal_strong_refs;  
    int local_weak_refs;  
    int local_strong_refs;  
    void __user *ptr;  
    void __user *cookie;  
    unsigned has_strong_ref : 1;  
    unsigned pending_strong_ref : 1;  
    unsigned has_weak_ref : 1;  
    unsigned pending_weak_ref : 1;  
    unsigned has_async_transaction : 1;  
    unsigned accept_fds : 1;  
    int min_priority : 8;  
    struct list_head async_todo;  
};  
```
rb_node和dead_node组成一个联合体。 如果这个Binder实体还在正常使用，则使用rb_node来连入proc->nodes所表示的红黑树的节点，这棵红黑树用来组织属于这个进程的所有Binder实体；如果这个Binder实体所属的进程已经销毁，而这个Binder实体又被其它进程所引用，则这个Binder实体通过dead_node进入到一个哈希表中去存放。proc成员变量就是表示这个Binder实例所属于进程了。refs成员变量把所有引用了该Binder实体的Binder引用连接起来构成一个链表。internal_strong_refs、local_weak_refs和local_strong_refs表示这个Binder实体的引用计数。ptr和cookie成员变量分别表示这个Binder实体在用户空间的地址以及附加数据。其余的成员变量就不描述了，遇到的时候再分析。

现在回到binder_ioctl函数中，首先是通过filp->private_data获得proc变量，这里binder_mmap函数是一样的。接着通过binder_get_thread函数获得线程信息，我们来看一下这个函数：

```c
static struct binder_thread *binder_get_thread(struct binder_proc *proc)  
{  
    struct binder_thread *thread = NULL;  
    struct rb_node *parent = NULL;  
    struct rb_node **p = &proc->threads.rb_node;  

    while (*p) {  
parent = *p;  
thread = rb_entry(parent, struct binder_thread, rb_node);  

if (current->pid < thread->pid)  
    p = &(*p)->rb_left;  
else if (current->pid > thread->pid)  
    p = &(*p)->rb_right;  
else  
    break;  
    }  
    if (*p == NULL) {  
thread = kzalloc(sizeof(*thread), GFP_KERNEL);  
if (thread == NULL)  
    return NULL;  
binder_stats.obj_created[BINDER_STAT_THREAD]++;  
thread->proc = proc;  
thread->pid = current->pid;  
init_waitqueue_head(&thread->wait);  
INIT_LIST_HEAD(&thread->todo);  
rb_link_node(&thread->rb_node, parent, p);  
rb_insert_color(&thread->rb_node, &proc->threads);  
thread->looper |= BINDER_LOOPER_STATE_NEED_RETURN;  
thread->return_error = BR_OK;  
thread->return_error2 = BR_OK;  
    }  
    return thread;  
}  
```
这里把当前线程current的pid作为键值，在进程proc->threads表示的红黑树中进行查找，看是否已经为当前线程创建过了binder_thread信息。在这个场景下，由于当前线程是第一次进到这里，所以肯定找不到，即*p == NULL成立，于是，就为当前线程创建一个线程上下文信息结构体binder_thread，并初始化相应成员变量，并插入到proc->threads所表示的红黑树中去，下次要使用时就可以从proc中找到了。注意，这里的thread->looper = BINDER_LOOPER_STATE_NEED_RETURN。

回到binder_ioctl函数，继续往下面，有两个全局变量binder_context_mgr_node和binder_context_mgr_uid，它定义如下：

```c
static struct binder_node *binder_context_mgr_node;  
static uid_t binder_context_mgr_uid = -1;  
```
binder_context_mgr_node用来表示Service Manager实体，binder_context_mgr_uid表示Service Manager守护进程的uid。在这个场景下，由于当前线程是第一次进到这里，所以binder_context_mgr_node为NULL，binder_context_mgr_uid为-1，于是初始化binder_context_mgr_uid为current->cred->euid，这样，当前线程就成为Binder机制的守护进程了，并且通过binder_new_node为Service Manager创建Binder实体：

```c
static struct binder_node *  
binder_new_node(struct binder_proc *proc, void __user *ptr, void __user *cookie)  
{  
    struct rb_node **p = &proc->nodes.rb_node;  
    struct rb_node *parent = NULL;  
    struct binder_node *node;  

    while (*p) {  
parent = *p;  
node = rb_entry(parent, struct binder_node, rb_node);  

if (ptr < node->ptr)  
    p = &(*p)->rb_left;  
else if (ptr > node->ptr)  
    p = &(*p)->rb_right;  
else  
    return NULL;  
    }  

    node = kzalloc(sizeof(*node), GFP_KERNEL);  
    if (node == NULL)  
return NULL;  
    binder_stats.obj_created[BINDER_STAT_NODE]++;  
    rb_link_node(&node->rb_node, parent, p);  
    rb_insert_color(&node->rb_node, &proc->nodes);  
    node->debug_id = ++binder_last_id;  
    node->proc = proc;  
    node->ptr = ptr;  
    node->cookie = cookie;  
    node->work.type = BINDER_WORK_NODE;  
    INIT_LIST_HEAD(&node->work.entry);  
    INIT_LIST_HEAD(&node->async_todo);  
    if (binder_debug_mask & BINDER_DEBUG_INTERNAL_REFS)  
printk(KERN_INFO "binder: %d:%d node %d u%p c%p created\n",  
       proc->pid, current->pid, node->debug_id,  
       node->ptr, node->cookie);  
    return node;  
}  
```
注意，这里传进来的ptr和cookie均为NULL。函数首先检查proc->nodes红黑树中是否已经存在以ptr为键值的node，如果已经存在，就返回NULL。在这个场景下，由于当前线程是第一次进入到这里，所以肯定不存在，于是就新建了一个ptr为NULL的binder_node，并且初始化其它成员变量，并插入到proc->nodes红黑树中去。

binder_new_node返回到binder_ioctl函数后，就把新建的binder_node指针保存在binder_context_mgr_node中了，紧接着，又初始化了binder_context_mgr_node的引用计数值。

这样，BINDER_SET_CONTEXT_MGR命令就执行完毕了，binder_ioctl函数返回之前，执行了下面语句：

```c
if (thread)  
thread->looper &= ~BINDER_LOOPER_STATE_NEED_RETURN;  
```
回忆上面执行binder_get_thread时，thread->looper = BINDER_LOOPER_STATE_NEED_RETURN，执行了这条语句后，thread->looper = 0。

回到frameworks/base/cmds/servicemanager/service_manager.c文件中的main函数，下一步就是调用binder_loop函数进入循环，等待Client来请求了。binder_loop函数定义在frameworks/base/cmds/servicemanager/binder.c文件中：

```c
void binder_loop(struct binder_state *bs, binder_handler func)  
{  
    int res;  
    struct binder_write_read bwr;  
    unsigned readbuf[32];  

    bwr.write_size = 0;  
    bwr.write_consumed = 0;  
    bwr.write_buffer = 0;  
      
    readbuf[0] = BC_ENTER_LOOPER;  
    binder_write(bs, readbuf, sizeof(unsigned));  

    for (;;) {  
bwr.read_size = sizeof(readbuf);  
bwr.read_consumed = 0;  
bwr.read_buffer = (unsigned) readbuf;  

res = ioctl(bs->fd, BINDER_WRITE_READ, &bwr);  

if (res < 0) {  
    LOGE("binder_loop: ioctl failed (%s)\n", strerror(errno));  
    break;  
}  

res = binder_parse(bs, 0, readbuf, bwr.read_consumed, func);  
if (res == 0) {  
    LOGE("binder_loop: unexpected reply?!\n");  
    break;  
}  
if (res < 0) {  
    LOGE("binder_loop: io error %d %s\n", res, strerror(errno));  
    break;  
}  
    }  
}  
```
首先是通过binder_write函数执行BC_ENTER_LOOPER命令告诉Binder驱动程序， Service Manager要进入循环了。

这里又要介绍一下设备文件/dev/binder文件操作函数ioctl的操作码BINDER_WRITE_READ了，首先看定义：

```c
#define BINDER_WRITE_READ           _IOWR('b', 1, struct binder_write_read)  

这个io操作码有一个参数，形式为struct binder_write_read：

​```c
struct binder_write_read {  
    signed long write_size; /* bytes to write */  
    signed long write_consumed; /* bytes consumed by driver */  
    unsigned long   write_buffer;  
    signed long read_size;  /* bytes to read */  
    signed long read_consumed;  /* bytes consumed by driver */  
    unsigned long   read_buffer;  
};  
```
这里顺便说一下，用户空间程序和Binder驱动程序交互大多数都是通过BINDER_WRITE_READ命令的，write_bufffer和read_buffer所指向的数据结构还指定了具体要执行的操作，write_bufffer和read_buffer所指向的结构体是struct binder_transaction_data：

```c
struct binder_transaction_data {  
    /* The first two are only used for bcTRANSACTION and brTRANSACTION, 
     * identifying the target and contents of the transaction. 
     */  
    union {  
size_t  handle; /* target descriptor of command transaction */  
void    *ptr;   /* target descriptor of return transaction */  
    } target;  
    void        *cookie;    /* target object cookie */  
    unsigned int    code;       /* transaction command */  

    /* General information about the transaction. */  
    unsigned int    flags;  
    pid_t       sender_pid;  
    uid_t       sender_euid;  
    size_t      data_size;  /* number of bytes of data */  
    size_t      offsets_size;   /* number of bytes of offsets */  

    /* If this transaction is inline, the data immediately 
     * follows here; otherwise, it ends with a pointer to 
     * the data buffer. 
     */  
    union {  
struct {  
    /* transaction data */  
    const void  *buffer;  
    /* offsets from buffer to flat_binder_object structs */  
    const void  *offsets;  
} ptr;  
uint8_t buf[8];  
    } data;  
};  
```
有一个联合体target，当这个BINDER_WRITE_READ命令的目标对象是本地Binder实体时，就使用ptr来表示这个对象在本进程中的地址，否则就使用handle来表示这个Binder实体的引用。只有目标对象是Binder实体时，cookie成员变量才有意义，表示一些附加数据，由Binder实体来解释这个个附加数据。code表示要对目标对象请求的命令代码，有很多请求代码，这里就不列举了，在这个场景中，就是BC_ENTER_LOOPER了，用来告诉Binder驱动程序， Service Manager要进入循环了。其余的请求命令代码可以参考kernel/common/drivers/staging/android/binder.h文件中定义的两个枚举类型BinderDriverReturnProtocol和BinderDriverCommandProtocol。

flags成员变量表示事务标志：

```c
enum transaction_flags {  
    TF_ONE_WAY  = 0x01, /* this is a one-way call: async, no return */  
    TF_ROOT_OBJECT  = 0x04, /* contents are the component's root object */  
    TF_STATUS_CODE  = 0x08, /* contents are a 32-bit status code */  
    TF_ACCEPT_FDS   = 0x10, /* allow replies with file descriptors */  
};  
```
每一个标志位所表示的意义看注释就行了，遇到时再具体分析。

sender_pid和sender_euid表示发送者进程的pid和euid。

data_size表示data.buffer缓冲区的大小，offsets_size表示data.offsets缓冲区的大小。这里需要解释一下data成员变量，命令的真正要传输的数据就保存在data.buffer缓冲区中，前面的一成员变量都是一些用来描述数据的特征的。data.buffer所表示的缓冲区数据分为两类，一类是普通数据，Binder驱动程序不关心，一类是Binder实体或者Binder引用，这需要Binder驱动程序介入处理。为什么呢？想想，如果一个进程A传递了一个Binder实体或Binder引用给进程B，那么，Binder驱动程序就需要介入维护这个Binder实体或者引用的引用计数，防止B进程还在使用这个Binder实体时，A却销毁这个实体，这样的话，B进程就会crash了。所以在传输数据时，如果数据中含有Binder实体和Binder引和，就需要告诉Binder驱动程序它们的具体位置，以便Binder驱动程序能够去维护它们。data.offsets的作用就在这里了，它指定在data.buffer缓冲区中，所有Binder实体或者引用的偏移位置。每一个Binder实体或者引用，通过struct flat_binder_object 来表示：

```c
/* 
* This is the flattened representation of a Binder object for transfer 
* between processes.  The 'offsets' supplied as part of a binder transaction 
* contains offsets into the data where these structures occur.  The Binder 
* driver takes care of re-writing the structure type and data as it moves 
* between processes. 
*/  
struct flat_binder_object {  
    /* 8 bytes for large_flat_header. */  
    unsigned long       type;  
    unsigned long       flags;  

    /* 8 bytes of data. */  
    union {  
void        *binder;    /* local object */  
signed long handle;     /* remote object */  
    };  

    /* extra data associated with local object */  
    void            *cookie;  
};  

type表示Binder对象的类型，它取值如下所示：

​```c
enum {  
    BINDER_TYPE_BINDER  = B_PACK_CHARS('s', 'b', '*', B_TYPE_LARGE),  
    BINDER_TYPE_WEAK_BINDER = B_PACK_CHARS('w', 'b', '*', B_TYPE_LARGE),  
    BINDER_TYPE_HANDLE  = B_PACK_CHARS('s', 'h', '*', B_TYPE_LARGE),  
    BINDER_TYPE_WEAK_HANDLE = B_PACK_CHARS('w', 'h', '*', B_TYPE_LARGE),  
    BINDER_TYPE_FD      = B_PACK_CHARS('f', 'd', '*', B_TYPE_LARGE),  
};  
```
flags表示Binder对象的标志，该域只对第一次传递Binder实体时有效，因为此刻驱动需要在内核中创建相应的实体节点，有些参数需要从该域取出。

type和flags的具体意义可以参考[Android Binder设计与实现](http://disanji.net/2011/02/28/android-bnder-design/)一文。

最后，binder表示这是一个Binder实体，handle表示这是一个Binder引用，当这是一个Binder实体时，cookie才有意义，表示附加数据，由进程自己解释。

数据结构分析完了，回到binder_loop函数中，首先是执行BC_ENTER_LOOPER命令：

```c
readbuf[0] = BC_ENTER_LOOPER;  
binder_write(bs, readbuf, sizeof(unsigned));  

进入到binder_write函数中：

​```c
int binder_write(struct binder_state *bs, void *data, unsigned len)  
{  
    struct binder_write_read bwr;  
    int res;  
    bwr.write_size = len;  
    bwr.write_consumed = 0;  
    bwr.write_buffer = (unsigned) data;  
    bwr.read_size = 0;  
    bwr.read_consumed = 0;  
    bwr.read_buffer = 0;  
    res = ioctl(bs->fd, BINDER_WRITE_READ, &bwr);  
    if (res < 0) {  
fprintf(stderr,"binder_write: ioctl failed (%s)\n",  
        strerror(errno));  
    }  
    return res;  
}  
```
注意这里的binder_write_read变量bwr，write_size大小为4，表示write_buffer缓冲区大小为4，它的内容是一个BC_ENTER_LOOPER命令协议号，read_buffer为空。接着又是调用ioctl函数进入到Binder驱动程序的binder_ioctl函数，这里我们也只是关注BC_ENTER_LOOPER相关的逻辑：

```c
static long binder_ioctl(struct file *filp, unsigned int cmd, unsigned long arg)  
{  
    int ret;  
    struct binder_proc *proc = filp->private_data;  
    struct binder_thread *thread;  
    unsigned int size = _IOC_SIZE(cmd);  
    void __user *ubuf = (void __user *)arg;  

    /*printk(KERN_INFO "binder_ioctl: %d:%d %x %lx\n", proc->pid, current->pid, cmd, arg);*/  

    ret = wait_event_interruptible(binder_user_error_wait, binder_stop_on_user_error < 2);  
    if (ret)  
return ret;  

    mutex_lock(&binder_lock);  
    thread = binder_get_thread(proc);  
    if (thread == NULL) {  
ret = -ENOMEM;  
goto err;  
    }  

    switch (cmd) {  
    case BINDER_WRITE_READ: {  
struct binder_write_read bwr;  
if (size != sizeof(struct binder_write_read)) {  
    ret = -EINVAL;  
    goto err;  
}  
if (copy_from_user(&bwr, ubuf, sizeof(bwr))) {  
    ret = -EFAULT;  
    goto err;  
}  
if (binder_debug_mask & BINDER_DEBUG_READ_WRITE)  
    printk(KERN_INFO "binder: %d:%d write %ld at %08lx, read %ld at %08lx\n",  
    proc->pid, thread->pid, bwr.write_size, bwr.write_buffer, bwr.read_size, bwr.read_buffer);  
if (bwr.write_size > 0) {  
    ret = binder_thread_write(proc, thread, (void __user *)bwr.write_buffer, bwr.write_size, &bwr.write_consumed);  
    if (ret < 0) {  
        bwr.read_consumed = 0;  
        if (copy_to_user(ubuf, &bwr, sizeof(bwr)))  
            ret = -EFAULT;  
        goto err;  
    }  
}  
if (bwr.read_size > 0) {  
    ret = binder_thread_read(proc, thread, (void __user *)bwr.read_buffer, bwr.read_size, &bwr.read_consumed, filp->f_flags & O_NONBLOCK);  
    if (!list_empty(&proc->todo))  
        wake_up_interruptible(&proc->wait);  
    if (ret < 0) {  
        if (copy_to_user(ubuf, &bwr, sizeof(bwr)))  
            ret = -EFAULT;  
        goto err;  
    }  
}  
if (binder_debug_mask & BINDER_DEBUG_READ_WRITE)  
    printk(KERN_INFO "binder: %d:%d wrote %ld of %ld, read return %ld of %ld\n",  
    proc->pid, thread->pid, bwr.write_consumed, bwr.write_size, bwr.read_consumed, bwr.read_size);  
if (copy_to_user(ubuf, &bwr, sizeof(bwr))) {  
    ret = -EFAULT;  
    goto err;  
}  
break;  
                    }  
    ......  
    default:  
ret = -EINVAL;  
goto err;  
    }  
    ret = 0;  
err:  
    if (thread)  
thread->looper &= ~BINDER_LOOPER_STATE_NEED_RETURN;  
    mutex_unlock(&binder_lock);  
    wait_event_interruptible(binder_user_error_wait, binder_stop_on_user_error < 2);  
    if (ret && ret != -ERESTARTSYS)  
printk(KERN_INFO "binder: %d:%d ioctl %x %lx returned %d\n", proc->pid, current->pid, cmd, arg, ret);  
    return ret;  
}  
```
函数前面的代码就不解释了，同前面调用binder_become_context_manager是一样的，只不过这里调用binder_get_thread函数获取binder_thread，就能从proc中直接找到了，不需要创建一个新的。

首先是通过copy_from_user(&bwr, ubuf, sizeof(bwr))语句把用户传递进来的参数转换成struct binder_write_read结构体，并保存在本地变量bwr中，这里可以看出bwr.write_size等于4，于是进入binder_thread_write函数，这里我们只关注BC_ENTER_LOOPER相关的代码：

```c
int  
binder_thread_write(struct binder_proc *proc, struct binder_thread *thread,  
            void __user *buffer, int size, signed long *consumed)  
{  
    uint32_t cmd;  
    void __user *ptr = buffer + *consumed;  
    void __user *end = buffer + size;  

    while (ptr < end && thread->return_error == BR_OK) {  
if (get_user(cmd, (uint32_t __user *)ptr))  
    return -EFAULT;  
ptr += sizeof(uint32_t);  
if (_IOC_NR(cmd) < ARRAY_SIZE(binder_stats.bc)) {  
    binder_stats.bc[_IOC_NR(cmd)]++;  
    proc->stats.bc[_IOC_NR(cmd)]++;  
    thread->stats.bc[_IOC_NR(cmd)]++;  
}  
switch (cmd) {  
......  
case BC_ENTER_LOOPER:  
    if (binder_debug_mask & BINDER_DEBUG_THREADS)  
        printk(KERN_INFO "binder: %d:%d BC_ENTER_LOOPER\n",  
        proc->pid, thread->pid);  
    if (thread->looper & BINDER_LOOPER_STATE_REGISTERED) {  
        thread->looper |= BINDER_LOOPER_STATE_INVALID;  
        binder_user_error("binder: %d:%d ERROR:"  
            " BC_ENTER_LOOPER called after "  
            "BC_REGISTER_LOOPER\n",  
            proc->pid, thread->pid);  
    }  
    thread->looper |= BINDER_LOOPER_STATE_ENTERED;  
    break;  
......  
default:  
    printk(KERN_ERR "binder: %d:%d unknown command %d\n", proc->pid, thread->pid, cmd);  
    return -EINVAL;  
}  
*consumed = ptr - buffer;  
    }  
    return 0;  
}  
```
回忆前面执行binder_become_context_manager到binder_ioctl时，调用binder_get_thread函数创建的thread->looper值为0，所以这里执行完BC_ENTER_LOOPER时，thread->looper值就变为BINDER_LOOPER_STATE_ENTERED了，表明当前线程进入循环状态了。

回到binder_ioctl函数，由于bwr.read_size == 0，binder_thread_read函数就不会被执行了，这样，binder_ioctl的任务就完成了。

回到binder_loop函数，进入for循环：

```c
for (;;) {  
    bwr.read_size = sizeof(readbuf);  
    bwr.read_consumed = 0;  
    bwr.read_buffer = (unsigned) readbuf;  

    res = ioctl(bs->fd, BINDER_WRITE_READ, &bwr);  

    if (res < 0) {  
LOGE("binder_loop: ioctl failed (%s)\n", strerror(errno));  
break;  
    }  

    res = binder_parse(bs, 0, readbuf, bwr.read_consumed, func);  
    if (res == 0) {  
LOGE("binder_loop: unexpected reply?!\n");  
break;  
    }  
    if (res < 0) {  
LOGE("binder_loop: io error %d %s\n", res, strerror(errno));  
break;  
    }  
}  
```
又是执行一个ioctl命令，注意，这里的bwr参数各个成员的值：

```c
bwr.write_size = 0;  
bwr.write_consumed = 0;  
bwr.write_buffer = 0;  
readbuf[0] = BC_ENTER_LOOPER;  
bwr.read_size = sizeof(readbuf);  
bwr.read_consumed = 0;  
bwr.read_buffer = (unsigned) readbuf;  
```
再次进入到binder_ioctl函数：

```c
static long binder_ioctl(struct file *filp, unsigned int cmd, unsigned long arg)  
{  
    int ret;  
    struct binder_proc *proc = filp->private_data;  
    struct binder_thread *thread;  
    unsigned int size = _IOC_SIZE(cmd);  
    void __user *ubuf = (void __user *)arg;  

    /*printk(KERN_INFO "binder_ioctl: %d:%d %x %lx\n", proc->pid, current->pid, cmd, arg);*/  

    ret = wait_event_interruptible(binder_user_error_wait, binder_stop_on_user_error < 2);  
    if (ret)  
return ret;  

    mutex_lock(&binder_lock);  
    thread = binder_get_thread(proc);  
    if (thread == NULL) {  
ret = -ENOMEM;  
goto err;  
    }  

    switch (cmd) {  
    case BINDER_WRITE_READ: {  
struct binder_write_read bwr;  
if (size != sizeof(struct binder_write_read)) {  
    ret = -EINVAL;  
    goto err;  
}  
if (copy_from_user(&bwr, ubuf, sizeof(bwr))) {  
    ret = -EFAULT;  
    goto err;  
}  
if (binder_debug_mask & BINDER_DEBUG_READ_WRITE)  
    printk(KERN_INFO "binder: %d:%d write %ld at %08lx, read %ld at %08lx\n",  
    proc->pid, thread->pid, bwr.write_size, bwr.write_buffer, bwr.read_size, bwr.read_buffer);  
if (bwr.write_size > 0) {  
    ret = binder_thread_write(proc, thread, (void __user *)bwr.write_buffer, bwr.write_size, &bwr.write_consumed);  
    if (ret < 0) {  
        bwr.read_consumed = 0;  
        if (copy_to_user(ubuf, &bwr, sizeof(bwr)))  
            ret = -EFAULT;  
        goto err;  
    }  
}  
if (bwr.read_size > 0) {  
    ret = binder_thread_read(proc, thread, (void __user *)bwr.read_buffer, bwr.read_size, &bwr.read_consumed, filp->f_flags & O_NONBLOCK);  
    if (!list_empty(&proc->todo))  
        wake_up_interruptible(&proc->wait);  
    if (ret < 0) {  
        if (copy_to_user(ubuf, &bwr, sizeof(bwr)))  
            ret = -EFAULT;  
        goto err;  
    }  
}  
if (binder_debug_mask & BINDER_DEBUG_READ_WRITE)  
    printk(KERN_INFO "binder: %d:%d wrote %ld of %ld, read return %ld of %ld\n",  
    proc->pid, thread->pid, bwr.write_consumed, bwr.write_size, bwr.read_consumed, bwr.read_size);  
if (copy_to_user(ubuf, &bwr, sizeof(bwr))) {  
    ret = -EFAULT;  
    goto err;  
}  
break;  
                    }  
    ......  
    default:  
ret = -EINVAL;  
goto err;  
    }  
    ret = 0;  
err:  
    if (thread)  
thread->looper &= ~BINDER_LOOPER_STATE_NEED_RETURN;  
    mutex_unlock(&binder_lock);  
    wait_event_interruptible(binder_user_error_wait, binder_stop_on_user_error < 2);  
    if (ret && ret != -ERESTARTSYS)  
printk(KERN_INFO "binder: %d:%d ioctl %x %lx returned %d\n", proc->pid, current->pid, cmd, arg, ret);  
    return ret;  
}  
```
这次，bwr.write_size等于0，于是不会执行binder_thread_write函数，bwr.read_size等于32，于是进入到binder_thread_read函数：

```c
static int  
binder_thread_read(struct binder_proc *proc, struct binder_thread *thread,  
           void  __user *buffer, int size, signed long *consumed, int non_block)  
{  
    void __user *ptr = buffer + *consumed;  
    void __user *end = buffer + size;  

    int ret = 0;  
    int wait_for_proc_work;  

    if (*consumed == 0) {  
if (put_user(BR_NOOP, (uint32_t __user *)ptr))  
    return -EFAULT;  
ptr += sizeof(uint32_t);  
    }  

retry:  
    wait_for_proc_work = thread->transaction_stack == NULL && list_empty(&thread->todo);  

    if (thread->return_error != BR_OK && ptr < end) {  
if (thread->return_error2 != BR_OK) {  
    if (put_user(thread->return_error2, (uint32_t __user *)ptr))  
        return -EFAULT;  
    ptr += sizeof(uint32_t);  
    if (ptr == end)  
        goto done;  
    thread->return_error2 = BR_OK;  
}  
if (put_user(thread->return_error, (uint32_t __user *)ptr))  
    return -EFAULT;  
ptr += sizeof(uint32_t);  
thread->return_error = BR_OK;  
goto done;  
    }  


    thread->looper |= BINDER_LOOPER_STATE_WAITING;  
    if (wait_for_proc_work)  
proc->ready_threads++;  
    mutex_unlock(&binder_lock);  
    if (wait_for_proc_work) {  
if (!(thread->looper & (BINDER_LOOPER_STATE_REGISTERED |  
    BINDER_LOOPER_STATE_ENTERED))) {  
        binder_user_error("binder: %d:%d ERROR: Thread waiting "  
            "for process work before calling BC_REGISTER_"  
            "LOOPER or BC_ENTER_LOOPER (state %x)\n",  
            proc->pid, thread->pid, thread->looper);  
        wait_event_interruptible(binder_user_error_wait, binder_stop_on_user_error < 2);  
}  
binder_set_nice(proc->default_priority);  
if (non_block) {  
    if (!binder_has_proc_work(proc, thread))  
        ret = -EAGAIN;  
} else  
    ret = wait_event_interruptible_exclusive(proc->wait, binder_has_proc_work(proc, thread));  
    } else {  
if (non_block) {  
    if (!binder_has_thread_work(thread))  
        ret = -EAGAIN;  
} else  
    ret = wait_event_interruptible(thread->wait, binder_has_thread_work(thread));  
    }  
.......  
}  
```
传入的参数*consumed == 0，于是写入一个值BR_NOOP到参数ptr指向的缓冲区中去，即用户传进来的bwr.read_buffer缓冲区。这时候，thread->transaction_stack == NULL，并且thread->todo列表也是空的，这表示当前线程没有事务需要处理，于是wait_for_proc_work为true，表示要去查看proc是否有未处理的事务。当前thread->return_error == BR_OK，这是前面创建binder_thread时初始化设置的。于是继续往下执行，设置thread的状态为BINDER_LOOPER_STATE_WAITING，表示线程处于等待状态。调用binder_set_nice函数设置当前线程的优先级别为proc->default_priority，这是因为thread要去处理属于proc的事务，因此要将此thread的优先级别设置和proc一样。在这个场景中，proc也没有事务处理，即binder_has_proc_work(proc, thread)为false。如果文件打开模式为非阻塞模式，即non_block为true，那么函数就直接返回-EAGAIN，要求用户重新执行ioctl；否则的话，就通过当前线程就通过wait_event_interruptible_exclusive函数进入休眠状态，等待请求到来再唤醒了。

至此，我们就从源代码一步一步地分析完Service Manager是如何成为Android进程间通信（IPC）机制Binder守护进程的了。总结一下，Service Manager是成为Android进程间通信（IPC）机制Binder守护进程的过程是这样的：

打开/dev/binder文件：open("/dev/binder", O_RDWR);

建立128K内存映射：mmap(NULL, mapsize, PROT_READ, MAP_PRIVATE, bs->fd, 0);

通知Binder驱动程序它是守护进程：binder_become_context_manager(bs);

进入循环等待请求的到来：binder_loop(bs, svcmgr_handler);

在这个过程中，在Binder驱动程序中建立了一个struct binder_proc结构、一个struct  binder_thread结构和一个struct binder_node结构，这样，Service Manager就在Android系统的进程间通信机制Binder担负起守护进程的职责了。
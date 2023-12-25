# LAB8

## 练习1: 完成读文件操作的实现（需要编码）

### 首先了解打开文件的处理流程，然后参考本实验后续的文件读写操作的过程分析，填写在 kern/fs/sfs/sfs_inode.c中 的sfs_io_nolock()函数，实现读文件中数据的代码。

如果进行文件的读取或写入操作，最终都会执行到`sfs_io_nolock()`函数中去，在其中完成对设备上基础块数据的读取或写入。

进行读取或写入前要先将数据与基础块对齐，以便于使用`sfs_block_op()`函数来操作基础块，提高读取或写入的效率。 

但是将数据对齐后,待操作数据的头部可能是之前一个基础块的末尾,尾部可能是之后一个基础块的起始

我们需要分别专门对这两个位置的基础块进行读取或写入，因为这两个位置的基础块所涉及到的数据都是一个完整块的一部分。而中间的数据由于已经对齐好基础块了，所以可以直接调用`sfs_block_op()`函数来读取或写入数据。

以下是相关操作的实现:
```
//LAB8:EXERCISE1 YOUR CODE HINT: call sfs_bmap_load_nolock, sfs_rbuf, sfs_rblock,etc. read different kind of blocks in file
	/*
	 * (1) If offset isn't aligned with the first block, Rd/Wr some content from offset to the end of the first block
	 *       NOTICE: useful function: sfs_bmap_load_nolock, sfs_buf_op
	 *               Rd/Wr size = (nblks != 0) ? (SFS_BLKSIZE - blkoff) : (endpos - offset)
	 * (2) Rd/Wr aligned blocks 
	 *       NOTICE: useful function: sfs_bmap_load_nolock, sfs_block_op
     * (3) If end position isn't aligned with the last block, Rd/Wr some content from begin to the (endpos % SFS_BLKSIZE) of the last block
	 *       NOTICE: useful function: sfs_bmap_load_nolock, sfs_buf_op	
	*/

if ((blkoff = offset % SFS_BLKSIZE) != 0) {
    // 计算当前块内偏移量，如果不为0，则说明当前位置不在块的起始位置
    size = (nblks != 0) ? (SFS_BLKSIZE - blkoff) : (endpos - offset);
    // 根据剩余块数和偏移量计算需要读取的大小
    if ((ret = sfs_bmap_load_nolock(sfs, sin, blkno, &ino)) != 0) {
        // 调用sfs_bmap_load_nolock函数获取块号对应的磁盘块号
        goto out;
    }
    if ((ret = sfs_buf_op(sfs, buf, size, ino, blkoff)) != 0) {
        // 调用sfs_buf_op函数将数据从磁盘块读入缓冲区
        goto out;
    }

    alen += size;
    buf += size;

    if (nblks == 0) {
        // 如果没有剩余块了，则直接跳出循环
        goto out;
    }

    blkno++;
    nblks--;
}

if (nblks > 0) {
    // 如果还有剩余块需要读取
    if ((ret = sfs_bmap_load_nolock(sfs, sin, blkno, &ino)) != 0) {
        // 调用sfs_bmap_load_nolock函数获取下一个块号对应的磁盘块号
        goto out;
    }
    if ((ret = sfs_block_op(sfs, buf, ino, nblks)) != 0) {
        // 调用sfs_block_op函数将数据从磁盘连续读入缓冲区
        goto out;
    }

    alen += nblks * SFS_BLKSIZE;
    buf += nblks * SFS_BLKSIZE;
    blkno += nblks;
    nblks -= nblks;
}

if ((size = endpos % SFS_BLKSIZE) != 0) {
    // 计算读取最后一个块的大小
    if ((ret = sfs_bmap_load_nolock(sfs, sin, blkno, &ino)) != 0) {
        // 调用sfs_bmap_load_nolock函数获取最后一个块号对应的磁盘块号
        goto out;
    }
    if ((ret = sfs_buf_op(sfs, buf, size, ino, 0)) != 0) {
        // 调用sfs_buf_op函数将最后一个块的数据读入缓冲区
        goto out;
    }
    alen += size;
}
```

## 练习2: 完成基于文件系统的执行程序机制的实现（需要编码）

### 改写proc.c中的load_icode函数和其他相关函数，实现基于文件系统的执行程序机制。执行：make qemu。如果能看看到sh用户程序的执行界面，则基本成功了。如果在sh用户界面上可以执行”ls”,”hello”等其他放置在sfs文件系统中的其他执行程序，则可以认为本实验基本成功。

```
// load_icode - called by sys_exec-->do_execve

static int
load_icode(int fd, int argc, char *kargv) {  // 加载可执行程序并创建进程
/ LAB8:EXERCISE2 YOUR CODE  HINT:how to load the file with handler fd in to process's memory? how to setup argc/argv?
* MACROs or Functions:
*  mm_create        - create a mm
*  setup_pgdir      - setup pgdir in mm
*  load_icode_read  - read raw data content of program file
*  mm_map           - build new vma
*  pgdir_alloc_page - allocate new memory for TEXT/DATA/BSS/stack parts
*  lcr3             - update Page Directory Addr Register -- CR3
*/

assert(argc >= 0 && argc <= EXEC_MAX_ARG_NUM);  // 检查参数数量的合法性
    if (current->mm != NULL) {  // 如果当前进程的内存管理结构不为空
        panic("load_icode: current->mm must be empty.\n");  // 报错
    }

    int ret = -E_NO_MEM;  // 定义返回值变量并初始化为内存不足的错误码
    struct mm_struct *mm;  // 定义内存管理结构指针
    if ((mm = mm_create()) == NULL) {  // 创建内存管理结构
        goto bad_mm;  // 处理错误
    }
    if (setup_pgdir(mm) != 0) {  // 设置页目录
        goto bad_pgdir_cleanup_mm;  // 处理错误
    }

    struct Page *page;  // 定义页面结构指针

    struct elfhdr __elf, *elf = &__elf;  // 定义 ELF 文件头结构
    if ((ret = load_icode_read(fd, elf, sizeof(struct elfhdr), 0)) != 0) {  // 读取 ELF 文件头
        goto bad_elf_cleanup_pgdir;  // 处理错误
    }

    if (elf->e_magic != ELF_MAGIC) {  // 检查 ELF 文件的魔数是否有效
        ret = -E_INVAL_ELF;  // 设置错误码为无效的 ELF 文件
        goto bad_elf_cleanup_pgdir;  // 处理错误
    }
    struct proghdr __ph, *ph = &__ph;  // 定义程序段头结构
    uint32_t vm_flags, perm, phnum;  // 定义虚拟内存标志、权限和段数

    for (phnum = 0; phnum < elf->e_phnum; phnum ++) {  // 遍历所有程序段
        off_t phoff = elf->e_phoff + sizeof(struct proghdr) * phnum;  // 计算程序段头的偏移量
        if ((ret = load_icode_read(fd, ph, sizeof(struct proghdr), phoff)) != 0) {  // 读取程序段头
            goto bad_cleanup_mmap;  // 处理错误
        }
        // 接下来的代码主要是根据程序段的信息进行虚拟内存映射和页面分配等操作，这里省略注释
        // ...

    }

    sysfile_close(fd);  // 关闭文件

    // 设置用户栈空间
    vm_flags = VM_READ | VM_WRITE | VM_STACK;
    if ((ret = mm_map(mm, USTACKTOP - USTACKSIZE, USTACKSIZE, vm_flags, NULL)) != 0) {
        goto bad_cleanup_mmap;  // 处理错误
    }
    // 分配用户栈空间的页面
    assert(pgdir_alloc_page(mm->pgdir, USTACKTOP-PGSIZE , PTE_USER) != NULL);
    assert(pgdir_alloc_page(mm->pgdir, USTACKTOP-2*PGSIZE , PTE_USER) != NULL);
    assert(pgdir_alloc_page(mm->pgdir, USTACKTOP-3*PGSIZE , PTE_USER) != NULL);
    assert(pgdir_alloc_page(mm->pgdir, USTACKTOP-4*PGSIZE , PTE_USER) != NULL);

    mm_count_inc(mm);  // 增加内存管理结构的引用计数
    current->mm = mm;  // 当前进程的内存管理结构指向新创建的内存管理结构
    current->cr3 = PADDR(mm->pgdir);  // 更新当前进程的页目录物理地址
    lcr3(PADDR(mm->pgdir));  // 刷新 TLB 缓存

    // 设置 argc, argv 参数
    // ...

    struct trapframe *tf = current->tf;  // 定义中断帧指针
    // ...
    
out:
    return ret;  // 返回结果
bad_cleanup_mmap:
    exit_mmap(mm);  // 退出内存映射
bad_elf_cleanup_pgdir:
    put_pgdir(mm);  // 释放页目录
bad_pgdir_cleanup_mm:
    mm_destroy(mm);  // 销毁内存管理结构
bad_mm:
    goto out;  // 跳转到结束
}
```

## 扩展练习 Challenge1：完成基于“UNIX的PIPE机制”的设计方案

### 如果要在ucore里加入UNIX的管道（Pipe)机制，至少需要定义哪些数据结构和接口？（接口给出语义即可，不必具体实现。数据结构的设计应当给出一个(或多个）具体的C语言struct定义。在网络上查找相关的Linux资料和实现，请在实验报告中给出设计实现”UNIX的PIPE机制“的概要设方案，你的设计应当体现出对可能出现的同步互斥问题的处理。）

**·数据结构设计**

定义结构体pipe如下：

```
#define PIPE_BUFFER_SIZE 1024

typedef struct {
    char buffer[PIPE_BUFFER_SIZE];
    int read_pos;
    int write_pos;
    semaphore_t mutex;
    semaphore_t items;
    semaphore_t spaces;
} pipe_t;
```

其中字段含义如下：

* char buffer[PIPE_BUFFER_SIZE]: 用于存储数据的缓冲区
* int read_pos: 缓冲区中下一个读取数据的位置
* int write_pos: 缓冲区中下一个写入数据的位置
* struct semaphore mutex: 用于同步对管道的访问
* struct semaphore items: 记录缓冲区中的数据项数量，用于同步和互斥
* struct semaphore spaces: 记录缓冲区中的空闲空间数量，用于同步和互斥

**·其他辅助结构**

可能还需要一些辅助结构来管理管道，比如用于跟踪所有活动管道的结构

**·接口设计**

**··pipe_create**

创建一个新的管道

初始化管道的数据结构，包括缓冲区、同步互斥信号量等

**··pipe_read**

从管道中读取数据，等待管道中有数据可读（使用items信号量），然后从缓冲区中读取数据，更新读取位置，并释放一个空间（使用spaces信号量）。

**··pipe_write**

向管道写入数据，等待管道中有空间可写（使用spaces信号量），然后向缓冲区写入数据，更新写入位置，并增加一个数据项（使用items信号量）。

**··pipe_close**

关闭管道，释放与管道相关的资源

**·同步互斥问题处理**

使用信号量来解决同步互斥问题，mutex 信号量用于确保对管道结构的互斥访问，而 items 和spaces信号量用于管道的同步。items确保读操作在有数据可读时进行，spaces确保写操作在有空间可写时进行。

## 扩展练习 Challenge2：完成基于“UNIX的软连接和硬连接机制”的设计方案

### 如果要在ucore里加入UNIX的软连接和硬连接机制，至少需要定义哪些数据结构和接口？（接口给出语义即可，不必具体实现。数据结构的设计应当给出一个(或多个）具体的C语言struct定义。在网络上查找相关的Linux资料和实现，请在实验报告中给出设计实现”UNIX的软连接和硬连接机制“的概要设方案，你的设计应当体现出对可能出现的同步互斥问题的处理。）

**·数据结构设计**

增加结构体inode字段如下：

```
typedef struct inode {
    ...
    unsigned int link_count;
    char* symlink_path;
    ...
} inode_t;
```

新增加的字段含义如下：

* unsigned int link_count: 表示硬连接的数量
* char* symlink_path: 如果是软连接，存储指向的路径

增加结构体file字段如下：

```
typedef struct file {
    ...
    inode_t* inode;
    ...
} file_t;
```

**·接口设计**

··link

创建硬连接，为指定的 inode 创建一个新的目录项（directory entry），使其link_count增加

··unlink

删除硬连接，移除一个inode的目录项，使其link_count减少。当link_count为0时，释放inode

··symlink

创建软连接，创建一个新的 inode，其类型为软连接，并存储目标路径

··readlink

读取软连接，返回软连接指向的路径。

**·同步互斥问题处理**

当更新 inode（例如，改变 link_count 或 symlink_path）时，需要使用锁（如互斥锁）来确保操作的原子性和一致性。

在进行文件系统的读写操作时，应确保对相关数据结构的互斥访问，以避免数据竞争和一致性问题。

# eBPF实战教程三｜数据库磁盘IO最精准的量化方法(含源码)

## 前言

感谢各位小伙伴对我们eBPF专题系列文章的持续关注与支持！在上一篇eBPF技术文章[《eBPF实战教程二｜数据库网络流量最精准的量化方法(含源码)》](https://github.com/DBdoctor-DAS/DBdoctor/blob/main/articles/EBPF02.md)中，我们探讨了如何手写一个Kprobe函数来观测MySQL的网络流量。许多热心的小伙伴纷纷私信我们，希望我们可以分享更多eBPF在数据库领域的应用场景。

为了满足大家的期待，我们特别推出该系列第四篇纯技术分享文章——如何手码一个Kprobe函数来分析MySQL数据库表维度的磁盘IO。我们希望通过这篇文章，为大家提供更深入的eBPF技术分析和实用的操作指南。同时，我们也将持续更新eBPF实战系列文章，敬请关注我们的公众号，获取更多精彩内容！

## 文件系统读写函数的选取

MySQL从5.6版本开始默认是独立表空间，Innodb每个表或者索引对应一个文件。那么我们可以基于Kprobe来探测表文件的读写。首先我们需要理解MySQL是如何进行读写，下图是数据库表文件IO的读写过程：

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZku16QCBs4xZsDQnVSic4PPPtXrrIdZwJ1FZ9IeJgTbZG7D3v8YfTHdbnURklrdvACwvcic6H9eiarhQ/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

Linux中的VFS（Virtual File System，虚拟文件系统）负责管理系统中所有的文件和文件系统。VFS提供了一个统一的接口，使得不同类型的文件系统可以在Linux中无缝协作。

读写操作是一个常见的文件系统操作，它用于向文件中写入数据。当应用程序需要向文件中读写入数据时，它会向VFS发出写请求。VFS负责将这个请求传递给相应的文件系统内核模块，然后由文件系统模块负责实际的读写操作。

### 1）表维度的IO写入
vfs_write()函数是负责处理写操作的主要函数之一。当应用程序调用write()系统调用时，实际上是调用了vfs_write()函数，该函数负责将数据写入文件中。在调用vfs_write()函数之前，应用程序需要先打开文件，并获取到文件的文件描述符。然后，通过文件描述符就可以向vfs_write()函数传递写操作的数据和参数。

下面是Linux vfs_write()函数源码

```C
ssize_t vfs_write(struct file *file, const char __user *buf, size_t count, loff_t *pos)
{
  ssize_t ret;
  //判断文件是否可写
  if (!(file->f_mode & FMODE_WRITE))
    return -EBADF;
  if (!(file->f_mode & FMODE_CAN_WRITE))
    return -EINVAL;
  if (unlikely(!access_ok(buf, count)))
    return -EFAULT;
  //写校验
  ret = rw_verify_area(WRITE, file, pos, count);
  if (ret)
    return ret;
  if (count > MAX_RW_COUNT)
    count =  MAX_RW_COUNT;
  file_start_write(file);
  //调用文件写操作方法
  if (file->f_op->write)
    ret = file->f_op->write(file, buf, count, pos);
  else if (file->f_op->write_iter)
    ret = new_sync_write(file, buf, count, pos);
  else
    ret = -EINVAL;
  if (ret > 0) {
    fsnotify_modify(file);
    add_wchar(current, ret);
  }
  inc_syscw(current);
  file_end_write(file);
  return ret;
}
```
函数的第一个形参file即为文件对象，第三个形参count是写入大小。
- file对象中的f_inode成员变量是文件的元数据信息对象指针，该对象中的i_ino即为文件的inode号，该id可作为全局唯一的文件标识。

- file->f_path.dentry即为该文件的dentry信息，从该指针中可分别获取文件名dentry->name（数据库的表名）以及父目录名de->d_parent->d_name.name（数据库的库名）

通过vfs_write()函数，可以方便地进行写操作，而无需关心底层文件系统的具体实现细节。VFS的设计使得Linux系统更加灵活和高效，为用户提供了方便的文件系统管理功能。

因此，我们可以选取vfs_write()函数作为探测点，来统计数据库库表维度的每秒数据写入量。

### 2）表维度的IO读

vfs_read()函数是负责处理读操作的主要函数之一，函数的这些参数与返回值与 vfs_write() 函数如出一辙，就不再赘述了。

```C
ssize_t vfs_read(struct file *file, char __user *buf, size_t count, loff_t *pos)
{
  ssize_t ret;
   //判断文件是否可读
  if (!(file->f_mode & FMODE_READ))
    return -EBADF;
  if (!(file->f_mode & FMODE_CAN_READ))
    return -EINVAL;
  if (unlikely(!access_ok(buf, count)))
    return -EFAULT;
  //读校验
  ret = rw_verify_area(READ, file, pos, count);
  if (ret)
    return ret;
  if (count > MAX_RW_COUNT)
    count =  MAX_RW_COUNT;
  //调用文件写操作方法
  if (file->f_op->read)
    ret = file->f_op->read(file, buf, count, pos);
  else if (file->f_op->read_iter)
    ret = new_sync_read(file, buf, count, pos);
  else
    ret = -EINVAL;
  if (ret > 0) {
    fsnotify_access(file);
    add_rchar(current, ret);
  }
  inc_syscr(current);
  return ret;
}
```
我们也可以选取vfs_read()函数作为探测点，来统计数据库库表维度的每秒数据读取量。
## eBPF Kprobe如何探测MySQL表维度的磁盘IO读写量?

### 1）环境准备
```
准备一台 Linux 机器，安装好g++和bcc
```
### 2）基于BCC工具实现探测MySQL
要实现库表维度的磁盘IO读写统计，我们首先定义一个存储结构用来存放进程库表维度读写的总Size，基于Kprobe分别对磁盘库表维度的读写量进行累加并存储到该结构中，然后每秒去读并打印当前存储结构中累加的磁盘库表读写量，即可实现每秒的库表磁盘读写的采集。

接下来我们将基于BCC，利用Kprobe写一个eBPF程序，观测MySQL库表维度的磁盘IO的读写。
#### a）分析内核文件系统源码相关VFS磁盘读写处理的函数

```C
//数据库写vfs
ssize_t vfs_write(struct file *file, const char __user *buf, size_t count, loff_t *pos){
  ...
}
//数据库读vfs
ssize_t vfs_read(struct file *file, char __user *buf, size_t count, loff_t *pos){
  ...
}
```
#### b）导入BCC的BPF对象
```C
//这个对象可以将我们的观测代码嵌入到观测点中执行
#include <bcc/BPF.h>

#include <string>
#include <iostream>
#include <thread>
#include <time.h>
```
#### c）用c编写eBPF代码
```C

std::string strBPF = R"(
#include <linux/ptrace.h>
#include <bcc/proto.h>
#include <linux/blkdev.h>
//定义采集的指标存储结构key
struct key_t{
    u32 pid;
    u64 inode;
};

struct val_t{
    u64 reads;
    u64 writes;
    u64 rbytes;
    u64 wbytes;
    char name[32];
    char path[64];
};
//定义采集的指标存储结构value
BPF_HASH(map,struct key_t,struct val_t,10240);
BPF_HASH(flag,u32,u32,2);

static inline bool isWork(){
    u32 key = 1;
    u32* v = flag.lookup(&key);
    if(v && *v == 1) return true;
    return false;
}

static inline int do_entry(struct pt_regs *ctx, struct file *file,char __user *buf, size_t count, int is_read){
    u32 pid = bpf_get_current_pid_tgid() >> 32;
    if(FILTER_PID) return 0;
    if(!isWork()) return 0;
    struct key_t k = {};
    k.pid = pid;
    k.inode = file->f_inode->i_ino;
    struct val_t *v = map.lookup(&k);
    if(v){
        if(is_read){
            v->reads++;
            v->rbytes+=count;
        } else{
            v->writes++;
            v->wbytes+=count;
        }
    } else{
        struct val_t tv = {};
        struct dentry *de = file->f_path.dentry;
        struct qstr d_name = de->d_name;
        if (d_name.len == 0) return 0;
        bpf_probe_read_kernel(&tv.name, sizeof(tv.name), d_name.name);
        bpf_probe_read_kernel(&tv.path,sizeof(tv.path),de->d_parent->d_name.name);
        if(is_read){
            tv.reads++;
            tv.rbytes+=count;
        } else{
            tv.writes++;
            tv.wbytes+=count;
        }
        map.update(&k,&tv);
    }
    return 0;
}

int kprobe__vfs_read(struct pt_regs *ctx, struct file *file,char __user *buf, size_t count){
    return do_entry(ctx,file,buf,count,1);
}

int kprobe__vfs_write(struct pt_regs *ctx, struct file *file,char __user *buf, size_t count){
    return do_entry(ctx,file,buf,count,0);
}
)";
```
#### d）观测代码关联系统文件读写需要观测的函数
```C
//用于ebpf代码程序中的pid替换
static std::string str_replace(std::string r, const std::string& s, const std::string& n)
{
        std::string y = std::move(r);
        std::string::size_type pos = 0;
        while((pos = y.find(s)) != std::string::npos)  
            y.replace(pos, s.length(), n);
        return y;
}

struct io_key{
    u32 pid;
    u64 ino;
};

struct io_val{
    u64 reads;
    u64 writes;
    u64 rbytes;
    u64 wbytes;
    char name[32];
    char path[64];
};

//指定进程pid进行kprobe diskio统计
int main(int argc, char* argv[]) {
    int pid = std::stoull(argv[1]);
    ebpf::BPF bpf;
    std::string strFilerPid = "pid != " + std::to_string(pid);
    std::string code = str_replace(strBPF, "FILTER_PID", strFilerPid);
    auto initRes = bpf.init(code);
    if (!initRes.ok()) {
        std::cerr << "bpf init error,msg: " << initRes.msg() << std::endl;
        return 1;
    }
    std::cout << "-----------------start to sample MySQL DiskIO (table and index read_bytes/write_bytes)-------------- " << std::endl;
    /*探测vfs_read*/
    auto attachRes = bpf.attach_kprobe("vfs_read", "kprobe__vfs_read",0,BPF_PROBE_ENTRY);
    if(!attachRes.ok()) {
        std::cerr << "attach vfs_read error,msg: "<< attachRes.msg() << std::endl;
        return 1;
    }
    /*探测vfs_write*/
    attachRes = bpf.attach_kprobe("vfs_write", "kprobe__vfs_write");
    if(!attachRes.ok()) {
        std::cerr << "attach vfs_write error,msg: "<< attachRes.msg() << std::endl;
        return 1;
    }
    u32 on=1,off=0,key=1;
    auto flag = bpf.get_hash_table<uint32_t,uint32_t>("flag");
    /*每秒完成一次读取并打印*/
    while (true){
      flag.update_value(key,on);
            std::this_thread::sleep_for(std::chrono::seconds(1));
      flag.update_value(key,off);
            auto io_map = bpf.get_hash_table<io_key, io_val>("map");
            auto table = io_map.get_table_offline();
            for (auto &item : table) {
                std::cout << "pid: " << item.first.pid << " inode: " << item.first.ino << " reads: " << item.second.reads << " writes: " << item.second.writes << " rbytes: " << item.second.rbytes << " wbytes: " << item.second.wbytes << " name: " << item.second.name << " path: " << item.second.path << std::endl;
                io_map.remove_value(item.first);
            }
        }
    return 0;
}
```
#### e）效果演示
编译并执行该eBPF程序
```Bash
#编译命令
g++ -std=c++17 -o io io.cpp -lbcc -pthread
```
指定mysqld进程pid 2004756进行diskio采集：

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZku16QCBs4xZsDQnVSic4PPPlBCbkRAf4eBcCSezIltdibVXAKrKbGzRj6J901yjGKRnZG3yvaQicjcw/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

远程执行连接MySQL的命令并执行SQL

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZku16QCBs4xZsDQnVSic4PPPxibJxHBib6WibXzKP3ia0PM4FR0kuAC5pibWYWLWKG3HmW1eqvTicCEPnuibQ/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

打印观测的结果
![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZku16QCBs4xZsDQnVSic4PPPKiaiaHJiaibyN93YQy5LBEBtScicNIHT4Wfflvf6pyibJoAbicwmUicnb0dbOQ/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

从上面的演示中我们能看到，客户端和MySQL建立连接，分别执行涵盖sbtest16和sbtest15两个表SQL，每秒会打印日志，显示mysqld的进程pid、累加磁盘读写的次数和Bytes、涉及的库表。然后我们针对采集上来的数据就可以做分析了：
- 如果存在rbytes过大，说明数据库上可能存在涉及该库表的单条查询SQL扫描行过大或者很多大字段等问题，会导致大量占用磁盘IO，甚至会导致整体数据库变慢。
- 如果存在wbytes过大，说明数据库上可能存在涉及该库表的大量写入SQL，比如做批量数据删除，可能导致整体数据库变慢，可建议更新缩小范围，分多批次删除，减少对磁盘IO的占用。

## 总结

利用eBPF技术可以在内核级别捕获和分析与数据库表操作相关的磁盘IO活动。这需要我们深入理解eBPF的工作原理、Kprobe的使用、MySQL存储引擎以及文件系统交互机制等。通过文中我们的介绍，您是否对eBPF技术又有了新的认知呢？

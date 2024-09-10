# eBPF Practical Tutorial 3 | The most accurate quantization method for database disk IO (including source code)

## Preface

Thank you for your continuous attention and support to our eBPF series of articles! In the previous eBPF technical article ["eBPF Practical Tutorial 2 | The Most Accurate Quantification Method for Database Network Traffic (including source code) " (../articles/EBPF02.md), we discussed how to handwrite a Kprobe function to observe MySQL network traffic. Many enthusiastic friends have privately messaged us, hoping that we can share more application scenarios of eBPF in the database field.

In order to meet everyone's expectations, we have specially launched the fourth purely technical sharing article in this series - How to manually code a Kprobe function to analyze disk IO in the database & table dimensions of MySQL data. We hope to provide you with a more in-depth eBPF technical analysis and practical operation guide through this article. At the same time, we will continue to update the eBPF practical series of articles. Please follow our official account for more exciting content!

## Selection of file system read and write functions

MySQL defaults to independent tablespaces starting from version 5.6. Each table or index in Innodb corresponds to a file. Therefore, we can use Kprobe to detect the reading and writing of table files. First, we need to understand how MySQL reads and writes. The following figure shows the reading and writing process of database & table file IO:

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZku16QCBs4xZsDQnVSic4PPPtXrrIdZwJ1FZ9IeJgTbZG7D3v8YfTHdbnURklrdvACwvcic6H9eiarhQ/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

VFS (Virtual File System) in Linux is responsible for managing all files and file systems in the system. VFS provides a unified interface so that different types of file systems can seamlessly cooperate in Linux.

Read and write operations are a common file system operation used to write data to a file. When an application needs to read and write data to a file, it sends a write request to VFS. VFS is responsible for passing this request to the corresponding file system kernel module, which is then responsible for the actual read and write operations.

### 1） IO write of table dimension
vfs_write () function is one of the main functions responsible for handling write operations. When an application calls the write () system call, it actually calls the vfs_write () function, which is responsible for writing data to a file. Before calling the vfs_write () function, the application needs to open the file and obtain the file descriptor of the file. Then, the data and parameters of the write operation can be passed to the vfs_write () function through the file descriptor.

Here is Linux vfs_write () function source code

```C
ssize_t vfs_write(struct file *file, const char __user *buf, size_t count, loff_t *pos)
{
  ssize_t ret;
  //Check if the file is writable
  if (!(file->f_mode & FMODE_WRITE))
    return -EBADF;
  if (!(file->f_mode & FMODE_CAN_WRITE))
    return -EINVAL;
  if (unlikely(!access_ok(buf, count)))
    return -EFAULT;
  //Write check
  ret = rw_verify_area(WRITE, file, pos, count);
  if (ret)
    return ret;
  if (count > MAX_RW_COUNT)
    count =  MAX_RW_COUNT;
  file_start_write(file);
  //Call the file write operation method
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
The first parameter of the function, file, is the file object, and the third parameter, count, is the write size.

- The f_inode member variable in the file object is the metadata object pointer of the file. The i_ino in this object is the inode number of the file, which can be used as a globally unique file identifier.

- file- > f_path is the dentry information of the file, from which the pointer can respectively obtain the file name dentry- > name (database table name) and the parent directory name de- > d_parent - > d_name.name (database database name).

With the vfs_write () function, writing operations can be easily performed without worrying about the specific implementation details of the underlying file system. The design of VFS makes the Linux system more flexible and efficient, providing users with convenient file system management functions.

Therefore, we can select the vfs_write () function as a probe point to Statistical Data database & table dimension of data write per second.

### 2）IO read of table dimension

vfs_read () function is one of the main functions responsible for handling read operations. These parameters and return values of function are exactly the same as those of vfs_write () function, so they will not be repeated here.

```C
ssize_t vfs_read(struct file *file, char __user *buf, size_t count, loff_t *pos)
{
  ssize_t ret;
   //Determine if the file is readable
  if (!(file->f_mode & FMODE_READ))
    return -EBADF;
  if (!(file->f_mode & FMODE_CAN_READ))
    return -EINVAL;
  if (unlikely(!access_ok(buf, count)))
    return -EFAULT;
  //Read check
  ret = rw_verify_area(READ, file, pos, count);
  if (ret)
    return ret;
  if (count > MAX_RW_COUNT)
    count =  MAX_RW_COUNT;
  //Call the file write operation method
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
We can also select the vfs_read () function as a probe point to Statistical Data database & table dimension data reads per second.
## How to detect the disk IO read and write volume of MySQL table dimension with eBPF Kprobe?

### 1）Environment preparation
```
Prepare a Linux machine and install g ++ and bcc
```
### 2）Implement MySQL detection based on BCC tool
To achieve disk IO read and write statistics in the database & table dimension, we first define a storage structure to store the total size of process database & table dimension reads and writes. Based on Kprobe, we accumulate and store the disk database & table dimension reads and writes in this structure. Then, we read and print the accumulated disk database & table reads and writes in the current storage structure every second to achieve the collection of database & table disk reads and writes per second.
Next, we will use Kprobe to write an eBPF program based on BCC to observe the disk IO read and write of MySQL database & table dimensions.
#### a）Analyze the kernel file system source code related VFS disk read and write processing function

```C
//Database writing to VFS
ssize_t vfs_write(struct file *file, const char __user *buf, size_t count, loff_t *pos){
  ...
}
//Database reads VFS
ssize_t vfs_read(struct file *file, char __user *buf, size_t count, loff_t *pos){
  ...
}
```
#### b）Import BPF objects from BCC
```C
//This object can embed our observation code into the observation point for execution
#include <bcc/BPF.h>

#include <string>
#include <iostream>
#include <thread>
#include <time.h>
```
#### c）Write eBPF code in C
```C

std::string strBPF = R"(
#include <linux/ptrace.h>
#include <bcc/proto.h>
#include <linux/blkdev.h>
//Define the index storage structure key for collection.
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
//Define the index storage structure value for collection
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
#### d）Observe the functions that need to be observed when reading and writing system files associated with the observation code
```C
//Used for pid replacement in ebpf code programs
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

//Specify process pid for kprobe diskio statistics
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
    /*Detection vfs_read*/
    auto attachRes = bpf.attach_kprobe("vfs_read", "kprobe__vfs_read",0,BPF_PROBE_ENTRY);
    if(!attachRes.ok()) {
        std::cerr << "attach vfs_read error,msg: "<< attachRes.msg() << std::endl;
        return 1;
    }
    /*Detection vfs_write*/
    attachRes = bpf.attach_kprobe("vfs_write", "kprobe__vfs_write");
    if(!attachRes.ok()) {
        std::cerr << "attach vfs_write error,msg: "<< attachRes.msg() << std::endl;
        return 1;
    }
    u32 on=1,off=0,key=1;
    auto flag = bpf.get_hash_table<uint32_t,uint32_t>("flag");
    /*Read and print once per second*/
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
#### e）Effect demonstration
Compile and execute the eBPF program
```Bash
#compile command
g++ -std=c++17 -o io io.cpp -lbcc -pthread
```
Specify the mysqld process pid 2004756 for diskio collection.

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZku16QCBs4xZsDQnVSic4PPPlBCbkRAf4eBcCSezIltdibVXAKrKbGzRj6J901yjGKRnZG3yvaQicjcw/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

Execute commands to connect to MySQL remotely and execute SQL.

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZku16QCBs4xZsDQnVSic4PPPxibJxHBib6WibXzKP3ia0PM4FR0kuAC5pibWYWLWKG3HmW1eqvTicCEPnuibQ/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

Print the results of the observation
![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZku16QCBs4xZsDQnVSic4PPPKiaiaHJiaibyN93YQy5LBEBtScicNIHT4Wfflvf6pyibJoAbicwmUicnb0dbOQ/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

From the above demonstration, we can see that the Client and MySQL establish a connection, execute two table SQL statements covering sbtest16 and sbtest15 respectively, print logs every second, display the process pid of mysqld, the accumulated disk read and write times and Bytes, and the involved database & table. Then we can analyze the collected data.

- If there are too many rbytes, it indicates that there may be problems such as large single query SQL scan rows or many large fields involving the database & table on the database, which will cause a large amount of disk IO occupation and even slow down the overall database.
- If wbytes are too large, it indicates that there may be a large amount of SQL writing involving the database & table in the database, such as batch data deletion, which may cause the overall database to slow down. It is recommended to update and narrow the scope, delete in multiple batches, and reduce the occupation of disk IO.

## Summary

Using eBPF technology, disk IO activities related to database & table operations can be captured and analyzed at the kernel level. This requires a deep understanding of the working principle of eBPF, the use of Kprobe, MySQL storage engine, and file system interaction mechanism. Through our introduction in the article, do you have a new understanding of eBPF technology?

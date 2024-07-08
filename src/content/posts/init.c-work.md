---
title: init.c都干了些什么？
published: 2017-08-08
description: 'init.c都干了些什么？'
image: ''
tags: [Android]
category: 'Android'
draft: false 
---

# 谁能告诉我，init.c都干了些什么？

> base on android 6.0

### Code path
> system/core/init/init.c  
> system/core/init/parser.c  
> system/core/init/init_parser.c  

在c的世界里，万物的入口都是`main(...)`，所以，让我们荡起双桨，走进 main 的海洋。

...
...

淹死了...

O__O "… 

首先，我先把代码拷贝一份进来，加点注释瞧一瞧

### main(...)

````java
int main(int argc, char **argv)
{
    int fd_count = 0;
    // 定义一个结构体数组，pullfd
    struct pollfd ufds[4];
    // 指针
    char *tmpdev;
    char* debuggable;
    char tmp[32];
    int property_set_fd_init = 0;
    int signal_fd_init = 0;
    int keychord_fd_init = 0;
    bool is_charger = false;
    bool is_ffbm = false;
	
	// 如果main函数的argv第一个元素等于"ueventd"，则执行
	// ueventd_main函数
    if (!strcmp(basename(argv[0]), "ueventd"))
        return ueventd_main(argc, argv);

	// 如果main函数的argv第一个元素等于"watchdogd"，则
	// 执行watchdogd_main函数
    if (!strcmp(basename(argv[0]), "watchdogd"))
        return watchdogd_main(argc, argv);

    /* 清空 umask */
    umask(0);

        /* Get the basic filesystem setup we need put
         * together in the initramdisk on / and then we'll
         * let the rc file figure out the rest.
         */
    // 先创建3个必要的文件夹
    mkdir("/dev", 0755);
    mkdir("/proc", 0755);
    mkdir("/sys", 0755);

	// 将tmpfs挂载到/dev下
    mount("tmpfs", "/dev", "tmpfs", MS_NOSUID, "mode=0755");
    // 创建几个文件夹
    mkdir("/dev/pts", 0755);
    mkdir("/dev/socket", 0755);
    // 将devpts挂载到/dev/pts下
    mount("devpts", "/dev/pts", "devpts", 0, NULL);
    // 将proc挂载到proc下
    mount("proc", "/proc", "proc", 0, NULL);
    // 将sysfs挂载到/sys下
    mount("sysfs", "/sys", "sysfs", 0, NULL);
    // 判断/dev/.booting是否可以读写和创建
    close(open("/dev/.booting", O_WRONLY | O_CREAT, 0000));
    
    /** main函数还没有完，先响应个中断，把main压栈了，
      * 我们先去执行open_devnull_stdio()和klog_init()
      */
    open_devnull_stdio();
    klog_init();
    ...
}
````

## 重定向标准IO
### open\_devnull\_stdio()

````java
void open_devnull_stdio(void)
{
    int fd;
    static const char *name = "/dev/__null__";
    // 创建字符设备文件，将stdio均指向/dev/__null__设备
    if (mknod(name, S_IFCHR | 0600, (1 << 8) | 3) == 0) {
	    // 打开上面创建的文件
        fd = open(name, O_RDWR);
        // 解除链接关系
        unlink(name);
        // 通过dup2指定0,1,2为fd的描述符
        // 这样标准读、写、错误都会输出到
        // /dev/__null__设备
        // 0:标准输入
        // 1:标准输出
        // 2:标准错误
            dup2(fd, 0);
            dup2(fd, 1);
            dup2(fd, 2);
            if (fd > 2) {
                close(fd);
            }
            return;
        }
    }

    exit(1);
}
````

>- mknod命令用于创建Linux中的字符设备文件和块设备文件。[参考](http://man.linuxde.net/mknod)

>- unlink函数的另一个用途就是用来创建临时文件，如果在程序中使用open创建了一个文件后，然后立即使用 unlink 函数删除文件，由于此时进程正在打开该文件，所以系统并不会释放该文件的 inode 节点，而只是删除其目录项。当进程退出时，该inode节点将会立即被释放。[参考](http://www.cnblogs.com/frank-yxs/p/5925687.html)

> - dup函数，利用dup函数，我们可以复制一个描述符。传给该函数一个既有的描述符，它就会返回一个新的描述符，这个新的描述符是传给它的描述符的拷贝。这意味着，这两个描述符共享同一个数据结构。例如，如果我们对一个文件描述符执行lseek操作，得到的第一个文件的位置和第二个是一样的。[参考](http://www.linuxdiyf.com/linux/31958.html)

> - dup2函数跟dup函数相似，但dup2函数允许调用者规定一个有效描述符和目标描述符的id。dup2函数成功返回时，目标描述符（dup2函数的第二个参数）将变成源描述符（dup2函数的第一个参数）的复制品，换句话说，两个文件描述符现在都指向同一个文件，并且是函数第一个参数指向的文件。[参考](http://www.linuxdiyf.com/linux/31958.html)

> - 描述符：文件描述符（file descriptor）是内核为了高效管理已被打开的文件所创建的索引，其是一个非负整数（通常是小整数），用于指代被打开的文件，所有执行I/O操作的系统调用都通过文件描述符。程序刚刚启动的时候，0是标准输入，1是标准输出，2是标准错误。如果此时去打开一个新的文件，它的文件描述符会是3。[参考](http://blog.csdn.net/cywosp/article/details/38965239)

## 日志服务
### klog_init(void)
````java
// 初始化日志系统
void klog_init(void)
{
    static const char *name = "/dev/__kmsg__";

    if (klog_fd >= 0) return; /* Already initialized */
	// 上面介绍过，mknod是用来创建字符设备文件的
    if (mknod(name, S_IFCHR | 0600, (1 << 8) | 11) == 0) {
        // 打开日志文件，记录描述符
        klog_fd = open(name, O_WRONLY);
        if (klog_fd < 0)
                return;
        fcntl(klog_fd, F_SETFD, FD_CLOEXEC);
        // unlink之后，只有当前进程可以访问
        // 这个文件
        unlink(name);
    }
}
````

> - fcntl: fcntl()针对(文件)描述符提供控制。参数fd是被参数cmd操作(如下面的描述)的描述符。 [参考](http://www.cnblogs.com/lonelycatcher/archive/2011/12/22/2297349.html)
> - F\_SETFD：设置close-on-exec标志，该标志以参数arg的FD\_CLOEXEC位决定，应当了解很多现存的涉及文件描述符标志的程序并不使用常数 FD_CLOEXEC，而是将此标志设置为0(系统默认，在exec时不关闭)或1(在exec时关闭)。[参考](http://www.cnblogs.com/lonelycatcher/archive/2011/12/22/2297349.html)

## 属性服务

### property_init();

````java
int main(int argc, char **argv)
{
    ...
    // 上面两个函数执行完毕以后，CPU又回到main方法
    // 先将压栈的数据出栈，我们继续往下走
    property_init();
}

听起来像是对属性的初始化
void property_init(void)
{
    init_property_area();
}
	
// 定义一个struct类型的结构体
// 里面包含了size和fd两个属性
typedef struct {
    size_t size;
    int fd;
} workspace;
    
// 声明这个结构体
static workspace pa_workspace;
    
static int init_property_area(void)
{
    // 先判断有没有初始化过
    if (property_area_inited)
        return -1;
	//  执行系统属性初始化
    if(__system_property_area_init())
        return -1;
	// 打开文件，记录fd和size
    if(init_workspace(&pa_workspace, 0))
        return -1;
	
	// fcntl函数上面有说明
    fcntl(pa_workspace.fd, F_SETFD, FD_CLOEXEC);
	
	// 初始化完成
    property_area_inited = 1;
    return 0;
}
	
// 看函数名的意思是系统属性初始化
int __system_property_area_init()
{
    return map_prop_area_rw();
}

static int map_prop_area_rw()
{
    /* dev is a tmpfs that we can use to carve a shared workspace
     * out of, so let's do that...
     * 打开这个可恶的属性文件
     */
    const int fd = open(property_filename,
                        O_RDWR | O_CREAT | O_NOFOLLOW | O_CLOEXEC | O_EXCL, 0444);
	
	// 这里有块检查打开文件失败情况被我删掉了
    ...

    // TODO: Is this really required ? Does android run on any kernels that
    // don't support O_CLOEXEC ?
    
  /** 
    * 上面那个TODO啥意思，他好像在抱怨说Android
    * 不需要专门调用fcntl，因为都支持O_CLOEXEC
    * 我水平有限，纯属瞎猜
    * 上面有介绍过fcntl，但是好像没有介绍它的返回
    * 值，是这样的，如果出错，统一都返回-1，其它的
    * 返回值跟命令有关，在代码块的下面有介绍
    */
    const int ret = fcntl(fd, F_SETFD, FD_CLOEXEC);
    if (ret < 0) {
        close(fd);
        return -1;
    }
	// 将fd文件调整为指定大小
    if (ftruncate(fd, PA_SIZE) < 0) {
        close(fd);
        return -1;
    }

    pa_size = PA_SIZE;
    pa_data_size = pa_size - sizeof(prop_area);
    compat_mode = false;
	
	// 建立虚拟地址到fd指向文件的内存映射
	// 这样以后对这块内存的操作都会被自动
	// 写入到硬盘中去
    void *const memory_area = mmap(NULL, pa_size, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
    if (memory_area == MAP_FAILED) {
        close(fd);
        return -1;
    }
	/**
	 * prop_area是个结构体在下面有列出
	 * 这句话就是在进程堆中申请一块memory_area大小的内存空间
	 * 并让__system_property_area__指向这块空间
	 */
    prop_area *pa = new(memory_area) prop_area(PROP_AREA_MAGIC, PROP_AREA_VERSION);

    /* plug into the lib property services */
    __system_property_area__ = pa;
	
	// 关闭文件，大功告成
    close(fd);
    return 0;
}

struct prop_area {
    uint32_t bytes_used;
    volatile uint32_t serial;
    uint32_t magic;
    uint32_t version;
    uint32_t reserved[28];
    char data[0];

    prop_area(const uint32_t magic, const uint32_t version) :
        serial(0), magic(magic), version(version) {
        memset(reserved, 0, sizeof(reserved));
        // Allocate enough space for the root node.
        bytes_used = sizeof(prop_bt);
    }

    
// 这个函数主要是打开属性文件
// 记录fd和size，当然，目前
// size = 0
static int init_workspace(workspace *w, size_t size)
{
    void *data;
    int fd = open(PROP_FILENAME, O_RDONLY | O_NOFOLLOW);
    if (fd < 0)
        return -1;

    w->size = size;
    w->fd = fd;
    return 0;
}
````

> - fcntl()的返回值与命令有关。如果出错，所有命令都返回－1，如果成功则返回某个其他值。下列三个命令有特定返回值：F_DUPFD , F_GETFD , F_GETFL以及F_GETOWN。
    F_DUPFD   返回新的文件描述符
    F_GETFD   返回相应标志
    F_GETFL , F_GETOWN   返回一个正的进程ID或负的进程组ID
    [参考](http://www.cnblogs.com/lonelycatcher/archive/2011/12/22/2297349.html)
> - ftruncate会将参数fd指定的文件大小改为参数length指定的大小。[参考](https://baike.baidu.com/item/ftruncate/1042822?fr=aladdin)
> - mmap是一种内存映射文件的方法，即将一个文件或者其它对象映射到进程的地址空间，实现文件磁盘地址和进程虚拟地址空间中一段虚拟地址的一一对映关系。实现这样的映射关系后，进程就可以采用指针的方式读写操作这一段内存，而系统会自动回写脏页面到对应的文件磁盘上，即完成了对文件的操作而不必再调用read,write等系统调用函数。相反，内核空间对这段区域的修改也直接反映用户空间，从而可以实现不同进程间的文件共享。[参考](http://www.cnblogs.com/huxiao-tee/p/4660352.html)

## 获取硬件信息
### get\_hardware\_name(...)

````java
// 从函数名猜测可能是获取硬件名称
// 将结果写到hardware和revision里面
void get_hardware_name(char *hardware, unsigned int *revision)
{
    const char *cpuinfo = "/proc/cpuinfo";
    char *data = NULL;
    size_t len = 0, limit = 1024;
    int fd, n;
    char *x, *hw, *rev;
	
    /* Hardware string was provided on kernel command line */
    // 如果已经有数据了，直接返回
    if (hardware[0])
        return;
	
	// 打开cpuinfo这个设备文件
	// "/proc/cpuinfo"
    fd = open(cpuinfo, O_RDONLY);
    if (fd < 0) return;

    for (;;) {
        // 申请一块limit大小的内存
        // 如果申请成功，原来的那块
        // 内存数据会被拷贝到新的内
        // 存，同时会释放原来的内存
        x = realloc(data, limit);
        if (!x) {
            ERROR("Failed to allocate memory to read %s\n", cpuinfo);
            goto done;
        }
        data = x;
		
		// 读取fd文件的data + len -> limit这段的数据
        n = read(fd, data + len, limit - len);
        if (n < 0) {
            ERROR("Failed reading %s: %s (%d)\n", cpuinfo, strerror(errno), errno);
            goto done;
        }
        len += n;

        if (len < limit)
            break;

        /* We filled the buffer, so increase size and loop to read more */
        // 如果读取的数据超过limit，翻倍扩大limit
        limit *= 2;
    }

    data[len] = 0;
    // 找到"\nHardware" 第一次在data中出现的位置
    hw = strstr(data, "\nHardware");
    // 找到"\nRevision" 第一次在data中出现的位置
    rev = strstr(data, "\nRevision");
    
	// 如果找到，解析Hardware后面的数据存储到hardware中
    if (hw) {
        x = strstr(hw, ": ");
        if (x) {
            x += 2;
            n = 0;
            while (*x && *x != '\n') {
                if (!isspace(*x))
                    hardware[n++] = tolower(*x);
                x++;
                if (n == 31) break;
            }
            hardware[n] = 0;
        }
    }
    
	// 如果找到，解析Revision后面的数据存储到hardware中
    if (rev) {
        x = strstr(rev, ": ");
        if (x) {
            *revision = strtoul(x + 2, 0, 16);
        }
    }

done:
    close(fd);
    free(data);
}
````
    
> - strstr(...) : 找出str2字符串在str1字符串中第一次出现的位置 [参考](http://blog.csdn.net/hgj125073/article/details/8444162)


### process\_kernel\_cmdline()

````java
static void process_kernel_cmdline(void)
{
    /* don't expose the raw commandline to nonpriv processes */
    /**
     * r = 4 可以读
     * w = 2 可以写
     * x = 1 可以执行
     * 0440中
     * 第二位4表示ower可以读
     * 第二位4表示group可以读
     * 第三位0表示其他用户无权限
     */
    chmod("/proc/cmdline", 0440);

    /* first pass does the common stuff, and finds if we are in qemu.
     * second pass is only necessary for qemu to export all kernel params
     * as props.
     * 导入内核的命令
     * 这里解释了为啥去要调用两遍
     * 那为啥要调用两遍呢，进取一
     * 探究竟吧
     */
    import_kernel_cmdline(0, import_kernel_nv);
    if (qemu[0])
        import_kernel_cmdline(1, import_kernel_nv);

    /* now propogate the info given on command line to internal variables
     * used by init as well as the current required properties
     */
    export_kernel_boot_props();
}
    
/**
 * 注意，这里的import_kernel_nv是个方法
 * 相当于这里传进一个方法的指针
 */
void import_kernel_cmdline(int in_qemu,
                           void (*import_kernel_nv)(char *name, int in_qemu))
{
    char cmdline[2048];
    char *ptr;
    int fd;
	 // 以只读的方式打开这个文件
    fd = open("/proc/cmdline", O_RDONLY);
    if (fd >= 0) {
    	  // 将信息读取到cmdline中
        int n = read(fd, cmdline, sizeof(cmdline) - 1);
        if (n < 0) n = 0;

        /* get rid of trailing newline, it happens */
        if (n > 0 && cmdline[n-1] == '\n') n--;

        cmdline[n] = 0;
        close(fd);
    } else {
        cmdline[0] = 0;
    }
	
    ptr = cmdline;
    // 指针不为空且指向数组第一位不为空
    while (ptr && *ptr) {
    	  // 前面介绍过一个函数:strstr()
    	  // 跟这个类似，所以这个一定是获取
    	  // char第一次出现的位置
        char *x = strchr(ptr, ' ');
        if (x != 0) *x++ = 0;
        import_kernel_nv(ptr, in_qemu);
        ptr = x;
    }
}
````

> - strchr：strchr() 用来查找某字符在字符串中首次出现的位置[参考](http://c.biancheng.net/cpp/html/161.html)

````java
// 这就是上面传入的那个函数
// 这个for_emulator就是之前那个函数
// 两次调用的区别，第一次是0，第二次是1
static void import_kernel_nv(char *name, int for_emulator)
{		
	 // 找第一个‘=’出现的位置
    char *value = strchr(name, '=');
    int name_len = strlen(name);

    if (value == 0) return;
    *value++ = 0;
    if (name_len == 0) return;
	
	 // 由于第一次传0，所以不会进这个if
    if (for_emulator) {
        /* in the emulator, export any kernel option with the
         * ro.kernel. prefix */
        char buff[PROP_NAME_MAX];
        // 从name中取sizeof(buff)个字符，传入buff中
        int len = snprintf( buff, sizeof(buff), "ro.kernel.%s", name );

        if (len < (int)sizeof(buff))
        	   // 这是啥？
            property_set( buff, value );
        return;
    }

    if (!strcmp(name,"qemu")) {
        strlcpy(qemu, value, sizeof(qemu));
    } else if (!strcmp(name,BOARD_CHARGING_CMDLINE_NAME)) {
        strlcpy(battchg_pause, value, sizeof(battchg_pause));
    } else if (!strncmp(name, "androidboot.", 12) && name_len > 12) {
        const char *boot_prop_name = name + 12;
        char prop[PROP_NAME_MAX];
        int cnt;
		  
		  // 从boot_prop_name中取sizeof(prop)个字符，传入prop中
        cnt = snprintf(prop, sizeof(prop), "ro.boot.%s", boot_prop_name);
        if (cnt < PROP_NAME_MAX)
            property_set(prop, value);
    }
}
````

> - sprintf()：将格式化的数据写入字符串 [参考](http://c.biancheng.net/cpp/html/2417.html)

````java
// 将数据存储到属性服务中
int property_set(const char *name, const char *value)
{
    prop_info *pi;
    int ret;
	
    size_t namelen = strlen(name);
    size_t valuelen = strlen(value);
	 
	 // 检查错误
    if (!is_legal_property_name(name, namelen)) return -1;
    if (valuelen >= PROP_VALUE_MAX) return -1;
	
	 // 查找目前有没有这个属性
    pi = (prop_info*) __system_property_find(name);
	
	 // 如果有，执行更新
    if(pi != 0) {
        /* ro.* properties may NEVER be modified once set */
        if(!strncmp(name, "ro.", 3)) return -1;
	
        __system_property_update(pi, value, valuelen);
    } else {
        // 否则，执行添加
        ret = __system_property_add(name, namelen, value, valuelen);
        if (ret < 0) {
            ERROR("Failed to set '%s'='%s'\n", name, value);
            return ret;
        }
    }
    /* If name starts with "net." treat as a DNS property. */
    if (strncmp("net.", name, strlen("net.")) == 0)  {
        if (strcmp("net.change", name) == 0) {
            return 0;
        }
       /*
        * The 'net.change' property is a special property used track when any
        * 'net.*' property name is updated. It is _ONLY_ updated here. Its value
        * contains the last updated 'net.*' property.
        */
        property_set("net.change", name);
    } else if (persistent_properties_loaded &&
            strncmp("persist.", name, strlen("persist.")) == 0) {
        /*
         * Don't write properties to disk until after we have read all default properties
         * to prevent them from being overwritten by default values.
         */
        write_persistent_property(name, value);
    } else if (strcmp("selinux.reload_policy", name) == 0 &&
               strcmp("1", value) == 0) {
        selinux_reload_policy();
    }
    property_changed(name, value);
    return 0;
}
    
// 如果修改属性成功，执行这个方法
// 通知属性被修改
void property_changed(const char *name, const char *value)
{
    if (property_triggers_enabled)
        queue_property_triggers(name, value);
}

````
    
### init\_parse\_config\_file(...)

解析配置文件

````c
int main(int argc, char **argv)
{
	...
	init_parse_config_file("/init.rc");
	...
}

// 读取配置文件
int init_parse_config_file(const char *fn)
{
    char *data;
    // 读取init.rc文件并存入data中
    // init.rc是一个配置文件
    data = read_file(fn, 0);
    if (!data) return -1;
	 // 读完以后，使用这个函数解析配置文件
    parse_config(fn, data);
    DUMP();
    return 0;
}
// parse_state结构体，在
// paser.h文件中
struct parse_state
{
    char *ptr;
    char *text;
    int line;
    int nexttoken;
    void *context;
    void (*parse_line)(struct parse_state *state, int nargs, char **args);
    const char *filename;
    void *priv;
};

// 解析配置文件
static void parse_config(const char *fn, char *s)
{
    struct parse_state state;
    struct listnode import_list;
    struct listnode *node;
    char *args[INIT_PARSER_MAXARGS];
    int nargs;

    nargs = 0;
    // 先给parse_state函数初始化
    state.filename = fn;
    state.line = 0;
    state.ptr = s;
    state.nexttoken = 0;
    // state.parse_line 默认指向pare_line_no_ope函数
    // 这是一个空的函数，一会它会根据不同的命令选择不同的函数
    state.parse_line = parse_line_no_op;

	// 初始化listnode
    list_init(&import_list);
    state.priv = &import_list;

	// 进入for循环开始解析文件
    for (;;) {
    	 /** 
    	  * next_token这个函数很长，所以就不列出来了
    	  * 
		  * on early-init
    	  * # Set init and its forked children's oom_adj.
   		  *	 write /proc/1/oom_score_adj -1000 
   		  *	 
   		  *	 以上面这句话为例：
   		  *	 首先进来碰到的是‘o’，则next_token发现它既不是'\n'
   		  *	 也不是 ‘\t’,' ','\r'
   		  *	 也不是 '#'
   		  *	 所以 goto 到 text 然后将state->text指向o，
   		  *	 然后顺着往下执行进入下一个for循环，判断第二个字符是'n'
   		  *	 发现不少特殊字符，指针 +1 ，然后碰到 ' ' 空格，指针 +1
   		  *	 同时 goto 到 textdone，函数返回 T_TEXT
   		  *	 然后 parse_config 函数的switch会走向 case T_TEXT，
   		  *	 让args[0]指向 state -> text
   		  *	 上面我们提到过，这个state -> text 现在指向的就是 'o'
   		  *	 的位置，然后重复上一个步骤，这次将解析 "early-init" 
   		  *	 中间没有特殊字符，最后遇到'\n'，指针 +1 同时要让 
   		  *	 state -> nexttoken = T_NEWLINE，goto 到 textdone，
   		  *	 方法返回 T_TEXT这时候，arg是这样的：
   		  *	 arg[0] -> 'o','n'
   		  *	 arg[1] -> 'e','a','r','l','y','-','i','n','i','t'
   		  *	 然后下次进next_token发现state->nexttoken不为0，
   		  *	 是T_NEWLINE，所以会返回state->nexttoken 
   		  *	 同时将state->nexttoken置为0
   		  *	 这时候进入 case T_NEWLINE，则先会执行lookup_keyword(args[0))
   		  *	 这个函数会寻找这一行的关键字，并根据关键字返回对应的代号，如
   		  *	 on 关键字的编号为 K_on，然后判断key_word是否为SECTION，
   		  *	 这里说一下 keyword 一共分三种类型：
   		  *	 #define SECTION 0x01
   		  *	 #define COMMAND 0x02
   		  *	 #define OPTION  0x04
   		  *	 这里有一个数组专门存储每个关键字和它的类型，所以通过对比发现
   		  *	 'on'是SECTION类型，所以进入if，然后执行state.parse_line
   		  *	 但是由于我们之前给state.parse_line一个没有干任何事情的
   		  *	 state.parse_line -> parse_line_no_op
   		  *	 所以，这里它什么也没有做，接着执行parse_new_section，它又会做
   		  *	 什么呢，它会根据关键字的类型给state -> parse_line合适的解析方法
   		  *	 同时将early-init这个插入到一个全局列表中
   		  *	 on关键字对应的方法为：parse_line_action
   		  *	 然后for循环进入下一次，开始读取第二行，发现第二行是'#'号开头，则会
   		  *	 忽略这一行
   		  *	 进入第三行，先读取write，让：
   		  *	 arg[0] -> write
   		  *	 arg[1] -> /proc/1/oom_score_adj
   		  *	 arg[2] -> -1000
   		  *	 然后if(kw_is(kw,SECTION)返回false
   		  *	 进入else，执行state.parse_line，此刻的parse_line = parse_line_action
    	  */ 
        switch (next_token(&state)) {
        case T_EOF:
            state.parse_line(&state, 0, 0);
            goto parser_done;
        case T_NEWLINE:
            state.line++;
            if (nargs) {
                int kw = lookup_keyword(args[0]);
                if (kw_is(kw, SECTION)) {
                    state.parse_line(&state, 0, 0);
                    parse_new_section(&state, kw, nargs, args);
                } else {
                    state.parse_line(&state, nargs, args);
                }
                nargs = 0;
            }
            break;
        case T_TEXT:
            if (nargs < INIT_PARSER_MAXARGS) {
                args[nargs++] = state.text;
            }
            break;
        }
    }

parser_done:
    list_for_each(node, &import_list) {
         struct import *import = node_to_item(node, struct import, list);
         int ret;

         INFO("importing '%s'", import->filename);
         ret = init_parse_config_file(import->filename);
         if (ret)
             ERROR("could not import file '%s' from '%s'\n",
                   import->filename, fn);
    }
}

// 根据不同的key_word，匹配不同的state->parse_line
static void parse_new_section(struct parse_state *state, int kw,
                       int nargs, char **args)
{
    printf("[ %s %s ]\n", args[0],
           nargs > 1 ? args[1] : "");
    switch(kw) {
    case K_service:
    case K_service_redefine:
        state->context = parse_service(state, nargs, args, (kw == K_service_redefine));
        if (state->context) {
            state->parse_line = parse_line_service;
            return;
        }
        break;
    case K_on:
        state->context = parse_action(state, nargs, args);
        if (state->context) {
            state->parse_line = parse_line_action;
            return;
        }
        break;
    case K_import:
        parse_import(state, nargs, args);
        break;
    }
    state->parse_line = parse_line_no_op;
}


static void *parse_action(struct parse_state *state, int nargs, char **args)
{
    struct action *act;
    // 检查args的合法性
    if (nargs < 2) {
        parse_error(state, "actions must have a trigger\n");
        return 0;
    }
    if (nargs > 2) {
        parse_error(state, "actions may not have extra parameters\n");
        return 0;
    }
    act = calloc(1, sizeof(*act));
    act->name = args[1];
    list_init(&act->commands);
    list_init(&act->qlist);
    list_add_tail(&action_list, &act->alist);
        /* XXX add to hash */
    return act;
}

struct action {
        /* node in list of all actions */
    struct listnode alist;
        /* node in the queue of pending actions */
    struct listnode qlist;
        /* node in list of actions for a trigger */
    struct listnode tlist;

    unsigned hash;
    const char *name;

    struct listnode commands;
    struct command *current;
};

````

> calloc:函数malloc()和calloc()都可以用来动态分配内存空间,但两者稍有区别。 
malloc()函数有一个参数,即要分配的内存空间的大小:   
void *malloc(size_t size);   
calloc()函数有两个参数,分别为元素的数目和每个元素的大小,这两个参数的乘积就是要分配的内存空间的大小。   
void *calloc(size_t numElements,size_t sizeOfElement); [参考](http://www.cnblogs.com/stevenwuzheng/p/5484986.html)

#### list讲解
讲到这里，我觉得有必要讲一下双向链表的用法，这个在Android源码中使用非常的广泛，常规用法是这样的

````c
// 首先声明一个链表
static list_declare(service_list);

/**
 * 然后，声明一个结构体，其中这个结构体中
 * 一定要包含listnode，节点主要包含
 * 的信息是它的上一个节点和下一个节点的
 * 地址
 */
struct service {
	listnode slist;
	char* name;
	...
}

/**
 * 当我们要插入数据到链表的时候
 * 可以调用下面这个方法
 */
 // 先声明一个结构体
 struct service *svc;
 // 将结构体插入到service_list中
 // 其实只是将节点信息跟service_list中
 // 已有的链接起来
 list_add_tail(&service_list, &svc->slist);
 // 或者
 list_add_head(&service_list, &svc->slist);
 
 // 遍历结构体可以这样
 struct listnode *node;
 list_for_each(node, &service_list) {
     // 这里node就是我们遍历到的节点
 }
 
 // 那拿到节点怎么获取节点所对应的结构体呢？
 // 只拿到节点也没有意义呀？
 // 山人自有妙计
 // 通过node_to_item这个方法，可以根据
 // 节点的地址算出结构体的起始地址
 // 哦，这个slist是你在struct声明的节点名称
 struct service *svc;
 svc = node_to_item(node, struct service, slist);
 
 // 删除节点也很简单
 list_remove(node);
````

#### 至此，解析init.rc完成

-------------------------------------
> 解析zygote我没有写，自己看了一下，太长了，懒得写...

接下来zygote这个进程是如何启动的吧

在哪里触发启动zygote我好想还没有找到，但是我通过查资料找到它使用哪个函数用来创建zygote的，答案就是

````c
// 这里的args参数只有一个 
// service name "main"
int do_class_start(int nargs, char **args)
{
    char prop[PROP_NAME_MAX];
    snprintf(prop, PROP_NAME_MAX, "class_start:%s", args[1]);

    /* Starting a class does not start services
     * which are explicitly disabled.  They must
     * be started individually.
     */
     // service_for_each_class 的作用是寻找
     // args[1]既main名称的service,并调用service_start_if_not_disabled
     // 处理
    service_for_each_class(args[1], service_start_if_not_disabled);
    action_for_each_trigger(prop, action_add_queue_tail);
    return 0;
}

然后就到了
static void service_start_if_not_disabled(struct service *svc)
{
    if (!(svc->flags & SVC_DISABLED)) {
        service_start(svc, NULL);
    } else {
        svc->flags |= SVC_DISABLED_START;
    }
}

// 最后又回到init.c了
// 绕了个大弯子又回来了
// 注意，svc就是解析的
// service

// 下面这个是zygote的配置信息
//service zygote /system/bin/app_process64 -Xzygote /system/bin --zygote --start-system-server
//    class main
//    socket zygote stream 660 root system
//    # option  command  args
//    onrestart write /sys/android_power/request_state wake
//    onrestart write /sys/power/state on
//    onrestart restart media
//    onrestart restart netd

void service_start(struct service *svc, const char *dynamic_args)
{
    struct stat s;
    pid_t pid;
    int needs_console;
    int n;
    char *scon = NULL;
    int rc;

        /* starting a service removes it from the disabled or reset
         * state and immediately takes it out of the restarting
         * state if it was in there
         */
    svc->flags &= (~(SVC_DISABLED|SVC_RESTARTING|SVC_RESET|SVC_RESTART|SVC_DISABLED_START));
    svc->time_started = 0;

        /* running processes require no additional work -- if
         * they're in the process of exiting, we've ensured
         * that they will immediately restart on exit, unless
         * they are ONESHOT
         */
    // 如果正在运行，就返回
    if (svc->flags & SVC_RUNNING) {
        return;
    }

    needs_console = (svc->flags & SVC_CONSOLE) ? 1 : 0;
    if (needs_console && (!have_console)) {
        ERROR("service '%s' requires console\n", svc->name);
        svc->flags |= SVC_DISABLED;
        return;
    }
	

	/** service zygote /system/bin/app_process64 -Xzygote /system/bin --zygote --start-system-server
	 * 上面这句话第一个单词表示它是个service
	 * 第二个是它的名字 zygote
	 * 第三个开始后面的都是它的参数
	 * 这些参数会存在service -> args中
	 * 而 stat 这个函数的作用是
	 * 通过文件名filename获取文件信息，并保存在buf所指的结构体stat中
	 * 参考自：http://blog.csdn.net/tigerjibo/article/details/11695763
	 * 所以 args[0] = zygote /system/bin/app_process64
	 * 读取这个路径下的信息
	 * 并将这些信息保存到 s 结构体中
	 * 其实是为了判断这个文件有没有存在
	 */
    if (stat(svc->args[0], &s) != 0) {
        ERROR("cannot find '%s', disabling '%s'\n", svc->args[0], svc->name);
        svc->flags |= SVC_DISABLED;
        return;
    }

	 // 这里因为dynamic_args为NULL，所以不用管它
    if ((!(svc->flags & SVC_ONESHOT)) && dynamic_args) {
        ERROR("service '%s' must be one-shot to use dynamic args, disabling\n",
               svc->args[0]);
        svc->flags |= SVC_DISABLED;
        return;
    }

	 // selinux 我也不懂，跳过
    if (is_selinux_enabled() > 0) {
        if (svc->seclabel) {
            scon = strdup(svc->seclabel);
            if (!scon) {
                ERROR("Out of memory while starting '%s'\n", svc->name);
                return;
            }
        } else {
            char *mycon = NULL, *fcon = NULL;

            INFO("computing context for service '%s'\n", svc->args[0]);
            rc = getcon(&mycon);
            if (rc < 0) {
                ERROR("could not get context while starting '%s'\n", svc->name);
                return;
            }

            rc = getfilecon(svc->args[0], &fcon);
            if (rc < 0) {
                ERROR("could not get context while starting '%s'\n", svc->name);
                freecon(mycon);
                return;
            }

            rc = security_compute_create(mycon, fcon, string_to_security_class("process"), &scon);
            if (rc == 0 && !strcmp(scon, mycon)) {
                ERROR("Warning!  Service %s needs a SELinux domain defined; please fix!\n", svc->name);
            }
            freecon(mycon);
            freecon(fcon);
            if (rc < 0) {
                ERROR("could not get context while starting '%s'\n", svc->name);
                return;
            }
        }
    }

    NOTICE("starting '%s'\n", svc->name);
	// 判断属性服务有没有启动，如果启动，就更新service状态？
    if (properties_inited())
        notify_service_state(svc->name, "starting");
	// 终于看到个关键的代码
	// fork()，克隆父进程，linux里面爷爷级别的方法
	//      一个进程，包括代码、数据和分配给进程的资源。
	// fork（）函数通过系统调用创建一个与原来进程几乎完全相同的进程，
	// 也就是两个进程可以做完全相同的事，但如果初始参数或者传入的变量不同，
	// 两个进程也可以做不同的事。
	// 参考：http://blog.csdn.net/jason314/article/details/5640969

    pid = fork();
    
	// 这里是啥意思呢？
	// 在fork函数执行完毕后，如果创建新进程成功，则出现两个进程，
	// 一个是子进程，一个是父进程。在子进程中，fork函数返回0，在父进程中，
	// fork返回新创建子进程的进程ID。我们可以通过fork返回的值来判断当前
	// 进程是子进程还是父进程。所以呢，从现在往下走if(pid == 0)里面要
	// 执行的代码段就是子进程的代码，if面是两个进程都会执行的代码段
	// ok，那也就是说我们的zygote进程其实已经创建了，那下面的代码
	// 猜测应该就是给zygote进程进行一些赋值和其他的功能配置
    if (pid == 0) {
    	// 这个socketinfo是为了让zygote支持与外界通信
    	// 而设置的，这时候还没有使用大名鼎鼎的Binder
        struct socketinfo *si;
        struct svcenvinfo *ei;
        char tmp[32];
        int fd, sz;
		// umask函数是减去权限 假如原来权限是 444
		// 则只需 umask(077) 后，权限变成了 (444 & 777) & ~077 = 700
		// 也就是说只有当前进程可以有读写和执行权限
        umask(077);
        if (properties_inited()) {
            get_property_workspace(&fd, &sz);
            sprintf(tmp, "%d,%d", dup(fd), sz);
            add_environment("ANDROID_PROPERTY_WORKSPACE", tmp);
        }

		// 将service中的envvars加入到环境变量中
        for (ei = svc->envvars; ei; ei = ei->next)
            add_environment(ei->name, ei->value);

        for (si = svc->sockets; si; si = si->next) {
            int socket_type = (
                    !strcmp(si->type, "stream") ? SOCK_STREAM :
                        (!strcmp(si->type, "dgram") ? SOCK_DGRAM : SOCK_SEQPACKET));
            // 创建socket服务端，至于如何创建socket
            // 这就是linux相关的知识了，过程我也不懂
            // 但是返回值，就是socket的地址了
            int s = create_socket(si->name, socket_type,
                                  si->perm, si->uid, si->gid, si->socketcon ?: scon);
            if (s >= 0) {
            	// 将socket的地址发布出去，以便其他的进程可以访问
            	// publish_socket代码很简单，其实就是讲s存在环境变量
            	// 中,socket的名称就是 "ANDROID_SOCKET_zygote"
            	// 这样根据这个约定，其他进程就可以给这个进程发消息啦
                publish_socket(si->name, s);
            }
        }

        freecon(scon);
        scon = NULL;

        if (svc->ioprio_class != IoSchedClass_NONE) {
            if (android_set_ioprio(getpid(), svc->ioprio_class, svc->ioprio_pri)) {
                ERROR("Failed to set pid %d ioprio = %d,%d: %s\n",
                      getpid(), svc->ioprio_class, svc->ioprio_pri, strerror(errno));
            }
        }

        if (needs_console) {
            setsid();
            open_console();
        } else {
            zap_stdio();
        }

#if 0
        for (n = 0; svc->args[n]; n++) {
            INFO("args[%d] = '%s'\n", n, svc->args[n]);
        }
        for (n = 0; ENV[n]; n++) {
            INFO("env[%d] = '%s'\n", n, ENV[n]);
        }
#endif

        setpgid(0, getpid());

    /* as requested, set our gid, supplemental gids, and uid */
        if (svc->gid) {
            if (setgid(svc->gid) != 0) {
                ERROR("setgid failed: %s\n", strerror(errno));
                _exit(127);
            }
        }
        if (svc->nr_supp_gids) {
            if (setgroups(svc->nr_supp_gids, svc->supp_gids) != 0) {
                ERROR("setgroups failed: %s\n", strerror(errno));
                _exit(127);
            }
        }
        if (svc->uid) {
            if (setuid(svc->uid) != 0) {
                ERROR("setuid failed: %s\n", strerror(errno));
                _exit(127);
            }
        }
        if (svc->seclabel) {
            if (is_selinux_enabled() > 0 && setexeccon(svc->seclabel) < 0) {
                ERROR("cannot setexeccon('%s'): %s\n", svc->seclabel, strerror(errno));
                _exit(127);
            }
        }
		// 由于前面我们说dynamic_args是NULL
		// 所以这里会进入if，执行execve
		// 这个函数的作用是执行svc->args[0]所指向的文件
		// 参数就是svc-args，然后环境变量是ENV
		// 如果成功无返回，失败返回-1
		// 而args[0]所指向的路径是啥呢？
		// 就是我们上面提到很多遍的
		// system/bin/app_process 这是个可执行文件
		// 对应的源代码是 app_main.cpp，不要问我为什么
		// 知道，因为我也是查资料找到的
		// 通过这句话，代码转入app_main.cpp，开始对
		// zygote进一步初始化
        if (!dynamic_args) {
            if (execve(svc->args[0], (char**) svc->args, (char**) ENV) < 0) {
                ERROR("cannot execve('%s'): %s\n", svc->args[0], strerror(errno));
            }
        } else {
            char *arg_ptrs[INIT_PARSER_MAXARGS+1];
            int arg_idx = svc->nargs;
            char *tmp = strdup(dynamic_args);
            char *next = tmp;
            char *bword;

            /* Copy the static arguments */
            memcpy(arg_ptrs, svc->args, (svc->nargs * sizeof(char *)));

            while((bword = strsep(&next, " "))) {
                arg_ptrs[arg_idx++] = bword;
                if (arg_idx == INIT_PARSER_MAXARGS)
                    break;
            }
            arg_ptrs[arg_idx] = '\0';
            execve(svc->args[0], (char**) arg_ptrs, (char**) ENV);
        }
        _exit(127);
    }

    freecon(scon);

    if (pid < 0) {
        ERROR("failed to start '%s'\n", svc->name);
        svc->pid = 0;
        return;
    }

    svc->time_started = gettime();
    svc->pid = pid;
    svc->flags |= SVC_RUNNING;

    if (properties_inited())
        notify_service_state(svc->name, "running");
}
````

到此呢，zygote 进程算是启动起来了，总结一下：
  
- fork()父进程
- socket 创建
- 将 socket 地址存入环境变量
- 配置 zygote 进程
- 执行system/bin/app_process文件
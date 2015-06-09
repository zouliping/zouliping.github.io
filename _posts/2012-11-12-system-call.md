--- 
layout: post
title: 系统调用
category: 操作系统
comments: true
image:
  feature: texture-feature-02.jpg
excerpt: 操作系统的第二个实验是在0.11下添加两个系统调用，iam 和 whoami.
---
<section id="table-of-contents" class="toc">
  <header>
    <h3>Overview</h3>
  </header>
<div id="drawer" markdown="1">
*  Auto generated table of contents
{:toc}
</div>
</section>

由于时间有些久远了，所以其中的内容可能稍有不当和具有一定的不完整性，敬请见谅！

### 实验内容概述

操作系统的第二个实验是在0.11下添加两个系统调用，iam和whoami.

#### iam()

第一个系统调用是iam()，其原型为：

{% highlight C linenos %}
int iam(const char * name);
{% endhighlight %}

完成的功能是将字符串参数name的内容拷贝到内核中保存下来。要求name的长度不能超过23个字符。返回值是拷贝的字符数。如果name的字符个数超过了23，则返回“-1”，并置errno为EINVAL。

#### whoami()

第二个系统调用是whoami()，其原型为：

{% highlight C linenos %}
int whoami(char* name, unsigned int size);
{% endhighlight %}

它将内核中由iam()保存的名字拷贝到name指向的用户地址空间中，同时确保不会对name越界访存（name的大小由size说明）。返回值是拷贝的字符数。如果size小于需要的空间，则返回“-1”，并置errno为EINVAL。

在仔细阅读了指导书后就会对应用程序如何调用系统调用有一个整体上的理解。

### 实验过程

下面是实验的一些过程：

添加系统调用时需要修改include/unistd.h文件，在系统原本的72个系统调用后面，给我们新添加的系统调用iam和whoami加上它们的系统调用编号72和73.

{% highlight C linenos %}
#define __NR_sgetmask	68
#define __NR_ssetmask	69
#define __NR_setreuid	70
#define __NR_setregid	71
#define __NR_iam	72
#define __NR_whoami	73
{% endhighlight %}

接着在/include/linux/sys.h文件里，仿照系统原本的系统调用的写法，将我们要添加的两个系统调用做类似的处理。

{% highlight C linenos %}
extern int sys_whoami();
extern int sys_iam();
{% endhighlight %}

同时，还需要在include/linux/sys.h的函数表fn_ptr sys_call_table[]中添加两个函数引用sys_iam和sys_whoami。在这里要注意的是，添加的位置顺序必须和你刚才添加系统调用编号的顺序相同,顺序一定不要弄错了。

{% highlight C linenos %}
fn_ptr sys_call_table[] = { sys_setup, sys_exit, sys_fork, sys_read,
sys_write, sys_open, sys_close, sys_waitpid, sys_creat, sys_link,
sys_unlink, sys_execve, sys_chdir, sys_time, sys_mknod, sys_chmod,
sys_chown, sys_break, sys_stat, sys_lseek, sys_getpid, sys_mount,
sys_umount, sys_setuid, sys_getuid, sys_stime, sys_ptrace, sys_alarm,
sys_fstat, sys_pause, sys_utime, sys_stty, sys_gtty, sys_access,
sys_nice, sys_ftime, sys_sync, sys_kill, sys_rename, sys_mkdir,
sys_rmdir, sys_dup, sys_pipe, sys_times, sys_prof, sys_brk, sys_setgid,
sys_getgid, sys_signal, sys_geteuid, sys_getegid, sys_acct, sys_phys,
sys_lock, sys_ioctl, sys_fcntl, sys_mpx, sys_setpgid, sys_ulimit,
sys_uname, sys_umask, sys_chroot, sys_ustat, sys_dup2, sys_getppid,
sys_getpgrp, sys_setsid, sys_sigaction, sys_sgetmask, sys_ssetmask,
sys_setreuid,sys_setregid ,sys_iam,sys_whoami};
{% endhighlight %}

接下来，在/kernel/system_call.s文件中修改系统调用的数目，将原本的72改为现在的74。

{% highlight C linenos %}
# offsets within sigaction
sa_handler = 0
sa_mask = 4
sa_flags = 8
sa_restorer = 12

nr_system_calls = 74
{% endhighlight %}

在完成了上面的全部步骤之后，可以开始who.c的编写了。这是添加系统调用的最后一步了，实现sys_iam()和sys_whoami()。在这里需要知道的是，在用户态和核心态传递数据的方法，因为指针参数传递的是应用程序所在地址空间的逻辑地址，在内核中如果直接访问这个地址，访问到的是内核空间中的数据，不会是用户空间的。所以我们就需要知道get_fs_byte()和put_fs_byte()就可以很快的实现who.c的编写了。

{% highlight C linenos %}
#include <errno.h>
#include <linux/kernel.h>
#include <asm/segment.h>

#define MAX 23
char userName[MAX+1];

int sys_iam(const char * name)
{
  int l=0,i=0;
  unsigned char n;
  char temp[MAX+1];

  while((n=get_fs_byte(&name[i]))!='\0')
  {
     l++;
     temp[i] = n;
     i++;
  }
  
  if(l>MAX)
  {
     return (-EINVAL);
  }
  else
  {
     for(i=0;i<l;i++)
     {  
        userName[i] = temp[i];
     }
     userName[i]='\0';
  }
  return (l);
}

int sys_whoami(char* name,unsigned int size)
{
  int l=0,i=0;
  while(userName[i]!='\0')
  {
     l++;
     i++;
  }

  if(l>size)
  {
    return (-EINVAL);
  }
  for(i=0;i<l;i++)
  {
     put_fs_byte(userName[i],&name[i]);
  }
  
  return (l);
}
{% endhighlight %}

到了这一步，我们要做的就是将who.c和其他linux代码编译链接到一起啦。我们要修改的就是Makefile文(/kernel/Makefile),直接根据指导书的内容将其替换就完成啦。

{% highlight C linenos %}
OBJS  = sched.o system_call.o traps.o asm.o fork.o 
        panic.o printk.o vsprintf.o sys.o exit.o 
        signal.o mktime.o
{% endhighlight %}
        
改为：

{% highlight C linenos %}
OBJS  = sched.o system_call.o traps.o asm.o fork.o
        panic.o printk.o vsprintf.o sys.o exit.o 
        signal.o mktime.o who.o
{% endhighlight %}        
        
另一处：

{% highlight C linenos %}
### Dependencies:
exit.s exit.o: exit.c ../include/errno.h ../include/signal.h 
  ../include/sys/types.h ../include/sys/wait.h ../include/linux/sched.h 
  ../include/linux/head.h ../include/linux/fs.h ../include/linux/mm.h
  ../include/linux/kernel.h ../include/linux/tty.h ../include/termios.h 
  ../include/asm/segment.h
{% endhighlight %}
  
改为：

{% highlight C linenos %}
### Dependencies:
who.s who.o: who.c ../include/linux/kernel.h ../include/unistd.h
exit.s exit.o: exit.c ../include/errno.h ../include/signal.h 
  ../include/sys/types.h ../include/sys/wait.h ../include/linux/sched.h 
  ../include/linux/head.h ../include/linux/fs.h ../include/linux/mm.h 
  ../include/linux/kernel.h ../include/linux/tty.h ../include/termios.h 
  ../include/asm/segment.h
{% endhighlight %}
  
最后，进行测试程序的编写工作，这样就可以知道系统调用添加是否成功了，还可以运行脚本来计算自己的得分。

### 报告内容

1. linux0.11中系统调用最多能传递3个参数。若要传递大块数据给内核，可以传递这块数据的指针值。可以采用通用寄存器传递方式如edx,ebx,ecx.另外也可以采用一种IntelCPU提供的系统调用门的参数传递方式，它在进程用户态堆栈和内核堆栈自动复制传递的参数。

2. 在unistd.h中定义 #define __NR_foo 72  系统调用的总数需要修改。

  在sys.h中增加函数引用extern int sys_foo();，还要在数组中加入相应的sys_foo。
  
  在内核中实现函数sys_foo()。
  
  修改Makefile 以使foo.c可以和其他linux代码编译链接到一起。
  
  编写测试程序 ，实现系统调用。
  
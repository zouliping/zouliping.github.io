--- 
layout: post
title: 进程运行轨迹的跟踪与统计
category: 操作系统
comments: true
image:
  feature: texture-feature-02.jpg
excerpt: 在做这次实验的时候，一定要耐心一点。我当时做的时候就因为在修改的过程中出现了一点小小的失误，导致了整个实验重新来过一遍。为了避免这样的情况发生，一定要记住修改的时候，要注意修改的内容无误。
---

在做这次实验的时候，一定要耐心一点。我当时做的时候就因为在修改的过程中出现了一点小小的失误，导致了整个实验重新来过一遍。为了避免这样的情况发生，一定要记住修改的时候，要注意修改的内容无误。

本次实验包括如下三个方面的内容：

基于模板“process.c”编写多进程的样本程序，实现如下功能：
所有子进程都并行运行，每个子进程的实际运行时间一般不超过30秒；
父进程向标准输出打印所有子进程的id，并在所有子进程都退出后才退出；

在Linux0.11上实现进程运行轨迹的跟踪。基本任务是在内核中维护一个日志文件/var/process.log，把从操作系统启动到系统关机过程中所有进程的运行轨迹都记录在这一log文件中。

在修改过的0.11上运行样本程序，通过分析log文件，统计该程序建立的所有进程的等待时间、完成时间（周转时间）和运行时间，然后计算平均等待时间，平均完成时间和吞吐量。可以自己编写统计程序，也可以使用python脚本程序—— stat_log.py ——进行统计。

修改0.11进程调度的时间片，然后再运行同样的样本程序，统计同样的时间数据，和原有的情况对比，体会不同时间片带来的差异。

首先，要进行的操作是在init/main.c中的main()中添加创建日志文件/var/process.log的语句.这一步在实验指导书中很明确的解释了如何操作以及为何这么做，我在这就不多说了。把init()的一小段代码移动到main()中，放在move_to_user_mode()之后，同时加上打开log文件的代码。修改后的main()如下：

{% highlight C linenos %}
void main(void)		
/* This really IS void, no error here. */
{			
/* The startup routine assumes (well, ...) this */
/*
 * Interrupts are still disabled. Do necessary setups, then
 * enable them
 */
 	ROOT_DEV = ORIG_ROOT_DEV;
 	drive_info = DRIVE_INFO;
	memory_end = (1<<20) + (EXT_MEM_K<<10);
	memory_end &= 0xfffff000;
	if (memory_end > 16*1024*1024)
		memory_end = 16*1024*1024;
	if (memory_end > 12*1024*1024)
		buffer_memory_end = 4*1024*1024;
	else if (memory_end > 6*1024*1024)
		buffer_memory_end = 2*1024*1024;
	else
		buffer_memory_end = 1*1024*1024;
	main_memory_start = buffer_memory_end;
#ifdef RAMDISK
	main_memory_start += rd_init(main_memory_start, RAMDISK*1024);
#endif
	mem_init(main_memory_start,memory_end);
	trap_init();
	blk_dev_init();
	chr_dev_init();
	tty_init();
	time_init();
	sched_init();
	buffer_init(buffer_memory_end);
	hd_init();
	floppy_init();
	sti();
	move_to_user_mode();

        setup((void *) &drive_info);
	(void) open("/dev/tty0",O_RDWR,0);
	(void) dup(0);
	(void) dup(0);
        (void) open("/var/process.log",O_CREAT|O_TRUNC|O_WRONLY,0666);/*创建文件*/
	if (!fork()) {		/* we count on this going ok */
		init();

	}
{% endhighlight %}

接下来，我们要做的就是把给定的fprintk()函数放入到kernel/printk.c中。fprintk()函数如下：

{% highlight C linenos %}
int fprintk(int fd, const char *fmt, ...)
{
	va_list args;
	int count;
	struct file * file;
	struct m_inode * inode;

	va_start(args, fmt);
	count=vsprintf(logbuf, fmt, args);
	va_end(args);

	if (fd < 3)	/* 如果输出到stdout或stderr，直接调用sys_write即可 */
	{
		__asm__("push %%fs\n\t"
			"push %%ds\n\t"
			"pop %%fs\n\t"
			"pushl %0\n\t"
			"pushl $logbuf\n\t" /* 注意对于Windows环境来说，是_logbuf,下同 */
			"pushl %1\n\t"
			"call sys_write\n\t" /* 注意对于Windows环境来说，是_sys_write,下同 */
			"addl $8,%%esp\n\t"
			"popl %0\n\t"
			"pop %%fs"
			::"r" (count),"r" (fd):"ax","cx","dx");
	}
	else	/* 假定>=3的描述符都与文件关联。事实上，还存在很多其它情况，这里并没有考虑。*/
	{
		if (!(file=task[0]->filp[fd]))	/* 从进程0的文件描述符表中得到文件句柄 */
			return 0;
		inode=file->f_inode;

		__asm__("push %%fs\n\t"
			"push %%ds\n\t"
			"pop %%fs\n\t"
			"pushl %0\n\t"
			"pushl $logbuf\n\t"
			"pushl %1\n\t"
			"pushl %2\n\t"
			"call file_write\n\t"
			"addl $12,%%esp\n\t"
			"popl %0\n\t"
			"pop %%fs"
			::"r" (count),"r" (file),"r" (inode):"ax","cx","dx");
	}
	return count;
}
{% endhighlight %}

在kernel/sched.c文件中定义为一个全局变量long volatile jiffies=0，记录着中断发生的次数。还是在这个文件中的sched_init（）函数中，时钟中断处理函数被设置如下：

{% highlight C linenos %}
set_intr_gate(0x20,&timer_interrupt);
{% endhighlight %}

而在kernel/system_call.s文件中将timer_interrupt定义为：

{% highlight C linenos %}
timer_interrupt:
……
incljiffies #增加jiffies计数值
……
{% endhighlight %}

接下来要做的一步，是这个实验的关键点了，要寻找状态的切换点。寻找状态的切换点要涉及的文件有三个，fork.c、sched.c和exit.c。相信在仔细阅读过实验指导书的相关状态转换的例子之后，大家可以根据我下面的代码，结合自己的理解把状态转换点都找出来。
下面将sched.c贴出：

{% highlight C linenos %}
void schedule(void)
{
	int i,next,c;
	struct task_struct ** p;

/* check alarm, wake up any interruptible tasks that have got a signal */

	for(p = &LAST_TASK ; p > &FIRST_TASK ; --p)
		if (*p) {
			if ((*p)->alarm && (*p)->alarm < jiffies) {
					(*p)->signal |= (1<<(SIGALRM-1));
					(*p)->alarm = 0;
				}
			if (((*p)->signal & ~(_BLOCKABLE & (*p)->blocked)) &&
			(*p)->state==TASK_INTERRUPTIBLE)
				{
				(*p)->state=TASK_RUNNING;
				fprintk(3, "%ld\t%c\t%ld\n", (*p)->pid, 'J', jiffies);
				}

		}

/* this is the scheduler proper: */

	while (1) {
		c = -1;
		next = 0;
		i = NR_TASKS;
		p = &task[NR_TASKS];
		while (--i) {
			if (!*--p)
				continue;
			if ((*p)->state == TASK_RUNNING && (*p)->counter > c)
				c = (*p)->counter, next = i;
		}
		if (c) break;
		for(p = &LAST_TASK ; p > &FIRST_TASK ; --p)
			if (*p)
				(*p)->counter = ((*p)->counter >> 1) +
						(*p)->priority;
	}

      /*if(current->pid != task[next]->pid)//判断当前的进程是否为被选出的进程
	{
                if(current->state == TASK_RUNNING)
                {
                   fprintk(3, "%1d\t%c\t%1d\n", current->pid, 'J', jiffies);
                }
		fprintk(3, "%1d\t%c\t%1d\n", task[next]->pid, 'R', jiffies);	
	}*/
         if(current->state == TASK_RUNNING && current != task[next])           
         fprintk(3,"%ld\t%c\t%ld\n",current->pid,'J',jiffies);   
    	 if(current != task[next])           
         fprintk(3,"%ld\t%c\t%ld\n",task[next]->pid,'R',jiffies);  

	switch_to(next);
}

int sys_pause(void)
{
	current->state = TASK_INTERRUPTIBLE;
 	if(current->pid != 0 )// 当不是进程0的时候打印
        fprintk(3, "%ld\t%c\t%ld\n", current->pid, 'W', jiffies);

	schedule();
	return 0;
}

void sleep_on(struct task_struct **p)
{
	struct task_struct *tmp;

	if (!p)
		return;
	if (current == &(init_task.task))
		panic("task[0] trying to sleep");
	tmp = *p;
	*p = current;
	current->state = TASK_UNINTERRUPTIBLE;
	if(current->pid != 0 )// 当不是进程0的时候打印
        fprintk(3, "%ld\t%c\t%ld\n", current->pid, 'W', jiffies);

	schedule();
	*p=tmp;
	if (tmp)
	{
          tmp->state=TASK_RUNNING;
          fprintk(3,"%ld\t%c\t%ld\n",tmp->pid,'J',jiffies);
        }
}

void interruptible_sleep_on(struct task_struct **p)
{
	struct task_struct *tmp;

	if (!p)
		return;
	if (current == &(init_task.task))
		panic("task[0] trying to sleep");
	tmp=*p;
	*p=current;
         repeat:	current->state = TASK_INTERRUPTIBLE;
	if(current->pid != 0 )// 当不是进程0的时候打印
        fprintk(3, "%ld\t%c\t%ld\n", current->pid, 'W', jiffies);

	schedule();
	if (*p && *p != current) {

		(**p).state=TASK_RUNNING;
                fprintk(3, "%ld\t%c\t%ld\n", (**p).pid, 'J', jiffies);

		goto repeat;
	}
	*p=tmp;
	if (tmp)
	{
          tmp->state=TASK_RUNNING;
          fprintk(3, "%ld\t%c\t%ld\n", tmp->pid, 'J', jiffies);
        }
}

void wake_up(struct task_struct **p)
{
	if (p && *p) {
              if((**p).state != TASK_RUNNING)
{
                    fprintk(3, "%ld\t%c\t%ld\n", (**p).pid, 'J', jiffies);
                    (**p).state=TASK_RUNNING;
}
	}
}
{% endhighlight %}

接着是fork.c：

{% highlight C linenos %}
int copy_process(int nr,long ebp,long edi,long esi,long gs,long none,
		long ebx,long ecx,long edx,
		long fs,long es,long ds,
		long eip,long cs,long eflags,long esp,long ss)
{
	struct task_struct *p;
	int i;
	struct file *f;

	p = (struct task_struct *) get_free_page();
	if (!p)
		return -EAGAIN;
	task[nr] = p;
	*p = *current;	/* NOTE! this doesn't copy the supervisor stack */
	p->state = TASK_UNINTERRUPTIBLE;
	p->pid = last_pid;
	p->father = current->pid;
	p->counter = p->priority;
	p->signal = 0;
	p->alarm = 0;
	p->leader = 0;		/* process leadership doesn't inherit */
	p->utime = p->stime = 0;
	p->cutime = p->cstime = 0;
	p->start_time = jiffies;
       
	p->tss.back_link = 0;
	p->tss.esp0 = PAGE_SIZE + (long) p;
	p->tss.ss0 = 0x10;
	p->tss.eip = eip;
	p->tss.eflags = eflags;
	p->tss.eax = 0;
	p->tss.ecx = ecx;
	p->tss.edx = edx;
	p->tss.ebx = ebx;
	p->tss.esp = esp;
	p->tss.ebp = ebp;
	p->tss.esi = esi;
	p->tss.edi = edi;
	p->tss.es = es & 0xffff;
	p->tss.cs = cs & 0xffff;
	p->tss.ss = ss & 0xffff;
	p->tss.ds = ds & 0xffff;
	p->tss.fs = fs & 0xffff;
	p->tss.gs = gs & 0xffff;
	p->tss.ldt = _LDT(nr);
	p->tss.trace_bitmap = 0x80000000;
	if (last_task_used_math == current)
		__asm__("clts ; fnsave %0"::"m" (p->tss.i387));
	if (copy_mem(nr,p)) {
		task[nr] = NULL;
		free_page((long) p);
		return -EAGAIN;
	}
	for (i=0; i<NR_OPEN;i++)
		if ((f=p->filp[i]))
			f->f_count++;
	if (current->pwd)
		current->pwd->i_count++;
	if (current->root)
		current->root->i_count++;
	if (current->executable)
		current->executable->i_count++;
	set_tss_desc(gdt+(nr<<1)+FIRST_TSS_ENTRY,&(p->tss));
	set_ldt_desc(gdt+(nr<<1)+FIRST_LDT_ENTRY,&(p->ldt));
       fprintk(3, "%1d\t%c\t%1d\n", last_pid, 'N', jiffies);
	p->state = TASK_RUNNING;	/* do this last, just in case */
        fprintk(3, "%1d\t%c\t%1d\n", last_pid, 'J', jiffies);
	return last_pid;
}

int find_empty_process(void)
{
	int i;

	repeat:
		if ((++last_pid)<0) last_pid=1;
		for(i=0 ; i<NR_TASKS ; i++)
			if (task[i] && task[i]->pid == last_pid) goto repeat;
	for(i=1 ; i<NR_TASKS ; i++)
		if (!task[i])
			return i;
	return -EAGAIN;
}
{% endhighlight %}

再来是exit.c：

{% highlight C linenos %}
int do_exit(long code)
{
	int i;
	free_page_tables(get_base(current->ldt[1]),get_limit(0x0f));
	free_page_tables(get_base(current->ldt[2]),get_limit(0x17));
	for (i=0 ; i<NR_TASKS ; i++)
		if (task[i] && task[i]->father == current->pid) {
			task[i]->father = 1;
			if (task[i]->state == TASK_ZOMBIE)
                        //fprintk(3, "%1d\t%c\t%1d\n", task[i]->pid, 'E', jiffies);
				/* assumption task[1] is always init */
				(void) send_sig(SIGCHLD, task[1], 1);
		}
	for (i=0 ; i<NR_OPEN ; i++)
		if (current->filp[i])
			sys_close(i);
	iput(current->pwd);
	current->pwd=NULL;
	iput(current->root);
	current->root=NULL;
	iput(current->executable);
	current->executable=NULL;
	if (current->leader && current->tty >= 0)
		tty_table[current->tty].pgrp = 0;
	if (last_task_used_math == current)
		last_task_used_math = NULL;
	if (current->leader)
		kill_session();
	current->state = TASK_ZOMBIE;
        //fprintk(3, "%1d\t%c\t%1d\n", current->pid, 'E', jiffies);
	current->exit_code = code;
	tell_father(current->father);
	schedule();
	return (-1);	/* just to suppress warnings */
}

int sys_exit(int error_code)
{
	return do_exit((error_code&0xff)<<8);
}

int sys_waitpid(pid_t pid,unsigned long * stat_addr, int options)
{
	int flag, code;
	struct task_struct ** p;

	verify_area(stat_addr,4);
repeat:
	flag=0;
	for(p = &LAST_TASK ; p > &FIRST_TASK ; --p) {
		if (!*p || *p == current)
			continue;
		if ((*p)->father != current->pid)
			continue;
		if (pid>0) {
			if ((*p)->pid != pid)
				continue;
		} else if (!pid) {
			if ((*p)->pgrp != current->pgrp)
				continue;
		} else if (pid != -1) {
			if ((*p)->pgrp != -pid)
				continue;
		}
		switch ((*p)->state) {
			case TASK_STOPPED:
				if (!(options & WUNTRACED))
					continue;
				put_fs_long(0x7f,stat_addr);
				return (*p)->pid;
			case TASK_ZOMBIE:
				current->cutime += (*p)->utime;
				current->cstime += (*p)->stime;
				flag = (*p)->pid;
                                //fprintk(3, "%1d\t%c\t%1d\n", (*p)->pid, 'E', jiffies);
				code = (*p)->exit_code;
				fprintk(3, "%ld\t%c\t%ld\n", flag, 'E', jiffies);
				release(*p);
				put_fs_long(code,stat_addr);
				return flag;
                                
			default:
				flag=1;
				continue;
		}
	}
	if (flag) {
		if (options & WNOHANG)
			return 0;
		current->state=TASK_INTERRUPTIBLE;
 		if(current->pid != 0 )// 当不是进程0的时候打印
                fprintk(3, "%1d\t%c\t%1d\n", current->pid, 'W', jiffies);
		schedule();
		if (!(current->signal &= ~(1<<(SIGCHLD-1))))
			goto repeat;
		else
			return -EINTR;
	}
	return -ECHILD;
}

{% endhighlight %}

下面这段话一定要认真阅读：

Linux0.11支持四种进程状态的转移：就绪到运行、运行到就绪、运行到睡眠和睡眠到就绪，此外还有新建和退出两种情况。其中就绪与运行间的状态转移是通过schedule()（它亦是调度算法所在）完成的；运行到睡眠依靠的是sleep_on()和interruptible_sleep_on()，还有进程主动睡觉的系统调用sys_pause()和sys_waitpid()；睡眠到就绪的转移依靠的是wake_up()。所以只要在这些函数的适当位置插入适当的处理语句就能完成进程运行轨迹的全面跟踪了。

编写样本程序，样本程序的作用就是生成几个进程，把我们之前对0.11的修改，通过这些进程的调用来写到log文件中去。这个样本程序的编写很简单就不多说了。

下面是process.c：

{% highlight C linenos %}
#include <stdio.h>
#include <unistd.h>
#include <time.h>
#include <sys/times.h>
#include <sys/types.h>
#include <sys/wait.h>

#define HZ	100

void cpuio_bound(int last, int cpu_time, int io_time);

int main(int argc, char * argv[])
{
   pid_t pid;
   int i = 0;
            
   for(i=0;i<3;i++)
   {
      pid = fork();
      if(pid<0)
      {
         printf("error in fork!");
      }
      else if(pid==0)
      {
         printf("process id is %d\n ",getpid());
         cpuio_bound(10,i,10-i);  
         return;
      }
   }
   wait(NULL);
   wait(NULL);
   wait(NULL);
   return 0;
}

/*
 * 此函数按照参数占用CPU和I/O时间
 * last: 函数实际占用CPU和I/O的总时间，不含在就绪队列中的时间，>=0是必须的
 * cpu_time: 一次连续占用CPU的时间，>=0是必须的
 * io_time: 一次I/O消耗的时间，>=0是必须的
 * 如果last > cpu_time + io_time，则往复多次占用CPU和I/O
 * 所有时间的单位为秒
 */
void cpuio_bound(int last, int cpu_time, int io_time)
{
	struct tms start_time, current_time;
	clock_t utime, stime;
	int sleep_time;

	while (last > 0)
	{
		/* CPU Burst */
		times(&start_time);
		/* 其实只有t.tms_utime才是真正的CPU时间。但我们是在模拟一个
		 * 只在用户状态运行的CPU大户，就像“for(;;);”。所以把t.tms_stime
		 * 加上很合理。*/
		do
		{
			times(&current_time);
			utime = current_time.tms_utime - start_time.tms_utime;
			stime = current_time.tms_stime - start_time.tms_stime;
		} while ( ( (utime + stime) / HZ )  < cpu_time );
		last -= cpu_time;

		if (last <= 0 )
			break;

		/* IO Burst */
		/* 用sleep(1)模拟1秒钟的I/O操作 */
		sleep_time=0;
		while (sleep_time < io_time)
		{
			sleep(1);
			sleep_time++;
		}
		last -= sleep_time;
	}
}
{% endhighlight %}

这样就把这个实验的大体部分做完了。通过查看log文件，可以看到如下面的结果类似的东西。

{% highlight C linenos %}
1	N	48
1	J	48
0	J	48
1	R	48
2	N	49
2	J	49
1	W	49
2	R	49
3	N	64
3	J	64
2	J	64
3	R	64
3	W	68
2	R	68
4	N	79
4	J	79
2	W	79
4	R	79
5	N	84
5	J	84
4	W	84
5	R	84
4	J	140
4	R	140
5	E	140
6	N	141
6	J	141
4	W	142
6	R	142
4	J	238
4	R	238
6	E	238
7	N	239
7	J	239
4	W	239
7	R	239
4	J	263
4	R	263
7	E	263
8	N	265
8	J	265
4	W	266
{% endhighlight %}

至于最后用脚本运行的结果是什么，现在有点记不清了，实在是不好意思啦。

对于时间片的修改，是修改INIT_TASK中的priority的值，可以实现时间片的大小调整。变化不明显，感觉上是因为输入输出设备的操作占用的时间影响的。

报告：

1.从程序设计者的角度来看，单进程和多进程的区别有以下几点：

（1）单进程是顺序执行的，是从上到下执行的，程序设计者需要做的是控制好程序的顺序。而多进程是同时执行的，进程可以说是并行执行的，程序设计者需要做的是合理安排每个进程的流程。

（2）单进程的数据是同步的，不需要考虑数据的共享，只要后面的程序使用前面的结果即可。而多进程就需要考虑数据的共享问题，同时对某个数据进行操作很有可能会出错。

（3）单进程的复杂度较小，而多进程则较为复杂。

（4）单进程的用途较为单一，而多进程的用途广泛。


2.根据指导书上给的提示，我们可以知道，如果在实验假定没有人调用过nice系统调用，时间片的初值就是进程0的priority，即宏INIT_TASK中定义的：

#define INIT_TASK 

 { 0,15,15, //分别对应state;counter;和priority;
 
想要修改时间片，只要修改这个宏定义的第三个值即可。

在进行一系列的数据分析之后从log文件的统计结果中可以发现：在一定的范围内，平均等待时间，平均完成时间的变化随着时间片的增大而减小。这是因为在时间片小的情况下，cpu将时间耗费在调度切换上，所以平均等待时间增加。而超过一定的范围之后，这些参数将不再有明显的变化，这是因为在这种情况下，RR轮转调度就变成了FCFS先来先服务了。随着时间片的修改，吞吐量始终没有明显的变化，这是因为在单位时间内，系统所能完成的进程数量是不会变的。
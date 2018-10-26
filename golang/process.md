# 1. 进程互斥与同步

- [1. 进程互斥与同步](#1-%E8%BF%9B%E7%A8%8B%E4%BA%92%E6%96%A5%E4%B8%8E%E5%90%8C%E6%AD%A5)
	- [1.1. PV操作](#11-pv%E6%93%8D%E4%BD%9C)
		- [1.1.1. 基本原理](#111-%E5%9F%BA%E6%9C%AC%E5%8E%9F%E7%90%86)
		- [1.1.2. PV实现读写锁。](#112-pv%E5%AE%9E%E7%8E%B0%E8%AF%BB%E5%86%99%E9%94%81)
	- [1.2. 管程](#12-%E7%AE%A1%E7%A8%8B)
		- [1.2.1. 基本原理](#121-%E5%9F%BA%E6%9C%AC%E5%8E%9F%E7%90%86)
		- [1.2.2. 实现读写锁管程](#122-%E5%AE%9E%E7%8E%B0%E8%AF%BB%E5%86%99%E9%94%81%E7%AE%A1%E7%A8%8B)

## 1.1. PV操作

### 1.1.1. 基本原理
信号量: 信号量s用于表示资源的个数.当s大于零时,表示资源可用;当s=0时,表示资源分配完;当s<0时;表示有abs(s)个进程在等待

P操作: 原语操作,在执行中,不能被中断.内部执行如下:
```
Typedef semaphore int

Void P(semaphore s){
	s--;
	if (s<0)
	    wait(s) //自我休眠，进入等待态,添加到等待信号量s的队列中
}
```

P操作: 原语操作,在执行中,不能被中断.内部执行如下:
```
void V(Semaphore s){
	s++;
	if(s<=0)
		wakeup(s) //唤醒一个在信号量s上等待的进程,
	endif
}
```

**提示:单CPU情况,原语操作就是在进入方法时屏蔽中断,退出时打开中断.**

### 1.1.2. PV实现读写锁。
要求：可同时读，但是读与写互斥。

当信号量=1时,可以使用PV操作产生临界区,达到同步的目的.
例如:
```
s:=1
// 申请进入临界区.
//如果此时s=0,执行此操作之后s=-1,当前进程就会进入等待状态.
//如果此时s=1,则申请成功,进入临界区执行
P(s)

{执行临界代码}

//退出临界区,并且尝试唤醒在条件s上进入等待状态的一个进程
V(s)
```

**代码实现：**
```
readThreadCount := 0
writeMutex: =1 // 写信号量
readMutex :=1 //读信号量

process read(){
	//读之前要拿到读信号量，对readThreadCount加一
	P(readMutex)
	readThreadCount=readThreadCount+1
	//如果读的线程数等于1,就申请写锁，如果此时，已经有其它线程正在写操作（writeMutex<=0），当前线程就转为就绪状态。
	if(readThreadCount==1){
		P(writeMutex)
	}
	V(readMutex)
	//执行读
	read_function()
	
	//读完成之后要对readThreadCount减1,操作之后，如果readThreadCount=0,则要释放writeMutex
	P(readMutex)
	readThreadCount=readThreadCount-1
	if(readThreadCount==0){
		V(writeMutex)
	}
	V(readMutex)
}

process write(){
	P(writeMutex)
	// 执行写操作
	write_function()

	V(writeMutex)
}
```
**提示:信号量和条件变量之所以可以在多个进程中使用,是因为信号量存放在共享存储器中.但是共享存储器只能存储少量的数据.**

## 1.2. 管程

### 1.2.1. 基本原理

PV操作容易出错,比如忘记写V就会出问题.管程就是将分散成各个进程中的临界区集中起来进行统一控制和管理.

管程有三个特性:
- 1. 互斥性:任何时刻最多一个进程在管程中活动.
- 2. 安全性:管程中的局部变量和函数只能管程访问.
- 3. 共享性:管程中申明的特定函数可以被外部进程访问.

**管程的一般结构:**
```
struct  Monitor {￼
	condition  条件变量列表;
	define  函数或方法;
	use   函数或方法；
	void  函数名() {}￼
	void  函数名() {}￼
	void init(){￼
		对管程中的局部变量进行初始化；
	}￼
}；

```

Hasnsen方法实现管程:

**用到了4个有原语:**

- 1. check() 检查管程是否可用,如果可用,就进入,同时关闭管程.如果不可用,进入入口队列Q_en,进入等待.
- 2. release() 释放管程.如果Q_en不为空,唤醒一个管程.否则,放开管程.
- 3. wait(s)  进入管程执行时如果条件不满足,就让自己进入阻塞态,进入等待队列Q_s,开放管程.
- 4. signal(s) 唤醒管程等待队列Q_s中的进程,唤醒由系统策略及算法决定.如果Q_s空,不影响.


### 1.2.2. 实现读写锁管程

**管程代码:**
```
stuct Monitor{
	condition R,W; //条件变量
	int r_count,w_count; // 读的进程数,写的进程数
	use wait,signal,check,release; 
	init(){
		R=0;
		W=0;
		r_count=0;
		w_coun=0;
	}

	void StartRead(){
		check();
		//如果有进程在进程写操作,就等待. 读写互斥.
		if(w_count>0){
			wait(R);
		}
		r_count++;
		//读与读进程可以并行,尝试唤醒一个在R队列等待的进程,但此时其它进程还不能进入.
		signal(R);
		release();
	}

	void EndRead(){
		check();
		r_count--;
		if(w_count==0){
			signal(W);
		}
		release();
	}

	void StartWrite(){
		check();
		w_count++;
		//保证只有一个写操作
		if(r_count>0||w_count>1){
			wait(W);
		}
		release();
	}

	void EndWrite(){
		check();
		r_count--;
		if(w_count>0){
			signal(W);
		}else{
			signal(R);
		}
		release();
	}
}

```

**读写进程代码:**

```
Monitor rw;

process read(){
	rw.StartRead()
	读....
	rw.EndRead()
}

process write(){
	rw.StartWrite()
	写....
	rw.EndWrite()
}
```


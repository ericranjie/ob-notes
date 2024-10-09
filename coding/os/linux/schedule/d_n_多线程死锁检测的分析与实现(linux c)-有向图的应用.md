Linux开发架构之路
_2024年03月27日 20:06_ _湖南_

# 一、提出问题

在日常的软件开发中，多线程是不可避免的，使用多线程中的一大问题就是线程对锁的不合理使用造成的死锁，死锁一旦发生，将导致多线程程序响应时间长，吞吐量下降甚至宕机崩溃，那么如何检测出一个多线程程序中是否存在死锁呢？在提出解决方案之前，先对死锁产生的原因以及产生的现象做一个分析。最后在用有向环来检测多线程中是否存在死锁的问题。

## 二、死锁存在的条件

所谓死锁，是指多个进程在运行过程中因争夺资源而造成的一种僵局，当进程处于这种僵持状态时，若无外力作用，它们都将无法再向前推进。

我们举个例子来描述，如果此时有一个线程A，按照先锁a再获得锁b的的顺序获得锁，而在此同时又有另外一个线程B，按照先锁b再锁a的顺序获得锁。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/8pECVbqIO0xZslZTn668Q7iaMZSibJMLMIjBF3SuRhzzBicqwXSMSqug5WVcl4yjSG1nVq3GsFAickJneNiaMN13jWA/640?wx_fmt=png&from=appmsg&wxfrom=13)

在来看看4个线程线程的情况，线程A想获取线程B的锁，线程B想获取线程C的锁，线程C想获取线程D的锁，线程D想获取线程A 的锁，在线程之间构建了一个资源获取环。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/8pECVbqIO0xZslZTn668Q7iaMZSibJMLMIiceLNQor88A8qMqregtZhBicgT0bqiaGtUSdTnQWU6UxfMIm1rP6Gw64w/640?wx_fmt=png&from=appmsg&wxfrom=13)

注意：后面的例子都是以4个线程的例子来说明的。

## 三、死锁的产生原因

（1）竞争资源

系统中的资源可以分为两类：

可剥夺资源，是指某进程在获得这类资源后，该资源可以再被其他进程或系统剥夺，CPU和主存均属于可剥夺性资源；

另一类资源是不可剥夺资源，当系统把这类资源分配给某进程后，再不能强行收回，只能在进程用完后自行释放，如磁带机、打印机等。

产生死锁中的竞争资源之一指的是竞争不可剥夺资源（例如：系统中只有一台打印机，可供进程P1使用，假定P1已占用了打印机，若P2继续要求打印机打印将阻塞）

产生死锁中的竞争资源另外一种资源指的是竞争临时资源（临时资源包括硬件中断、信号、消息、缓冲区内的消息等），通常消息通信顺序进行不当，则会产生死锁。

（2）进程间推进顺序非法

若P1保持了资源R1,P2保持了资源R2，系统处于不安全状态，因为这两个进程再向前推进，便可能发生死锁。

例如，当P1运行到P1：Request（R2）时，将因R2已被P2占用而阻塞；当P2运行到P2：Request（R1）时，也将因R1已被P1占用而阻塞，于是发生进程死锁。

# 四、死锁必要条件

1.互斥条件：进程要求对所分配的资源进行排它性控制，即在一段时间内某资源仅为一进程所占用。
2.请求和保持条件：当进程因请求资源而阻塞时，对已获得的资源保持不放。
3.不剥夺条件：进程已获得的资源在未使用完之前，不能剥夺，只能在使用完时由自己释放。
4.环路等待条件：在发生死锁时，必然存在一个进程–资源的环形链。

# 五、死锁问题分析

从死锁存在的条件的图中2个线程右边的图和4个线程使用锁（互斥资源）图的来看，发生死锁之后，就构成了线程之间的一个有向环形图，因此，我们很自然的想到，死锁的问题就转换为有向环（图）问题，只要线程之间存在环形链，那么就产生了死锁问题。产生死锁的测试代码见源代码部分。

# 六、环形链（死锁）的检测

如果对检测死锁问题就是对有向环的检测有一个了解之后，那么如果当死锁时肯定是一个有向环。那么我们如何构造这个有向环并且检测出这个有向环图呢？

一般在多线程程序中，我们会对某一段代码进行加锁，防止其他线程访问，线程执行完该段代码之后会释放锁操作；之所以造成死锁，主要原因是因为某个进程需要对某个锁进行lock操作，然而该锁已被其他线程lock了，而且当前线程还不知道这个锁当前被哪个线程lock了，更为重要的是其他线程又需要对该线程的某个锁进行lock操作，同样的道理，其他的线程也不知道其线程的将要lock的锁已被哪个线程lock了（也就是说只有线程自己知道自己lock了哪些锁）。

既然线程只能知道自己拥有了哪些锁，那么想解决线程死锁还不简单吗？直接将自己拥有的锁释放，然后让其他线程执行不就行了？如果有这个想法的朋友，跟我犯了同样的错，锁的目的就是告诉我要执行某段逻辑了，其他线程不能进入，从而保证了线程之间的同步性。从另一个方面来说，如果多个线程发现自己需要加的锁正在被其他线程lock,然后unlock自己占有的锁，假设所有的线程都unlock了所有锁，线程什么时候去对自己需要的锁进行加锁？特殊的情况就是所有线程又去争夺这些锁，这不是同样会造成死锁吗？所以发现自己占有锁被别的线程lock时，去释放自己的锁的思路是行不通的。出现了死锁-其实是业务逻辑出现了问题，需要结合业务逻辑的代码去修改，而不仅仅是单方面从解决死锁的方面出发。另外，本文的目的也是发现自己的代码中是否存在死锁，没有提供解决死锁的方法。

由于系统没有提供死锁检测的机制，我们需要在程序的运行期间时刻监控线程与锁之间的关系，也就是维护有向图的状态，即通过线程在加锁前、加锁后以及释放锁之后的3个阶段来维护有向图的状态（通过有向图的状态我们就可以判断是否有死锁）

（1）加锁之前：当前线程需要加的锁是否被其他线程占用，如果是，就让当前线程指向占有锁的线程。

举例：线程A需要对线程B已经lock的锁进行lock操作的话，需要在线程A到线程B之间加一个边，线程A指向线程B。

（2）加锁之后：需要将锁和线程建立起一对一的关系（说明该锁目前被哪个线程使用）,存在2种情况：

a.该锁之前没有被其他线程lock过，直接建立起线程id和锁id的关系-这种情况很明了，就是使用一个结构体变量来表示对应的线程id和锁id。

b.该锁之前被其他线程lock过，但是后来被该线程unlock了，这时候需要判断当前线程和该线程之间是否存在边，如果存在，需要先删除边，然后再将锁id和当前线程建立起一对一关系。

举例：对于b来说，按照步骤（1）的例子来说，如果线程A与线程B之间有线程A指向线程B的边，并且B在lock锁之后又unlock了该锁，这时候A就能够对该锁lock了，但是lock之前需要将A到B的边进行删除，因为该锁已经从B转移到了A。

（3）释放锁之后：查询锁id的下标，然后将其锁id和线程id设置为0（清除步骤二建立的对应关系）。

具体细节会在代码里提现。

# 七、Hook

前面说过造成死锁的原因是因为当前线程只知道自己lock的锁，而通过环形链检测的3步骤之后，就知道了目前哪些锁当前正在被哪些线程lock了，哪些锁没有被lock。从而使问题得以解决。

既然在每次加锁之前和解锁之后都要完成这些操作，是不是可以考虑到将这3个操作融入到加锁和解锁的函数API中，但是就常规情况而言，C语言没有C++类的函数重写功能，但是对于早期编译器而言，C的语法是支持自定义和系统API一样的函数名(虽然我们可以定义的函数和系统函数重名而不报错，那么我们又如何使用系统的API呢？)，这时候我们可以自定义pthread_mutex_lock()接口-和系统api同名，但是在调用系统api加锁之前需要进行”加锁之前的操作”以及在加锁之后的”加锁之后”操作，同时定义pthread_mutex_unlock()接口需要实现解锁操作和”释放锁之后”的操作。这样用户依然可以像调用系统API一样，去进行加锁和解锁操作。然而比较新的编译器会将这种视为”重定义”的错误。然而Hook帮我们实现了这个问题，关于hook的原理这里不做分析，有兴趣的朋友可以自行百度，Hook的原理就是使用函数指针的思想，找到我们想要指向的系统API的入口，然后在某个时候进行调用即可，Hook是使用系统的dlsym API接口来实现的，下面是使用实例。

```cpp
typedef int (*pthread_mutex_lock_t)(pthread_mutex_t *mutex);  
pthread_mutex_lock_t pthread_mutex_lock_f;  
typedef int (*pthread_mutex_unlock_t)(pthread_mutex_t *mutex);  
pthread_mutex_unlock_t pthread_mutex_unlock_f;  

//在函数入口处进行初始化  
static int init_hook() {   
   //锁住系统函数 pthread_mutex_lock  
   pthread_mutex_lock_f = dlsym(RTLD_NEXT, "pthread_mutex_lock");  
   pthread_mutex_unlock_f = dlsym(RTLD_NEXT, "pthread_mutex_unlock");  
}

int pthread_mutex_lock(pthread_mutex_t *mutex) {  
    pthread_t selfid = pthread_self(); //  
    lock_before(selfid, (uint64)mutex);//加锁前操作  
    pthread_mutex_lock_f(mutex);//调用系统的pthread_mutex_lock  
    lock_after(selfid, (uint64)mutex);//加锁后操作  
}

int pthread_mutex_unlock(pthread_mutex_t *mutex) {  
    pthread_t selfid = pthread_self();  
    pthread_mutex_unlock_f(mutex);//调用系统的pthread_mutex_unlock  
    unlock_after(selfid, (uint64)mutex);//解锁后操作  
}
```

以上代码就是使用hook的例子，在函数入口处调用init_hook().，init_hook()里面为什么会hook()系统的函数，而不是自定义的函数，这个在这篇博客中”hook function注意事项”处说明：hook介绍,然后就可以像你平时使用pthread_mutex_lock和pthread_mutex_unlock使用这2个函数了。

如果对函数指针熟悉，上面的代码很容易明白上面意思。其中lock_before、lock_after、unlock_after就是前面说的环检测的3步骤，后面源代码部分有实现过程。

# 八、死锁中涉及图的结构分析

关于图的基本概念、遍历等在本文中就不细说了，这里结合源代码来对其进行说明。

## 结点数据信息

```cpp
enum Type {PROCESS, RESOURCE};//进程，资源  
//结点数据  
struct source_type {  
    uint64 id;//线程id  
    enum Type type;//结点类型（没有使用-不用考虑这个类型信息）  
    uint64 lock_id;//mutex id  
    //当前 mutex id 正在被线程lock的标志（我可能理解的有问题）  
    int degress;  
};
```

图由顶点（结点）和边组成，在死锁中结点代表了线程和锁的对应关系，包括线程id，mutex_id等。

## 图的邻接表表示

图的邻接表所有结点使用数组表示，而结点的边是使用链表来表示。

```cpp
// 结点  
struct vertex {  
    struct source_type s;  
    struct vertex *next;//结点与结点之间的边的链表表示  
};  
// 图-邻接表表示  
struct task_graph {  
    struct vertex list[MAX];//结点列表  
    int num;//结点数量  
    //所有线程对应的锁的列表（一个线程可能有多个锁，那么就是一个线程就有多个记录，）  
    struct source_type locklist[MAX];  
    int lockidx;//锁 id下标  
    pthread_mutex_t mutex;  
};
```

## 深度遍历（DFS）

从某结点开始遍历，从该节点某边进行遍历，一直遍历到最后一个结点，然后返回继续遍历该节点的其他边。对每个结点都进行递归操作。

## 有向环的检测

在深度遍历（DFS）遍历的过程中，如果发现某个结点的访问标志已经为1，那么就存在环了。

## 

九 实现

### 

C语言源代码

#define \_GNU_SOURCE\
#include \<dlfcn.h>

#include \<stdio.h>\
#include \<pthread.h>\
#include \<unistd.h>

#include \<stdlib.h>\
#include \<stdint.h>

#include \<unistd.h>

#define THREAD_NUM      10

typedef unsigned long int uint64;

typedef int (\*pthread_mutex_lock_t)(pthread_mutex_t \*mutex);

pthread_mutex_lock_t pthread_mutex_lock_f;

typedef int (\*pthread_mutex_unlock_t)(pthread_mutex_t \*mutex);

pthread_mutex_unlock_t pthread_mutex_unlock_f;

#if 1 // graph

#define MAX		100

enum Type {PROCESS, RESOURCE};//进程，资源

//结点数据\
struct source_type {\
uint64 id;//线程id\
enum Type type;//结点类型（没有使用-不用考虑这个类型信息）\
uint64 lock_id;//mutex id\
int degress;//当前 mutex id 正在被线程lock的标志（我可能理解的有问题）\
};

//结点\
struct vertex {\
struct source_type s;\
struct vertex \*next;//结点与结点之间的边的链表表示\
};

//图-邻接表表示\
struct task_graph {\
struct vertex list\[MAX\];//结点列表\
int num;//结点数量\
struct source_type locklist\[MAX\];//所有线程对应的锁的列表（一个线程可能有多个锁，那么就是一个线程就有多个记录，）\
int lockidx;//锁 id下标\
pthread_mutex_t mutex;\
};

struct task_graph \*tg = NULL;//图的表示\
int path\[MAX+1\];//访问的线程id路径\
int visited\[MAX\]; //标识结点是否被访问的标志\
int k = 0;//用来存放路径的序号\
int deadlock = 0; //死锁标志

//创建结点\
struct vertex \*create_vertex(struct source_type type) {

```
struct vertex *tex = (struct vertex *)malloc(sizeof(struct vertex ));  

tex->s = type;  
tex->next = NULL;//  

return tex;  
```

}

//搜索结点，存在返回下标，否则返回-1\
int search_vertex(struct source_type type) {

```
int i = 0;  

for (i = 0;i < tg->num;i ++) {  

	if (tg->list[i].s.type == type.type && tg->list[i].s.id == type.id) {//结点类型（不考虑）和结点id共同决定一个顶点  
		return i;  
	}  
}  

return -1;  
```

}

//增加结点\
void add_vertex(struct source_type type) {

```
if (search_vertex(type) == -1) {//不存在结点，则添加  

	tg->list[tg->num].s = type;  
	tg->list[tg->num].next = NULL;  
	tg->num ++;  

}  
```

}

//增加边\
int add_edge(struct source_type from, struct source_type to) {

```
add_vertex(from);  
add_vertex(to);  

struct vertex *v = &(tg->list[search_vertex(from)]);//找到数组下标位置  

while (v->next != NULL) {//将边插入到当前结点的所有出边之后  
	v = v->next;  
}  

v->next = create_vertex(to);  
```

}

//验证结点i和结点j之间是否存在边\
int verify_edge(struct source_type i, struct source_type j) {

```
if (tg->num == 0) return 0;  

int idx = search_vertex(i);  
if (idx == -1) {  
	return 0;  
}  

struct vertex *v = &(tg->list[idx]);  
//遍历i的所有出边，直到找到为止  
while (v != NULL) {  

	if (v->s.id == j.id) return 1;  

	v = v->next;  
	  
}  

return 0;  
```

}

//删除边 from-起点 to-终点\
int remove_edge(struct source_type from, struct source_type to) {

```
int idxi = search_vertex(from);  
int idxj = search_vertex(to);  

if (idxi != -1 && idxj != -1) {  

	struct vertex *v = &tg->list[idxi];  
	struct vertex *remove;  

	while (v->next != NULL) {  
        //找到要删除的边  
		if (v->next->s.id == to.id) {  

			remove = v->next;  
			v->next = v->next->next;  

			free(remove);  
			break;  
		}  
		v = v->next;  
	}  

}  
```

}

void print_deadlock(void) {

```
int i = 0;  

printf("deadlock : ");  
for (i = 0;i < k-1;i ++) {  

	printf("%ld --> ", tg->list[path[i]].s.id);  

}  

printf("%ld\n", tg->list[path[i]].s.id);  
```

}

//深度优先搜索-idx:结点序号（要遍历的第一个结点）\
int DFS(int idx) {

```
struct vertex *ver = &tg->list[idx];  
if (visited[idx] == 1) { //只有该结点是遍历过，那么就是存在环（死锁）  
	path[k++] = idx;  
	print_deadlock();  
	deadlock = 1;  
	return 0;  
}  
visited[idx] = 1;//遍历过后就将该结点标志设置为1  
path[k++] = idx;  
while (ver->next != NULL) {  
	DFS(search_vertex(ver->next->s));//深度遍历下一个结点  
	k --;//idx的某一条边的所有已经遍历完成，路径有回退的操作.这里有点不太好理解，可以自己画一个很简单的有向图进行分析，  
	ver = ver->next;//指向该节点的下一条边  
}  
return 1;  
```

}

//从idx结点开始遍历，看是否存在环\
int search_for_cycle(int idx) {

```
struct vertex *ver = &tg->list[idx];  
visited[idx] = 1;  
k = 0;  
path[k++] = idx;//记下idx序号结点的路径编码  

while (ver->next != NULL) {  
	int i = 0;  
	for (i = 0;i < tg->num;i ++) {//除该节点之外的所有结点将其标志设置设置为0  
		if (i == idx) continue  
		visited[i] = 0;  
	}  

	for (i = 1;i <= MAX;i ++) {  
		path[i] = -1;//路径编码设置为-1，代表除起始节点的所有结点还不存在路径即：path[max]={idx,-1.-1.-1.-1......}  
	}  
	k = 1;  
	DFS(search_vertex(ver->next->s));//从该节点的某个边的节点开始遍历  
	ver = ver->next;  
}  
```

}

#if 0\
int main() {

```
tg = (struct task_graph*)malloc(sizeof(struct task_graph));  
tg->num = 0;  

struct source_type v1;  
v1.id = 1;  
v1.type = PROCESS;  
add_vertex(v1);  

struct source_type v2;  
v2.id = 2;  
v2.type = PROCESS;  
add_vertex(v2);  

struct source_type v3;  
v3.id = 3;  
v3.type = PROCESS;  
add_vertex(v3);  

struct source_type v4;  
v4.id = 4;  
v4.type = PROCESS;  
add_vertex(v4);  

  
struct source_type v5;  
v5.id = 5;  
v5.type = PROCESS;  
add_vertex(v5);  


add_edge(v1, v2);  
add_edge(v2, v3);  
add_edge(v3, v4);  
add_edge(v4, v5);  
add_edge(v3, v1);  
  
search_for_cycle(search_vertex(v1));  
```

}\
#endif

#endif

//检测是否有死锁\
void check_dead_lock(void) {

```
int i = 0;  

deadlock = 0;  
for (i = 0;i < tg->num;i ++) {//对所有结点进行遍历-看结点是否存在环  
	if (deadlock == 1) break;  
	search_for_cycle(i);  
}  

if (deadlock == 0) {  
	printf("no deadlock\n");  
}  
```

}

//检测是否有死锁的线程\
static void \*thread_routine(void \*args) {

```
while (1) {  
	sleep(5);  
	check_dead_lock();  

}  
```

}

void start_check(void) {

```
tg = (struct task_graph*)malloc(sizeof(struct task_graph));  
tg->num = 0;  
tg->lockidx = 0;  
pthread_t tid;  
pthread_create(&tid, NULL, thread_routine, NULL);  
```

}

#if 1\
//lock:mutex id\
//判断mutex id是否在所有的锁列表中\
int search_lock(uint64 lock) {

```
int i = 0;  
  
for (i = 0;i < tg->lockidx;i ++) {  
	  
	if (tg->locklist[i].lock_id == lock) {  
		return i;  
	}  
}  

return -1;  
```

}

//lock:mutex id\
//\
int search_empty_lock(uint64 lock) {

```
int i = 0;  
  
for (i = 0;i < tg->lockidx;i ++) {  
	  
	if (tg->locklist[i].lock_id == 0) {  
		return i;  
	}  
}  
return tg->lockidx;  
```

}

#endif

//原子操作-++操作（这里涉及到底层码-没必要细细研究）\
int inc(int \*value, int add) {

```
int old;       
__asm__ volatile(  
	"lock;xaddl %2, %1;"  
	: "=a"(old)  
	: "m"(*value), "a" (add)  
	: "cc", "memory"  
);  
  
return old;  
```

}

void print_locklist(void) {\
int i = 0;\
printf("print_locklist: \\n");\
printf("---------------------\\n");\
for (i = 0;i \< tg->lockidx;i ++) {\
printf("threadid : %ld, lockid: %ld\\n", tg->locklist\[i\].id, tg->locklist\[i\].lock_id);\
}\
printf("---------------------\\n\\n\\n");\
}

//thread_id:线程id\
//lockaddr:线程即将要加的mutex id\
void lock_before(uint64 thread_id, uint64 lockaddr) {

```
int idx = 0;  
// list<threadid, toThreadid>  
for(idx = 0;idx < tg->lockidx;idx ++) {  
	if ((tg->locklist[idx].lock_id == lockaddr)) {  

		struct source_type from;  
		from.id = thread_id;  
		from.type = PROCESS;  
		add_vertex(from);  

		struct source_type to;  
		to.id = tg->locklist[idx].id;  
		tg->locklist[idx].degress++;  
		to.type = PROCESS;  
		add_vertex(to);  
		  
		if (!verify_edge(from, to)) {  
			add_edge(from, to); //   
		}  
	}  
}  
```

}

void lock_after(uint64 thread_id, uint64 lockaddr) {

```
int idx = 0;  
if (-1 == (idx = search_lock(lockaddr))) {  // 该锁还没被其他线程lock过  

	int eidx = search_empty_lock(lockaddr);  
	  
	tg->locklist[eidx].id = thread_id;  
	tg->locklist[eidx].lock_id = lockaddr;  
	  
	inc(&tg->lockidx, 1);//lock id下标增加+1  
	  
} else {//idx不为-1，说明该锁被某个线程lock过，只是被该线程unlock了  

	struct source_type from;  
	from.id = thread_id;  
	from.type = PROCESS;  

	struct source_type to;  
	to.id = tg->locklist[idx].id;  
	tg->locklist[idx].degress --;  
	to.type = PROCESS;  

	if (verify_edge(from, to))//判断是否存在边，如果存在对应的边  
		remove_edge(from, to);  

	  
	tg->locklist[idx].id = thread_id; //设置对应下标的锁的线程id为当前锁的线程id  

}  
```

}

void unlock_after(uint64 thread_id, uint64 lockaddr) {

```
int idx = search_lock(lockaddr);  

if (tg->locklist[idx].degress == 0) {  
	tg->locklist[idx].id = 0;  
	tg->locklist[idx].lock_id = 0;  
	//inc(&tg->lockidx, -1);  
}  
```

}

int pthread_mutex_lock(pthread_mutex_t \*mutex) {

```
pthread_t selfid = pthread_self(); //  
lock_before(selfid, (uint64)mutex);//加锁前操作  
pthread_mutex_lock_f(mutex);  
lock_after(selfid, (uint64)mutex);//加锁后操作  
```

}

int pthread_mutex_unlock(pthread_mutex_t \*mutex) {

```
pthread_t selfid = pthread_self();  
pthread_mutex_unlock_f(mutex);  
unlock_after(selfid, (uint64)mutex);//解锁后操作  
```

}

static int init_hook() {\
//锁住系统函数 pthread_mutex_lock\
pthread_mutex_lock_f = dlsym(RTLD_NEXT, "pthread_mutex_lock");\
pthread_mutex_unlock_f = dlsym(RTLD_NEXT, "pthread_mutex_unlock");\
}

#if 1  //debug\
pthread_mutex_t mutex_1 = PTHREAD_MUTEX_INITIALIZER;\
pthread_mutex_t mutex_2 = PTHREAD_MUTEX_INITIALIZER;\
pthread_mutex_t mutex_3 = PTHREAD_MUTEX_INITIALIZER;\
pthread_mutex_t mutex_4 = PTHREAD_MUTEX_INITIALIZER;

void \*thread_rountine_1(void \*args)\
{\
pthread_t selfid = pthread_self(); //

```
printf("thread_routine 1 : %ld \n", selfid);  
  
pthread_mutex_lock(&mutex_1);//执行自定义的函数  
sleep(1);  
pthread_mutex_lock(&mutex_2);  

pthread_mutex_unlock(&mutex_2);  
pthread_mutex_unlock(&mutex_1);  

return (void *)(0);  
```

}

void \*thread_rountine_2(void \*args)\
{\
pthread_t selfid = pthread_self(); //

```
printf("thread_routine 2 : %ld \n", selfid);  
  
pthread_mutex_lock(&mutex_2);  
sleep(1);  
pthread_mutex_lock(&mutex_3);  

pthread_mutex_unlock(&mutex_3);  
pthread_mutex_unlock(&mutex_2);  

return (void *)(0);  
```

}

void \*thread_rountine_3(void \*args)\
{\
pthread_t selfid = pthread_self(); //

```
printf("thread_routine 3 : %ld \n", selfid);  

pthread_mutex_lock(&mutex_3);  
sleep(1);  
pthread_mutex_lock(&mutex_4);  

pthread_mutex_unlock(&mutex_4);  
pthread_mutex_unlock(&mutex_3);  

return (void *)(0);  
```

}

void \*thread_rountine_4(void \*args)\
{\
pthread_t selfid = pthread_self(); //

```
printf("thread_routine 4 : %ld \n", selfid);  
  
pthread_mutex_lock(&mutex_4);  
sleep(1);  
pthread_mutex_lock(&mutex_1);  

pthread_mutex_unlock(&mutex_1);  
pthread_mutex_unlock(&mutex_4);  

return (void *)(0);  
```

}

int main()\
{\
init_hook();\
start_check();\
printf("start_check\\n");\
pthread_t tid1, tid2, tid3, tid4;\
pthread_create(&tid1, NULL, thread_rountine_1, NULL);\
pthread_create(&tid2, NULL, thread_rountine_2, NULL);\
pthread_create(&tid3, NULL, thread_rountine_3, NULL);\
pthread_create(&tid4, NULL, thread_rountine_4, NULL);\
pthread_join(tid1, NULL);//等待线程1的结束\
pthread_join(tid2, NULL);\
pthread_join(tid3, NULL);\
pthread_join(tid4, NULL);\
return 0;\
}\
#endif

### 

运行

gcc -o deadlock deadlock.c -lpthread -ldl;

阅读 999

​

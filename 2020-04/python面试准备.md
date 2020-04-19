# python3
### python内存管理
1. **垃圾回收机制**(见下)
2. **内存池机制**：预先在内存中，申请一定数量和大小的内存块，作为备用  
&emsp;&emsp;&emsp;&emsp; 针对小内存(默认小于256kb)，会从内存池中分配，释放空间返回内存池  
&emsp;&emsp;&emsp;&emsp; 针对大内存，之间调用c语言的new/malloc函数，申请地址空间
3. **对象缓冲池**： python解释器启动时，会开辟一定的内存地址空间，用于存储高频使用的对象;  
&emsp;&emsp;&emsp;&emsp; 数值型、字符串、列表、字典等都会有自己的对象缓存池

<br/>

### python垃圾回收
1. **引用计数:** 当引用计数为0时，删除
2. **标记清除:**  针对循环引用的情况  
       **&emsp;&emsp;具体实现**： 垃圾回收时，将集合中的对应的引用计数复制一份副本，改动改对象引用的副本。  
       拷贝副本中，将对象分为两组，一组为root object(存活组，引用计数大于0), 一组为unreachable(死亡组，引用计数为0),然后遍历存活组对象，如果有引用代死亡组的对象，则该对象的引用计数+1，并且从死亡组迁移到存储组，这样可以避免被回收。只有当循环引用的两个对象均被删除后，才会被回收。
    >使用弱引用(weakref)，可以有效避免循环引用的问题(弱引用不记入引用计数)  
       
3. **分代回收**： 默认共分3代，垃圾回收的执行频率随着代数的增加而降低，即认为存活越久的数据，越不可能是垃圾

<br/>

### python调优
1. 多线程 到 多进程，因为有GIL锁的存在，多线程无法充分利用多核CPU的情况
2. 如果确认单线程可以满足程序需求，不必要使用多线程(减少线程间的竞争和CPU切换的消耗)
3. 减少函数调用层级
   > 每一层函数调用的函数栈开销不多，但是对于调用特别频繁的函数，多层调用开销会较大，这时候可以考虑将多层调用展开。

4. 优化对象属性查找(因为对象的属性查找过程需要查看__dict__, \_\_get__, __getattr__等方法，速度较慢)，如果是函数内循环中需要使用的对象属性，可以定义为局部变量
5. 如果数据量大，可以使用生成器替代循环，减少内存占用
6. 关闭垃圾回收机制。python垃圾回收会中断正在执行的程序，禁用后，性能可提高10%(gc.disable())
    > 关闭后存在的循环引用问题，可使用弱引用解决(weakref)
    
7. 对于固定对象，使用__slots__(对象属性不可变)代替__dict__，减少内存消耗
8. 其他pythonic细节：
        
        * 判断是否是同一个对象使用 is(比较对象id是否相等) 而不是 == (比较对象值是否相等)
        * 逻辑表达式，利用短路原则，先对大概率事件进行条件判断
        * 大量字符串的拼接，优先选用join()，而不是 `+`  
            >> `+`是字符串拼接， join是列表拼接
        * 使用for-else/while-else语法，可以减少一个标志位的使用
        * 交换两个变量的值使用： a, b = b, a (减少一个临时变量的使用)
        * 判断一个对象是否在一个集合中，使用set而不是list(set底层是hash表，时间为O(1))
        * dict优先使用iteritems方法(返回迭代器)，而不items方法(返回列表)

<br/>

#### join效率高的原因:
1. 字符串对象是不可改变的，也就是说在python创建一个字符串后，你不能把这个字符中的某一部分改变。  
2. 任何函数改变了字符串后，都会返回一个新的字符串，原字串并没有变。  
3. 因此，任何对字符串的操作包括’+’操作符都将创建一个新的字符串对象，而不是改变原来的对象.所以，N个字符串相加必将丢弃中间N-1个结果。
4. 而列表不同，列表是可以改变的，因此前面使用list的append，再使用join还原成字符串，只内建了一次，节省了很多资源和时间。

<br/>

#### 迭代器&生成器
##### 迭代器
迭代器是一个可以记住遍历的位置的对象。一个实现了_iter_方法和_next_方法的类对象，就是迭代器。  
**_iter_(self)方法**：在for循环遍历时被iter()调用;因为在遍历的时候，是直接调用的python内置函数iter()，由iter()通过调用_iter_(self)获得对象的迭代器。        
**_next_(self)方法**：在逐个遍历的时候，使用内置的next(）函数，通过调用对象的_next_(self)方法对迭代器对象进行遍历。  
*__注意点__*：_iter_(self)只会被调用一次,而_next_(self)会被调用 n 次，直到出现StopIteration异常。
##### 生成器
> 生成器是一类特殊的迭代器。  
> 生成器是只能遍历一次的。  
> 延迟操作。也就是在需要的时候才产生结果，不是立即产生结果。  

**生成器函数:** 使用 def 定义函数，但是，使用yield而不是return语句返回结果。yield语句一次返回一个结果，在每个结果中间，挂起函数的状态，以便下次从它离开的地方继续执行。  
**生成器表达式:** 使用一对小括号(),括号里是和列表推导一样的for循环；按需产生一个生成器结果对象，要想拿到每一个元素，就需要循环遍历。  
*__好处__*: 生成器表达式可以同时节省 cpu 和 内存(RAM);如果你构造一个列表(list)的目的仅仅是传递给别的函数,比如 传递给tuple()或者set(), 那就用生成器表达式替代吧!

<br/>

#### 装饰器
##### 1. 闭包原理
```
def add(x):
    def do_add(value):
        return x + value
    return do_add
    
add_5 = add(5)
print add_5(1)  # 6
```
如果在一个内部函数里，对在外部作用域（但不是在全局作用域）的变量进行引用，那么内部函数就被认为是**闭包**(closure)。    
**闭包=函数块+定义函数时的环境**，do_add就是函数块，x就是环境(当然这个环境可以有很多，不止一个简单的x。)   
**注意事项：** 1. 闭包无法修改外部函数的局部变量  
&emsp;&emsp;&emsp;&emsp;&emsp;2. 闭包无法直接访问外部函数的局部变量   
&emsp;&emsp;&emsp;&emsp;&emsp;3. python循环中不包含域的概念  
**用途1**：当闭包执行完后，仍然能够保持住当前的运行环境。
```
origin = [0, 0]
def create(pos=origin):
    def player(direction,step):
        new_x = pos[0] + direction[0]*step
        new_y = pos[1] + direction[1]*step
        pos[0] = new_x
        pos[1] = new_y
        #注意！此处不能写成 pos = [new_x, new_y]，因为参数变量不能被修改，而pos[]是容器类的解决方法
        return pos
    return player

player = create() # 创建棋子player，起点为原点
print (player([1,0],10)) # 向x轴正方向移动10步 [10, 0]
print (player([0,1],20)) # 向y轴正方向移动20步 [10, 20]
print (player([-1,0],10)) # 向x轴负方向移动10步 [0, 20]
```
**用途2**：闭包可以根据外部作用域的局部变量来得到不同的结果  
参考连接:[https://blog.csdn.net/Yeoman92/article/details/67636060](https://blog.csdn.net/Yeoman92/article/details/67636060)
##### 2. 其他
* 使用@符号作为装饰器的语法糖
* 使用functools.wraps(func)，把原函数的元信息拷贝到装饰器里面的 func 函数中。函数的元信息包括docstring、name、参数列表等等
* 类也可以作为装饰器，装饰器也可以修饰类，但是装饰器在类中，无法继承，即子类无法使用父类的装饰器
* 函数作为Python作为第一类对象；函数可以作为对象、可以作为返回值、可以赋值给变量、可以添加到容器里

<br/>

#### 线程 --  threading 模块
##### 创建方法：
1. 通过threading.Thread直接在线程中运行函数；直接传入函数和函数的参数  
```
thread1 = threading.Thread(target=threadfun,args=(1,6))
thread1.start()  # 调用start()，运行线程
```
2. 通过继承 thread.Thread 类 来创建线程,重载 threading.Thread 类的 run 方法
```
class mythread(threading.Thread):
    def run(self):
        for i in range(1,5):
            print(i)
 
threada = mythread()
threada.start()
```
##### 其他函数:
**join():** 调用 join() 的线程阻塞直到 某一线程结束才继续执行

    ta.start()          #开启ta线程
    ta.join()           #主线程等待 ta线程结束才继续执行
**isAlive():** 用于判断线程是否运行。  
**name属性:** 表示线程的线程名 默认是 Thread-x  x是序号，由1开始，第一个创建的线程名字就是 Thread-1   
**daemon属性：** 设置线程是否随主线程退出而退出； False(默认),线程不会随主线程退出; True, 主线程退出时，其他子线程也退出
##### 多线程间同步&通讯
**1. 锁:** 互斥锁(threading.Lock())和可重入锁(threading.Lock())  
&emsp;&emsp; 可重入锁在一个线程中可以多次获取，防止发生死锁  
&emsp;&emsp; acquire() -- 获取, release() -- 释放  
**2. 信号量:** threading.Semaphore(3) PV操作，每次获取减少1，减少到0时阻塞  
&emsp;&emsp; acquire() -- 获取, release() -- 释放  
**3. Event事件:**  threading.Event()  
&emsp;&emsp; 事件在内部管理了一个标志Flag，如果Flag值为False(默认值)，那么线程在执行event.wait方法时就会阻塞等值直到Flag值为True，该线程便会顺利执行，而Flag的值是通过event.set()和event.clear()设定的。
        
        event.clear() -- 将Flag标志设为Flase(默认值)，阻塞wait()中的线程
        event.set() -- 将Flag标志设置为True, 会唤醒wait()中的线程
        event.wait(timeout) -- 调用该方法的线程会被阻塞，如果设置了timeout参数，超时后，线程会停止阻塞继续执行  
        event.isSet() -- 获取当前event的标志位
**4. 条件变量:** threading.Condition()  
&emsp;&emsp; 可以接受一个Lock/RLock对象作为参数，如果没有指定，则Condition对象会在内部自行创建一个RLock。

        c.acquire(*args):获取底层锁。此方法将调用底层锁上对应的acquire(*args)方法。
        c.release()：释放底层锁。此方法将调用底层锁上对应的release()方法。
        c.wait(timeout)：等待直到获取通知或出现超时为止。此方法在acquire()方法后调用。
          >> 调用时，将释放底层锁，而且线程将进入睡眠状态，直到另一个线程在条件变量上执行notify()或notify_all()方法将其唤醒为止。
          >> 如果超时，也会唤醒，重新获取锁。
        c.notify(n)：唤醒一个或多个等待此条件变量的线程。此方法只会acquire()方法后调用。
        c.notify_all()：唤醒所有等待此条件的线程。

**基本原理:**  
&emsp;&emsp; Condition对象维护了一个锁（Lock/RLock)和一个waiting池。线程通过acquire获得Condition对象，当调用wait方法时，线程会释放Condition内部的锁并进入blocked状态，同时在waiting池中记录这个线程。当调用notify方法时，Condition对象会从waiting池中挑选一个线程，通知其调用acquire方法尝试取到锁。  
&emsp;&emsp; 除了notify方法外，Condition对象还提供了notifyAll方法，可以通知waiting池中的所有线程尝试acquire内部锁。由于上述机制，处于waiting状态的线程只能通过notify方法唤醒，所以notifyAll的作用在于防止有的线程永远处于沉默状态。  
**5. 消息队列:** queue.Queue()--先进先出队列, LifoQueue()--先进后出队列, PriorityQueue()--优先级队列
        
        Queue.qsize() -- 返回队列大小
        Queue.empty() -- 判断队列释放为空，空返回True，非空返回False
        Queue.full() -- 判断队列释放为满，满返回True，非满返回False
        Queue.put(item, block=True, timeout=None) -- 添加数据
        Queue.get(block=True, timeout=None) -- 读取队列首数据

<br/>

#### 线程池
**实现逻辑：** 队列+多个常驻现在
1.  threadpool模块的ThreadPool
2.  concurrent.futures模块的ThreadPoolExecutor(目前python3较多使用这个)  

**功能：**
1. 自动调度
2. 主线程可获取某一线程(任务)的状态和返回值
3. 一个线程完成时，主线程立刻知道  

**参数:**  `def __init__(self, max_workers=None, thread_name_prefix='',initializer=None, initargs=())`:
    
    max_workers -- 最大线程数
    thread_name_prefix  -- 线程名前缀,默认"ThreadPoolExecutor-"
    initializer -- 一个函数或方法，在启用线程前会调用这个函数（给线程池添加额外任务）。
    initargs -- 以元祖的方式给initializer中的函数传递参数。

**提交任务:**  
`submit(fn, *arg, **kwargs)`: 单独提交任务， 以`fn(*args **kwargs)`方式执行，并立即返回一个Future对象    
`map(func, *iterables, timeout=None, chunksize=1)`: 批量提交任务, 以迭代的方式将参数传递给函数，等所有执行完成后，返回
> chunksize：使用 ProcessPoolExecutor 时，这个方法会将 iterables 分割任务块，并作为独立的任务提交到执行池中。这些块的数量可以由 chunksize 指定设置。 对很长的迭代器来说，设置chunksize 值比默认值 1 能显著地提高性能。 chunksize 对 ThreadPoolExecutor 没有效果。

**提交任务返回:**  
`submit()`提交的任务，得到的返回是无序的，按照任务执行结束先后返回
`map()`提交的任务，得到的输出顺序和列表参数顺序一致，与返回先后无关

**as_completed(fs, timeout=None)**  
返回一个任务完成后的future对象，是一个生成器，在没有任务完成的时候，会阻塞，在有某个任务完成的时候，会yield这个任务; 多次相同提交，只返回一个结果    
参数： future对象列表(任务提交时的对象，任务提交后会立刻返回)  
任务的返回结果: future.result()

**wait(fs, timeout=None, return_when=ALL_COMPLETED)**  
阻塞主线程，直到满足设定的条件

    fs -- future对象列表(任务提交时的对象，任务提交后会立刻返回) 
    timeout -- 最长等待事件，None为永远等待
    return_when -- wait结束条件
        ALL_COMPLETED -- 所以任务完成
        FIRST_COMPLETED -- 任何一任务完成
        FIRST_EXCEPTION -- 任何一任务异常

**shutdown(wait=True)**:  
清理与线程池相关的资源；可重复调用；不管 wait 的值是什么，整个 Python 程序将等到所有待执行的期程完成执行后才退出。
    
        wait=True(默认), shutdown()函数不会返回(主线程一直阻塞)，直至线程池中所有的线程都执行完。
        wait=False, shutdown()函数立刻返回(主线程不会阻塞)


参考:[https://www.lagou.com/lgeduarticle/37786.html](https://www.lagou.com/lgeduarticle/37786.html)


<br/>

#### 队列详解
##### 分类  
**class queue.Queue(maxsize=0)**  
&emsp;&emsp; FIFO队列，maxsize为队列最大容量，maxsize=0表示无限大  
**class queue.LifoQueue(maxsize=0)**  
&emsp;&emsp; LIFO队列，maxsize为队列最大容量，maxsize=0表示无限大  
**class queue.PriorityQueue(maxsize=0)**  
&emsp;&emsp; 优先级队列，maxsize为队列最大容量，maxsize=0表示无限大，条目的典型为元组:`(priority_number, data)`

##### task_done() 和 join()的理解:  

**Queue.task_done()**  
表示前面排队的任务已经完成，通知消息队列将消息队列计数减一。  
每个`get()`被用于获取一个任务， 后续调用`task_done()`告诉队列，该任务的处理已经完成。  
**注意:**  `task_done()`的计数，只争对`join()`有效，对于`qsize()`、`empty()`、`full()`无效

**Queue.join()**  
阻塞至队列中所有的元素都被接收和处理完毕。
如果在join()前，未想Queue中添加任何条目，不会阻塞(所以，使用时需要在join所在的主线程先添加一定数据的元素)  

**比较:**  
task_done()和join() -- 关注任务又没有执行，担任任务执行完用task_done()通知，所有任务执行完join()解除阻塞  
get()、put()、empty()、full()和qsize() -- 关注队列中的元素个数，不关注队列中元素取出来后是否被使用

##### 其他问题
##### 1. 深拷贝、浅拷贝、赋值 
**赋值:** 对象引用的传递，相对于取别名，对象的引用计数加一，赋值前后，对象的id是一样的，因此对对象的修改，会对所有引用都产生影响    
**浅拷贝copy**： 拷贝的是父对象，不会拷贝父对象的子对象。拷贝前后，父对象的id不同，但是子对象的id未发送改变(因为父对象保存的是子对象的引用)，因此对子对象的修改，会对拷贝前后的对象，都有影响  
**深拷贝copy.deepcopy():** 完全开辟新的地址空间，用来保存原来的对象。新对象和原来互补影响，只是初始数据相同。  

参考:[https://www.runoob.com/w3cnote/python-understanding-dict-copy-shallow-or-deep.html](https://www.runoob.com/w3cnote/python-understanding-dict-copy-shallow-or-deep.html)

##### 2. return返回和yield返回的区别  
**return:** 表示函数调用结束，返回后函数结束运行，调用函数调用栈减一  
**yield:** 让函数变成一个生成器，生成器每次产生一个值(yield语句),然后被冻结，等待下一次唤醒，再产生新的值;    
即函数并没有运行结束，临时变量仍然存在

##### 3. 异常处理
* try catch exception else finally
* with语法: 实现一个上下文管理器，必须包含`__enter__`和`__exit__`
    
        __enter__ 执行一些初始化操作，并且函数的返回值会赋值给 as target中的target变量
        __exit__  执行资源清理工作，它接受三个参数，异常类型、异常实例和异常栈，根据这些信息进行异常处理和资源回收

##### 4. GIL(Global Interpreter Lock)  全局解释器锁
**明确:** GIL并不是Python的特性，它是在实现Python解析器(CPython)时所引入的一个概念。   
**观点:** GIL只会影响CPU密集型的程序，对IO密集型的程序，影响不大
**原因:** Python开始支持多线程。而解决多线程之间数据完整性和状态同步的最简单方法自然就是加锁。 于是有了GIL这把超级大锁,即默认python内部对象是thread-safe的，无需在实现时考虑额外的内存锁和同步操作。  
**影响:**  
> 1. 无法充分利用多核cpu的特性
> 2. cpu密集型操作，多线程的效果不如单线程

**解决办法:**
> 1. 使用多进程(multiprocessing)和进程池(concurrent.futures模块的ProcessPoolExecutor)
> 2. 在python中，使用C扩展编程，cpu密集型操作转交给C执行
> 3. 更换其他解释器(比如JPython就没有GIL)

##### 5. 部分对象的底层实现
**list:** 一个长度可变的连续数组(连续地址空间)。其中底层结构体中:ob_item 是一个指针列表，里边的每一个指针都指向列表中的元素。采用了指数过分配(空间不足时会使用4倍扩容，到5W时，会2倍扩容)，所以并不是每次操作都需要改变数组的大小。  
**tuple:** 一个长度不可变的连续数组(连续地址空间)。  
**dict:** 字典是通过散列表或说哈希表实现的。  
**set:** 集合是通过散列表或说哈希表实现的。集合被实现为带有空值的字典，只有键才是实际的集合元素。  
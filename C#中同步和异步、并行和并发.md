# C#中同步和异步、并行和并发(应用实例篇)

## 目录
- [基本概念](#基本概念)
  - [同步和异步](#同步和异步)
  - [并行和并发](#并行和并发)
- [C#中多线程处理](#C#中多线程处理)
  - [Thread的使用](#Thread的使用)
  - [ThreadPool的使用](#ThreadPool的使用)
  - [Task的使用](#Task的使用)
- [Task使用细节](#Task使用细节)
  - [Task延续](#Task延续)
  - [Task使用过程中的异常处理](#Task使用过程中的异常处理)
  - [取消Task](#取消Task)
  - [创建长时间执行的任务](#创建长时间执行的任务)
- [asyn和await的使用](#asyn和await的使用)
- [线程同步](#线程同步)
  - [使用Monitor来同步](#使用Monitor来同步)
  - [使用lock关键字](#使用lock关键字)
  - [lock对象的选择](#lock对象的选择)
  - [死锁](#死锁)
- [任务调度器和同步上下文](#任务调度器和同步上下文)
- [一些专业名称解释](#一些专业名词解释)
   


## 基本概念
### 同步和异步
---
   在维基和百度百科中并没有关于同步和异步的详细定义。有个有趣的说法是，**同步和异步是最终的目的，而线程是实现目的的一种途径**.这句话对我在学习这方面的知识时，还是有比较大的作用的.  
   因此，我们可以通过分析线程来理解同步和异步.  
   这样，我们大概需要知道什么是线程同步?什么是线程异步?  
   线程同步，我觉得主要有两点需要注意:**有序性和一致性**.  
         **有序性**:主要针对程序的执行顺序来说.比如单线程编程中，A();B(); ，必须等待A方法执行完了，B方法才可以执行.再比如,lock(sync){A();},无论多少个线程调用这段代码，A方法在同一个时刻只允许一个线程调用，其它线程必须等待.  
         **一致性**:主要针对数据来说.我们必须确保对临界区数据的变更不会影响其它线程.比如说，A线程和B线程在某一段时间都对data进行修改,为避免出现两个线程同时修改，后者的修改将前者的修改覆盖掉，我们对data的修改进行加锁,这样data一次只会允许一个线程进行修改.也就保证了数据的一致性.  
   线程异步,其实就是各干各的，开启多线程的时候就已经达到的异步的目的，为了让程序更加安全，我们需要在某些地方加一些特殊的限制，以达到同步的目的.  
   **事实上，在多线程问题的处理上，我们是在异步中谋求同步的目的，以确保程序的安全.**  
### 并行和并发
---
  两队人，进入一个门，每队每次轮流进一个，这称之为并发.  
  两队人，分别进入两个门，各进各的，这称之为并行.  
## C#中多线程处理
### Thread的使用
---
``` csharp
        static void Main(string[] args)
        {
            Thread th = new Thread(new ThreadStart(SayHello));
            Thread th1 = new Thread(new ParameterizedThreadStart(SayHi));

            th.Start();
            th1.Start("XXX");

            th.Join();
            th1.Join();

            Console.WriteLine("all child thread end...");
        }

        private static void SayHello()
        {
            Console.WriteLine("hello");
        }

        private static void SayHi(object o)
        {
            Console.WriteLine("hi " + o);
        }
```
### ThreadPool的使用
---
优点:可以通过重用线程获得更高IDE执行效率(创建线程的代价比较高昂,每个线程大概占用1MB的内存)
缺点: 1.使用线程池的作业执行时间较短
      2.不提供正在线程池中执行线程的引用,因此无法管理线程(比如无法知道线程什么时候结束)

``` csharp
        static void Main(string[] args)
        {
            ThreadPool.QueueUserWorkItem(SayHello);
            ThreadPool.QueueUserWorkItem(SayHello, "XX");
            Thread.Sleep(2000);
        }

        private static void SayHello(object state)
        {
            Console.WriteLine($"thread ID : {Thread.CurrentThread.ManagedThreadId}" + $" hi,{state}");
        }
```

### Task的使用
---
无返回值的Task调用
``` csharp
            Console.WriteLine($"current ID: {Thread.CurrentThread.ManagedThreadId} : Hello");

            Task task =Task.Run(() =>
            {
                Console.WriteLine($"current ID: {Thread.CurrentThread.ManagedThreadId} : Hello");
            });

            task.Wait();
```

带返回值的Task调用
``` csharp
        static void Main(string[] args)
        {
            Task<bool> task = Task.Run(() =>
            {
                return true;
            });

            task.Wait();

            System.Console.WriteLine(task.Result);
        }
```
## Task使用细节
### Task延续
---
  任务延续可以用于建立具有前后依赖关系的任务链
``` csharp
        static void Main(string[] args)
        {
            Console.WriteLine($"current ID: {Thread.CurrentThread.ManagedThreadId} : Hello");

            Task task =Task.Run(() =>
            {
                Console.WriteLine($"current ID: {Thread.CurrentThread.ManagedThreadId} : Hello");
            });

            Task taskA = task.ContinueWith((preTask) =>
            {
                Trace.Assert(preTask.Status == TaskStatus.RanToCompletion);
                Console.WriteLine("taskA ...");
            },TaskContinuationOptions.OnlyOnRanToCompletion);

            Task taskB = task.ContinueWith((preTask) =>
            {
                Trace.Assert(preTask.Status == TaskStatus.Canceled);
                Console.WriteLine("taskB...");
            },TaskContinuationOptions.OnlyOnCanceled);

            Task taskC = task.ContinueWith((preTask) =>
            {
                Trace.Assert(preTask.Status == TaskStatus.Faulted);
                Console.WriteLine("taskC...");
            },TaskContinuationOptions.OnlyOnFaulted);

            Thread.Sleep(5000);
            
        }
```

### Task使用过程中的异常处理
---

``` csharp
        static void Main(string[] args)
        {
            Task task = Task.Run(() =>
            {
                throw new InvalidOperationException();
            });

            try
            {
                task.Wait();

            }
            catch (AggregateException ex)
            {
                ex.Handle((eachException) =>
                {
                    Console.WriteLine(eachException.Message);
                    return true;
                });
            }
        }
```
### 取消Task
---
``` csharp
        static void Main(string[] args)
        {
            CancellationTokenSource cancellationTokenSource = new CancellationTokenSource();
            cancellationTokenSource.Token.Register(() =>
            {
                Console.WriteLine("cancel callback");
            });
            Task task = Task.Run(() =>
            {
                while (!cancellationTokenSource.Token.IsCancellationRequested)
                {
                   // Console.Write("-");
                }

                Console.WriteLine("cancel task ....");
            },cancellationTokenSource.Token);

            Console.WriteLine("Push any key to Cancel task!");
            Console.ReadKey();
            cancellationTokenSource.Cancel();
            task.Wait();
        }
```

### 创建长时间执行的任务
---
``` csharp
        static void Main(string[] args)
        {
            Task task = Task.Factory.StartNew(() =>
            {
                Console.WriteLine("this is long long time task...");
            }, TaskCreationOptions.LongRunning);

            task.Wait();
        }
```

## asyn和await的使用
---
``` csharp
        static void Main(string[] args)
        {
            Task task = DownloadDataAsyn();
            task.Wait();
        }

        public static async Task DownloadDataAsyn()
        {
            string url = "http://www.baidu.com";
            WebRequest webRequest = WebRequest.Create(url);
            WebResponse webResponse = await webRequest.GetResponseAsync();
            using (StreamReader sr = new StreamReader(webResponse.GetResponseStream()))
            {
                string text = await sr.ReadToEndAsync();
                Console.WriteLine("size: " + text.Length);
            }
        }
```
## 线程同步
### 使用Monitor来同步
---
``` csharp
        private static readonly object Sync = new object();
        private static int Count = 0;
        static void Main(string[] args)
        {
            Task task = Task.Run(() => Decrement());

            Increment();
            task.Wait();

            Console.WriteLine($"Count={Count}");


        }

        static void Decrement()
        {
            for (int i = 0; i < 100; i++)
            {
                bool lockToken = false;
                try
                {
                    Monitor.Enter(Sync, ref lockToken);
                    Count--;
                }
                finally
                {
                    if (lockToken)
                    {
                        Monitor.Exit(Sync);
                    }
                }
            }
        }

        static void Increment()
        {
            for (int i = 0; i < 100; i++)
            {
                bool lockToken = false;
                try
                {
                    Monitor.Enter(Sync, ref lockToken);
                    Count++;
                }
                finally
                {
                    if (lockToken)
                    {
                        Monitor.Exit(Sync);
                    }
                }
            }
        }
```

### 使用lock关键字
---
``` csharp
        private static readonly object Sync = new object();
        private static int Count = 0;
        static void Main(string[] args)
        {
            Task task = Task.Run(() =>
            {
                Decrement();
            });

            Increment();
            task.Wait();
        } 

        static void Increment()
        {
            lock (Sync)
            {
                for (int i = 0; i < 100; i++)
                {
                    Thread.Sleep(100);
                    Count++;
                    Console.WriteLine($"thread ID: {Thread.CurrentThread.ManagedThreadId},Count={Count}");
                }
            }
        }

        static void Decrement()
        {
            lock (Sync)
            {
                for (int i = 0; i < 100; i++)
                {
                    Thread.Sleep(100);
                    Count--;
                    Console.WriteLine($"thread ID: {Thread.CurrentThread.ManagedThreadId},Count={Count}");
                }   
            }

        }
```
### lock对象的选择（同步对象）
---
1.lock(this) 一般不建议这种,这种锁定用于锁定一个实例对象。只有基于当前实例对象的线程才会被同步

``` csharp
    class Program
    {
        static void Main(string[] args)
        {
            LockTest test = new LockTest();

            //正确lock(this)示范
            Thread th = new Thread(test.LockMethod);
            Thread th1 = new Thread(test.LockMethod);

            th.Start();
            th1.Start();

            th.Join();
            th1.Join();

            //使用this产生的矛盾点

            LockTest test1 = new LockTest();
            Thread th2 = new Thread(test.LockMethod);
            Thread th3 = new Thread(test1.LockMethod);

            th2.Start();
            th3.Start();

            th2.Join();
            th3.Join();

        }
    }

    public class LockTest
    {
        public void LockMethod()
        {
            lock (this)
            {
                Console.WriteLine($"thread ID: " + Thread.CurrentThread.ManagedThreadId + "sleep 5000 ms");
                Thread.Sleep(5000);
                Console.WriteLine($"thread ID: " + Thread.CurrentThread.ManagedThreadId + "Awake");
            }
        }
    }

    //thread ID: 3sleep 5000 ms
    //thread ID: 3Awake
    //thread ID: 4sleep 5000 ms
    //thread ID: 4Awake
    //thread ID: 5sleep 5000 ms
    //thread ID: 6sleep 5000 ms
    //thread ID: 5Awake
    //thread ID: 6Awake
```

2.lock(type) 一般不建议这种,这种锁定用于锁定类型.只要线程调用方法时，没有获取该种类型的锁，则会被阻塞.且此种锁定，该类型
``` csharp
    class Program
    {
        static void Main(string[] args)
        {
            Thread th = new Thread(ALockMethod);
            Thread th1 = new Thread(BLockMethod);

            th.Start();
            th1.Start();

            th.Join();
            th1.Join();
        }

        public static void ALockMethod()
        {
            lock (typeof(LockTest))
            {
                System.Console.WriteLine($"thread ID: {Thread.CurrentThread.ManagedThreadId} sleep 2000ms A ");
                Thread.Sleep(2000);
                System.Console.WriteLine($"thread ID: {Thread.CurrentThread.ManagedThreadId} Awake");
            }
        }

        public static void BLockMethod()
        {
            lock (typeof(LockTest))
            {
                System.Console.WriteLine($"thread ID: {Thread.CurrentThread.ManagedThreadId} sleep 2000ms B ");
                Thread.Sleep(2000);
                System.Console.WriteLine($"thread ID: {Thread.CurrentThread.ManagedThreadId} Awake");
            }
        }
    }

    class LockTest
    {

    }
```

3.lock(object) 推荐使用,锁 私有静态object 。 此种不局限于实例对象，不局限于类型，可以锁 想要锁定的部分.示例代码参考死锁.

### 死锁
---
注意:如果某一个线程上抛出异常,锁是会被释放的
1.因相互等待锁的释放而陷入死锁
``` csharp
        private static readonly object Sync = new object();
        private static readonly object Sync_1 = new object();
        static void Main(string[] args)
        {
            Thread th = new Thread(ALockMethod);
            Thread th1 = new Thread(BLockMethod);

            th.Start();
            th1.Start();

            th.Join();
            th1.Join();
            Console.WriteLine("end...");
        }

        public static void ALockMethod()
        {
            lock (Sync)
            {
                Console.WriteLine($"thread ID: {Thread.CurrentThread.ManagedThreadId} call ALockMethod");
                BLockMethod();
            }
        }

        public static void BLockMethod()
        {
            lock (Sync_1)
            {
                Console.WriteLine($"thread ID: {Thread.CurrentThread.ManagedThreadId} call BLockMethod");
                ALockMethod();
            }
        }
```

## 任务调度器和同步上下文
---
任务调度器 是为线程高效分配工作时所扮演的角色,可以将任务分配给合适的线程. 在C#中 是 System.Threading.Tasks.TaskScheduler 的实例.  
FCL提供了两个派生自TaskScheduler的任务调度器:线程池任务调度器和同步上下文调度器.其中,同步上下文通常应用于图形界面，将任务调度给GUI线程.而在控制台应用程序中,默认使用线程池调度任务，决定安全有效的执行他们-何时重用、何时进行资源清理和何时创建额外的线程.  

控制台应用程序中TaskScheduler和SynchronizationContext
``` csharp
        static void Main(string[] args)
        {
            Console.WriteLine(TaskScheduler.Default);
            Console.WriteLine(TaskScheduler.Current);

            //output
            //System.Threading.Tasks.ThreadPoolTaskScheduler
            //System.Threading.Tasks.ThreadPoolTaskScheduler

            Console.WriteLine(SynchronizationContext.Current==null);
            //output
            //true
        }
```

WPF 中演示同步上下文.只有在延续任务中才能更新UI，在task中更新UI会抛出异常
``` csharp
   public partial class MainWindow : Window
    {
        private readonly TaskScheduler taskScheduler;
        private CancellationTokenSource cancellationTokenSource;
        public MainWindow()
        {
            taskScheduler = TaskScheduler.FromCurrentSynchronizationContext();
            InitializeComponent();
            label1.Content = "initialize text";
            cancellationTokenSource = new CancellationTokenSource();
        }

        private void Button_Click(object sender, RoutedEventArgs e)
        {
            Task task = Task.Run(() => {
                //while (!cancellationTokenSource.Token.IsCancellationRequested)
                //{

                //}
                //label1.Content = "excuting task";
                Thread.Sleep(2000);

            }, cancellationTokenSource.Token);

            Task taskA = task.ContinueWith((preTask) =>
            {
                Trace.Assert(preTask.IsCompleted);
                label1.Content = "Task Completion...";
            },CancellationToken.None, TaskContinuationOptions.OnlyOnRanToCompletion,taskScheduler);


            Task taskB = task.ContinueWith((preTask) =>
            {
                Trace.Assert(preTask.IsCanceled);
                label1.Content = "Task Canceled...";
            }, CancellationToken.None, TaskContinuationOptions.OnlyOnCanceled, taskScheduler);

            Task taskC = task.ContinueWith((preTask) =>
            {
                Trace.Assert(preTask.IsFaulted);

                label1.Content = "Task Faulted...";
            }, CancellationToken.None,TaskContinuationOptions.OnlyOnFaulted, taskScheduler);

        }
    }
```

从TaskScheduler派生出新类型，可以创建自己的任务调度器，从而对任务调度做出不同的选择.可以使用静态方法FromCurrentSynchronizationContext()将任务调度给当前线程(更确切地说，调度给当前线程关联的同步上下文).  

## 一些专业名词解释
---
**TPL**: Task Parallel Library 任务并行库  
**TAP**: Task-based Asynchronous Pattern 基于任务的异步模式  
**前台线程/后台线程**:进程必须等待所有的前台进行执行完毕才能退出，而无需等待后台进程执行完毕.  
**任务原子性**:任务执行的最小单位。如果是数据的话，和系统位数相关。比如64位操作系统，可以完整的读取long,但是在读取decimal(128位)时，可能被中断  
**FCL**: Framework Class Library


                
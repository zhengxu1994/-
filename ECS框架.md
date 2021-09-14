# ECS框架

### 对ECS框架的理解:

### 解决了项目存在的最大问题，解耦，我理解的有哪些解耦：

##### 1.数据与逻辑分离，纯数据的部分存在component中，逻辑根本不关心数据存放在哪里，只需要拿过来用就可以，如果没有这份数据那我就不关心它。

##### 2.逻辑分离，单个system只关心它关心的部分（拥有N个component的对象），system只需要运行的自己逻辑就可以，不需要关心其他对象的逻辑。这样就保证了只需要当前这个system的逻辑没有问题就可以，不用关心其他的逻辑造成我当前的逻辑出错，或者我当前的逻辑造成其他的逻辑错误。我需要哪个逻辑时只要在Entity上挂在对应的component即可，不需要时移除component，不想传统的设计方式，根本没有办法把某块逻辑移除（除非给删了哈哈）。

##### 3.表现与逻辑分离，加了表现的组件才会做表现的事情，不加就是纯逻辑，这样客户端的逻辑就能直接拿到服务器上。



### ECS介绍：

- E: Entity  一个不代表任何意义的实体（可以理解为Unity里的一个空的GameObject）
- C: Component 一个只包含数据的组件（可以理解为Unity的一个自定义组件，里面只有数据，没有任何方法）
- S: System 一个用来处理数据的系统（可以理解为Unity的一个自定义组件，里面只有方法，没有任何数据）



- 面向过程： 1、摇（所有狗的尾巴） 2、摇（所有猪的尾巴）
- 面向对象： 1、所有狗.（摇尾巴） 2、所有猪.（摇尾巴）
- 面向数据： 1、收集所有的尾巴    2、摇（尾巴）



#### 除了解耦的其他优点：

##### 1.面向数据的方式，让内存排列天然的紧密，非常适合现代cpu的缓存机制，极大增加CPU的缓存命中率，大幅提升性能。





#### C# ECS源码分析

- ECS中的S,也就是system系统，通过system去处理组件和实体的逻辑，ecs中system分为5种system，所有的system最终都继承自ISystem接口。

  ```c#
      /// This is the base interface for all systems.
      /// It's not meant to be implemented.
      /// Use IInitializeSystem, IExecuteSystem,
      /// ICleanupSystem or ITearDownSystem.
      public interface ISystem {
      }
  ```

  

  - IExecuteSytem，每帧都会去执行。

    ```c#
        /// Implement this interface if you want to create a system which should be
        /// executed every frame.
        public interface IExecuteSystem : ISystem {
    
            void Execute();
        }
    ```

    JobSystem继承自IExecuteSystem，主要用于处理多线程任务系统。https://www.cnblogs.com/sifenkesi/p/12258842.html，讲述了JobSystem是如何使用多线程工作的，有什么优势。

    ```c#
        //JobSystem调用带有实体子集的Execute(实体)，并将工作负载分配到指定数量的线程上。在Entitas中编写多线程代码时，不要使用生成的方法，如AddXyz()和ReplaceXyz()。
        public abstract class JobSystem<TEntity> : IExecuteSystem where TEntity : class, IEntity {
    
            readonly IGroup<TEntity> _group;
            readonly int _threads;
            //job system管理job依赖关系，并保证执行时序的正确性
            readonly Job<TEntity>[] _jobs;
    
            int _threadsRunning;
    
            protected JobSystem(IGroup<TEntity> group, int threads) {
                _group = group;
                _threads = threads;
                _jobs = new Job<TEntity>[threads];
                for (int i = 0; i < _jobs.Length; i++) {
                    _jobs[i] = new Job<TEntity>();
                }
            }
    
            protected JobSystem(IGroup<TEntity> group) : this(group, Environment.ProcessorCount) {
            }
    
            public virtual void Execute() {
                _threadsRunning = _threads;
                var entities = _group.GetEntities();
                var remainder = entities.Length % _threads;
                //计算出每个线程处理多少个对象
                var slice = entities.Length / _threads + (remainder == 0 ? 0 : 1);
                for (int t = 0; t < _threads; t++) {
                    var from = t * slice;
                    var to = from + slice;
                    if (to > entities.Length) {
                        to = entities.Length;
                    }
    
                    var job = _jobs[t];
                    job.Set(entities, from, to);
                    if (from != to) {
                        //线程池
                        //将jobs放在一个job queue里面，worker threads从job queue里面获取job然后执行
                        ThreadPool.QueueUserWorkItem(queueOnThread, _jobs[t]);
                    } else {
                        Interlocked.Decrement(ref _threadsRunning);
                    }
                }
    
                while (_threadsRunning != 0) {
                }
    
                foreach (var job in _jobs) {
                    if (job.exception != null) {
                        throw job.exception;
                    }
                }
            }
            
            //线程callback 调用Execute处理对象
            void queueOnThread(object state) {
                var job = (Job<TEntity>)state;
                try {
                    for (int i = job.from; i < job.to; i++) {
                        Execute(job.entities[i]);
                    }
                } catch (Exception ex) {
                    job.exception = ex;
                } finally {
                    Interlocked.Decrement(ref _threadsRunning);
                }
            }
    
            protected abstract void Execute(TEntity entity);
        }
        //完成特定任务的一个小的工作单元。job接收参数并操作数据，类似于函数调用。job之间可以有依赖关系，也就是一个job可以等另一个job完成之后再执行。
        class Job<TEntity> where TEntity : class, IEntity {
    
            public TEntity[] entities;
            public int from;
            public int to;
            public Exception exception;
    
            public void Set(TEntity[] entities, int from, int to) {
                this.entities = entities;
                this.from = from;
                this.to = to;
                exception = null;
            }
        }
    ```

    

  - IInitializeSystem,只会在ecs环境初始化时调用。

    ```c#
      /// Implement this interface if you want to create a system which should be
        /// initialized once in the beginning.
        public interface IInitializeSystem : ISystem {
    
            void Initialize();
        }

  - IReactiveSystem,只某些条件触发时才会被调用,它继承自IExecuteSystem,但是复写的Execute()方法只有在收集器触发时才会调用。

    ```c#
       public interface IReactiveSystem : IExecuteSystem {
            //激活收集器
            void Activate();
            //禁用收集器
            void Deactivate();
            //清空收集器
            void Clear();
        }
    ```

    ReactiveSystem<TEntity>继承IReactiveSystem,继承ReactiveSystem的类需要复写ICollector，Filter，Execute方法。

    - ICollector，关注某类组件，当实体身上的某类组件发生改变时会收集这些实体对象
    - Filter，对收集的实体对象列表再次过滤，条件自定义。
    - Execute(List<TEntity>entities)，传入的参数就是每帧通过上面两个方法收集到的实体对象列表
    - ReactiveSystem构造函数中会初始化收集信息（context.CreateCollector()）context的拓展和一个用于存放实体对象的列表（ _buffer = new List<TEntity>()）
    - Execute(),从收集器中获取目标并通过Filter方法过滤后获得最终list列表传入到有参Execute方法，并清空收集器，在list使用完毕后release entity并清空列表list

    ```c#
     public abstract class ReactiveSystem<TEntity> : IReactiveSystem where TEntity : class, IEntity {
    
            readonly ICollector<TEntity> _collector;
            readonly List<TEntity> _buffer;
            string _toStringCache;
    
            protected ReactiveSystem(IContext<TEntity> context) {
                _collector = GetTrigger(context);
                _buffer = new List<TEntity>();
            }
    
            protected ReactiveSystem(ICollector<TEntity> collector) {
                _collector = collector;
                _buffer = new List<TEntity>();
            }
    
            /// Specify the collector that will trigger the ReactiveSystem.
            protected abstract ICollector<TEntity> GetTrigger(IContext<TEntity> context);
    
            /// This will exclude all entities which don't pass the filter.
            protected abstract bool Filter(TEntity entity);
    
            protected abstract void Execute(List<TEntity> entities);
    
            /// Activates the ReactiveSystem and starts observing changes
            /// based on the specified Collector.
            /// ReactiveSystem are activated by default.
            public void Activate() {
                _collector.Activate();
            }
    
            /// Deactivates the ReactiveSystem.
            /// No changes will be tracked while deactivated.
            /// This will also clear the ReactiveSystem.
            /// ReactiveSystem are activated by default.
            public void Deactivate() {
                _collector.Deactivate();
            }
    
            /// Clears all accumulated changes.
            public void Clear() {
                _collector.ClearCollectedEntities();
            }
    
            /// Will call Execute(entities) with changed entities
            /// if there are any. Otherwise it will not call Execute(entities).
            public void Execute() {
                if (_collector.count != 0) {
                    foreach (var e in _collector.collectedEntities) {
                        if (Filter(e)) {
                            e.Retain(this);
                            _buffer.Add(e);
                        }
                    }
    
                    _collector.ClearCollectedEntities();
    
                    if (_buffer.Count != 0) {
                        try {
                            Execute(_buffer);
                        } finally {
                            for (int i = 0; i < _buffer.Count; i++) {
                                _buffer[i].Release(this);
                            }
                            _buffer.Clear();
                        }
                    }
                }
            }
    
            public override string ToString() {
                if (_toStringCache == null) {
                    _toStringCache = "ReactiveSystem(" + GetType().Name + ")";
                }
    
                return _toStringCache;
            }
    
            ~ReactiveSystem() {
                Deactivate();
            }
        }
    ```

    MultiReactiveSystem继承自IReactiveSystem,多收集器触发系统,与ReactuveSystem差不多，只不过对象一个是通过单个collector收集一个是通过多个collector收集，其他的机制都一样。

    ```c#
     public abstract class MultiReactiveSystem<TEntity, TContexts> : IReactiveSystem
            where TEntity : class, IEntity
            where TContexts : class, IContexts {
    
            readonly ICollector[] _collectors;
            readonly HashSet<TEntity> _collectedEntities;
            readonly List<TEntity> _buffer;
            string _toStringCache;
    
            protected MultiReactiveSystem(TContexts contexts) {
                _collectors = GetTrigger(contexts);
                _collectedEntities = new HashSet<TEntity>();
                _buffer = new List<TEntity>();
            }
    
            protected MultiReactiveSystem(ICollector[] collectors) {
                _collectors = collectors;
                _collectedEntities = new HashSet<TEntity>();
                _buffer = new List<TEntity>();
            }
    
            /// Specify the collector that will trigger the ReactiveSystem.
            protected abstract ICollector[] GetTrigger(TContexts contexts);
    
            /// This will exclude all entities which don't pass the filter.
            protected abstract bool Filter(TEntity entity);
    
            protected abstract void Execute(List<TEntity> entities);
    
            /// Activates the ReactiveSystem and starts observing changes
            /// based on the specified Collector.
            /// ReactiveSystem are activated by default.
            public void Activate() {
                for (int i = 0; i < _collectors.Length; i++) {
                    _collectors[i].Activate();
                }
            }
    
            /// Deactivates the ReactiveSystem.
            /// No changes will be tracked while deactivated.
            /// This will also clear the ReactiveSystem.
            /// ReactiveSystem are activated by default.
            public void Deactivate() {
                for (int i = 0; i < _collectors.Length; i++) {
                    _collectors[i].Deactivate();
                }
            }
    
            /// Clears all accumulated changes.
            public void Clear() {
                for (int i = 0; i < _collectors.Length; i++) {
                    _collectors[i].ClearCollectedEntities();
                }
            }
    
            /// Will call Execute(entities) with changed entities
            /// if there are any. Otherwise it will not call Execute(entities).
            public void Execute() {
                for (int i = 0; i < _collectors.Length; i++) {
                    var collector = _collectors[i];
                    if (collector.count != 0) {
                        _collectedEntities.UnionWith(collector.GetCollectedEntities<TEntity>());
                        collector.ClearCollectedEntities();
                    }
                }
    
                foreach (var e in _collectedEntities) {
                    if (Filter(e)) {
                        e.Retain(this);
                        _buffer.Add(e);
                    }
                }
    
                if (_buffer.Count != 0) {
                    try {
                        Execute(_buffer);
                    } finally {
                        for (int i = 0; i < _buffer.Count; i++) {
                            _buffer[i].Release(this);
                        }
                        _collectedEntities.Clear();
                        _buffer.Clear();
                    }
                }
            }
    
            public override string ToString() {
                if (_toStringCache == null) {
                    _toStringCache = "MultiReactiveSystem(" + GetType().Name + ")";
                }
    
                return _toStringCache;
            }
    
            ~MultiReactiveSystem() {
                Deactivate();
            }
        }
    ```

    

  - IClearUpSystem,在每帧update后调用。

    ```c#
        /// Implement this interface if you want to create a system which should
        /// execute cleanup logic after execution.
        public interface ICleanupSystem : ISystem {
    
            void Cleanup();
        }
    ```

    

  - ITearDownSystem ,游戏结束销毁时调用。

    ```c#
        /// Implement this interface if you want to create a system which should
        /// tear down once in the end.
        public interface ITearDownSystem : ISystem {
    
            void TearDown();
        }
    ```

  - System，管理各个类型的system，将子system加入到system中，system会调用Initialize，Execute，Cleanup,TearDown,ActivateReactiveSystems,DeactivateReactiveSystems,ClearReactiveSystems方法去遍历调用各个system的逻辑接口。

    ```c#
     public class Systems : IInitializeSystem, IExecuteSystem, ICleanupSystem, ITearDownSystem {
    
            protected readonly List<IInitializeSystem> _initializeSystems;
            protected readonly List<IExecuteSystem> _executeSystems;
            protected readonly List<ICleanupSystem> _cleanupSystems;
            protected readonly List<ITearDownSystem> _tearDownSystems;
    
            /// Creates a new Systems instance.
            public Systems() {
                _initializeSystems = new List<IInitializeSystem>();
                _executeSystems = new List<IExecuteSystem>();
                _cleanupSystems = new List<ICleanupSystem>();
                _tearDownSystems = new List<ITearDownSystem>();
            }
    
            /// Adds the system instance to the systems list.
            public virtual Systems Add(ISystem system) {
                var initializeSystem = system as IInitializeSystem;
                if (initializeSystem != null) {
                    _initializeSystems.Add(initializeSystem);
                }
    
                var executeSystem = system as IExecuteSystem;
                if (executeSystem != null) {
                    _executeSystems.Add(executeSystem);
                }
    
                var cleanupSystem = system as ICleanupSystem;
                if (cleanupSystem != null) {
                    _cleanupSystems.Add(cleanupSystem);
                }
    
                var tearDownSystem = system as ITearDownSystem;
                if (tearDownSystem != null) {
                    _tearDownSystems.Add(tearDownSystem);
                }
    
                return this;
            }
    
            /// Calls Initialize() on all IInitializeSystem and other
            /// nested Systems instances in the order you added them.
            public virtual void Initialize() {
                for (int i = 0; i < _initializeSystems.Count; i++) {
                    _initializeSystems[i].Initialize();
                }
            }
    
            /// Calls Execute() on all IExecuteSystem and other
            /// nested Systems instances in the order you added them.
            public virtual void Execute() {
                for (int i = 0; i < _executeSystems.Count; i++) {
                    _executeSystems[i].Execute();
                }
            }
    
            /// Calls Cleanup() on all ICleanupSystem and other
            /// nested Systems instances in the order you added them.
            public virtual void Cleanup() {
                for (int i = 0; i < _cleanupSystems.Count; i++) {
                    _cleanupSystems[i].Cleanup();
                }
            }
    
            /// Calls TearDown() on all ITearDownSystem  and other
            /// nested Systems instances in the order you added them.
            public virtual void TearDown() {
                for (int i = 0; i < _tearDownSystems.Count; i++) {
                    _tearDownSystems[i].TearDown();
                }
            }
    
            /// Activates all ReactiveSystems in the systems list.
            public void ActivateReactiveSystems() {
                for (int i = 0; i < _executeSystems.Count; i++) {
                    var system = _executeSystems[i];
                    var reactiveSystem = system as IReactiveSystem;
                    if (reactiveSystem != null) {
                        reactiveSystem.Activate();
                    }
    
                    var nestedSystems = system as Systems;
                    if (nestedSystems != null) {
                        nestedSystems.ActivateReactiveSystems();
                    }
                }
            }
    
            /// Deactivates all ReactiveSystems in the systems list.
            /// This will also clear all ReactiveSystems.
            /// This is useful when you want to soft-restart your application and
            /// want to reuse your existing system instances.
            public void DeactivateReactiveSystems() {
                for (int i = 0; i < _executeSystems.Count; i++) {
                    var system = _executeSystems[i];
                    var reactiveSystem = system as IReactiveSystem;
                    if (reactiveSystem != null) {
                        reactiveSystem.Deactivate();
                    }
    
                    var nestedSystems = system as Systems;
                    if (nestedSystems != null) {
                        nestedSystems.DeactivateReactiveSystems();
                    }
                }
            }
    
            /// Clears all ReactiveSystems in the systems list.
            public void ClearReactiveSystems() {
                for (int i = 0; i < _executeSystems.Count; i++) {
                    var system = _executeSystems[i];
                    var reactiveSystem = system as IReactiveSystem;
                    if (reactiveSystem != null) {
                        reactiveSystem.Clear();
                    }
    
                    var nestedSystems = system as Systems;
                    if (nestedSystems != null) {
                        nestedSystems.ClearReactiveSystems();
                    }
                }
            }
        }
    ```

- ECS中的C，Component组件，组件中存放的是数据，实体通过添加组件来获取数据，系统通过操作组件来修改数据，实体本身不对数据进行操作。




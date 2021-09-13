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
    - Execute，传入的参数就是每帧通过上面两个方法收集到的实体对象列表
    - ReactiveSystem构造函数中会初始化收集信息（context.CreateCollector()）context的拓展方法和一个用于存放实体对象的列表（ _buffer = new List<TEntity>()）
    - 

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

    




# `k8s` client-go源码解析 -  1) tools/cache 

> client-go是一个用来与Kubernetes API交互的go工具库，开发CRD需要用到的底层库。之前简略的看过其源码，但是不够细致，导致对于一些概念理解不够深刻。所以，准备再抽时间细读一下。

![自定义控制器的工作流程示意图](http://devops-1255386119.cos.ap-beijing.myqcloud.com/2023-09-24-075942.jpg)



![image-20230928220521167](http://devops-1255386119.cos.ap-beijing.myqcloud.com/2023-09-28-140525.png)

[架构图](https://doc.weixin.qq.com/flowchart/f4_ACcAswZ6ACcDIRL11phTc00g9cL50?scode=AJEAIQdfAAossDciuOACcAswZ6ACc&is_external=0&commentVersion=1695889647000&doc_title=client-go&open_source=timeline)

## TLDR

Reflector 负责通过ListWatch将变更事件写入DeltaFIFO,然后Informer负责消费变更事件。为了相同资源能够实现Reflector的共享和降低api-server的压力，又出现了SharedInformer的概念。SharedInformer又通过sharedProcessor支持将变更事件分发到不同的listener,从而实现一个事件可以发送给多个handler。

## ThreadSafeStore

> 假设存在 podobj(name: myapp, namespace: default, label: A,B,C)、 deploymentobj(name: mydeployment, namespace: formal,  label: B,C,D)等数据
>
> 现在要查找 快速查找
>
> 1. name为myapp的obj
>
> 2. namespace 为 default 的 obj 
>
> 3. label为A的obj 

每个维度创建一个map？`map[${name}]interface{}`, `map[${namespace]interface{}`, `map[${label}]interface`

不可以，更新或者删除索引时无法维护。那就

`map[${name}]interface{}`作为主索引，而`map[${namespace]map[string]struct{}`, `map[${label}]map[string]struct{}`作为二级索引呢？

(⊙o⊙)…，这不是就是mysql的Secondary Index么

<img src="http://devops-1255386119.cos.ap-beijing.myqcloud.com/2023-09-23-084329.png" alt="image-20230923164328536"  />

所以实际最终存储的方式是：

```golang
var primaryIndex = map[string]interface{}{
	"default/myapp":        podobj,
	"formal/mydeployment": deploymentobj,
}

var secondaryIndex = map[string]map[string]map[string]struct{}{
	"namespace": {
		"default": {
			"myapp": {},
		},
		"formal": {
			"mydeployment": {},
		},
	},

	"label": {
		"A": {
			"myapp": {},
		},
		"B": {
			"myapp":        {},
			"mydeployment": {},
		},
		"C": {
			"myapp":        {},
			"mydeployment": {},
		},
		"D": {
			"mydeployment": {},
		},
	},
}
```

知道了大概功能，源码读起来就相当简单了。简要代码如下，添加了少量注释。

```go
type ThreadSafeStore interface {
	Add(key string, obj interface{})
	Update(key string, obj interface{})
	Delete(key string)
	Get(key string) (item interface{}, exists bool)
	List() []interface{}
	ListKeys() []string
	Replace(map[string]interface{}, string)
  // 示例：indexer.Index(NamespaceIndex, &metav1.ObjectMeta{Namespace: "formal"})
  // 获取namespace为formal的所有obj
	Index(indexName string, obj interface{}) ([]interface{}, error)
	IndexKeys(indexName, indexedValue string) ([]string, error)
	ListIndexFuncValues(name string) []string
  // 比如获取namespace为formal的所有obj
	ByIndex(indexName, indexedValue string) ([]interface{}, error)
	GetIndexers() Indexers

	// AddIndexers adds more indexers to this store.  If you call this after you already have data
	// in the store, the results are undefined.
	AddIndexers(newIndexers Indexers) error
	// Resync is a no-op and is deprecated
	Resync() error
}

// storeIndex implements the indexing functionality for Store interface
type storeIndex struct {
	// indexers maps a name to an IndexFunc
	indexers Indexers
	// indices maps a name to an Index
	indices Indices
}

// threadSafeMap implements ThreadSafeStore
type threadSafeMap struct {
	lock  sync.RWMutex
	items map[string]interface{}

	// index implements the indexing functionality
	index *storeIndex
}

// IndexFunc knows how to compute the set of indexed values for an object.
// IndexFunc 获取一个结构体的特征,比如label\annotations\namespace
// IndexFunc knows how to compute the set of indexed values for an object.
type IndexFunc func(obj interface{}) ([]string, error)

type Index map[string]sets.String

// Indexers maps a name to an IndexFunc
type Indexers map[string]IndexFunc

// Indices maps a name to an Index
type Indices map[string]Index
```

## Store\Indexer
Store Interface和Indexer Interface的的底层实现都是cache, 而Indexer也继承了Store Interface。

Store和ThreadSafeStore的区别可以从Add函数实现看出：

ThreadSafeStore的key是输入参数，用户可以传入任意值，更为灵活；

而Store则是直接将生成key的函数keyFunc作为结构体的成员，通过keyFunc(obj)直接生成key，更为具体。一般情况下对应函数为：`MetaNamespaceKeyFunc`，生成的格式为`namespace/metadata.data`

Indexer则更为具体，直接将keyFunc和indexers,这里的给定indexes相当于建好表的同时指定好索引。


```go
type Store interface {

	// Add adds the given object to the accumulator associated with the given object's key
	Add(obj interface{}) error

	// Update updates the given object in the accumulator associated with the given object's key
	Update(obj interface{}) error

	// Delete deletes the given object from the accumulator associated with the given object's key
	Delete(obj interface{}) error

	// List returns a list of all the currently non-empty accumulators
	List() []interface{}

	// ListKeys returns a list of all the keys currently associated with non-empty accumulators
	ListKeys() []string

	// Get returns the accumulator associated with the given object's key
	Get(obj interface{}) (item interface{}, exists bool, err error)

	// GetByKey returns the accumulator associated with the given key
	GetByKey(key string) (item interface{}, exists bool, err error)

	// Replace will delete the contents of the store, using instead the
	// given list. Store takes ownership of the list, you should not reference
	// it after calling this function.
	Replace([]interface{}, string) error

	// Resync is meaningless in the terms appearing here but has
	// meaning in some implementations that have non-trivial
	// additional behavior (e.g., DeltaFIFO).
	Resync() error
}


type Indexer interface {
	Store
	// Index returns the stored objects whose set of indexed values
	// intersects the set of indexed values of the given object, for
	// the named index
	Index(indexName string, obj interface{}) ([]interface{}, error)
	// IndexKeys returns the storage keys of the stored objects whose
	// set of indexed values for the named index includes the given
	// indexed value
	IndexKeys(indexName, indexedValue string) ([]string, error)
	// ListIndexFuncValues returns all the indexed values of the given index
	ListIndexFuncValues(indexName string) []string
	// ByIndex returns the stored objects whose set of indexed values
	// for the named index includes the given indexed value
	ByIndex(indexName, indexedValue string) ([]interface{}, error)
	// GetIndexers return the indexers
	GetIndexers() Indexers

	// AddIndexers adds more indexers to this store.  If you call this after you already have data
	// in the store, the results are undefined.
	AddIndexers(newIndexers Indexers) error
}
```

```go
// `*cache` implements Indexer in terms of a ThreadSafeStore and an
// associated KeyFunc.
type cache struct {
	// cacheStorage bears the burden of thread safety for the cache
	cacheStorage ThreadSafeStore
	// keyFunc is used to make the key for objects stored in and retrieved from items, and
	// should be deterministic.
	keyFunc KeyFunc
}

// NewStore returns a Store implemented simply with a map and a lock.
func NewStore(keyFunc KeyFunc) Store {
	return &cache{
		cacheStorage: NewThreadSafeStore(Indexers{}, Indices{}),
		keyFunc:      keyFunc,
	}
}

// NewIndexer returns an Indexer implemented simply with a map and a lock.
func NewIndexer(keyFunc KeyFunc, indexers Indexers) Indexer {
	return &cache{
		cacheStorage: NewThreadSafeStore(indexers, Indices{}),
		keyFunc:      keyFunc,
	}
}

```




## ExpirationCache

ExpirationCache相比ThreadSafeStore就是多了个有效期，比较简单。

```go
type ExpirationCache struct {
	cacheStorage     ThreadSafeStore
	keyFunc          KeyFunc
	clock            clock.Clock
	expirationPolicy ExpirationPolicy
	expirationLock sync.Mutex
}

type ExpirationPolicy interface {
	IsExpired(obj *TimestampedEntry) bool
}

// TTLPolicy implements a ttl based ExpirationPolicy.
type TTLPolicy struct {
	//	 >0: Expire entries with an age > ttl
	//	<=0: Don't expire any entry
	TTL time.Duration

	// Clock used to calculate ttl expiration
	Clock clock.Clock
}
```

## FIFO

```go
type Queue interface {
	Store

	Pop(PopProcessFunc) (interface{}, error)

	AddIfNotPresent(interface{}) error
  
	HasSynced() bool

	Close()
}

type FIFO struct {
	lock sync.RWMutex
	cond sync.Cond

	items map[string]interface{}
	queue []string

	populated bool
	initialPopulationCount int


	keyFunc KeyFunc

	closed bool
}
```

讲道理，先入先出队列应该很容理解。但是代码读起来还是有点困难，主要是： `populated`是干什么的？`Replace`是干什么的？看起来很重要的样子。



这个要从client-go的工作机制说起了, client-go刚起启动是如何自己所关注的全部资源状态呢？

**ListAndWatch** 就像MySQL的数据同步一样，新增slave节点要先从数据快照中恢复数据到某一版本(对应`List`)，然后同步binlog同步增量数据(对应`Watch`)。

所以呢，Replace其实就是为了`List`之后初始化FIFO或者Store。



有了这个背景，再看代码会容易很多。

```GO
func (f *FIFO) Pop(process PopProcessFunc) (interface{}, error) {
	f.lock.Lock()
	defer f.lock.Unlock()
	for {
		for len(f.queue) == 0 {
			// When the queue is empty, invocation of Pop() is blocked until new item is enqueued.
			if f.closed {
				return nil, ErrFIFOClosed
			}

			f.cond.Wait()
		}
    // 通过List初始化的item是不是已经处理完成，即是不是在队列的初始填充列表
		isInInitialList := !f.hasSynced_locked() 
		id := f.queue[0]
		f.queue = f.queue[1:]
		if f.initialPopulationCount > 0 {
			f.initialPopulationCount--
		}
		item, ok := f.items[id]
		if !ok {
			// Item may have been deleted subsequently.
			continue
		}
		delete(f.items, id)
		err := process(item, isInInitialList)
    // 如果处理失败了，要放回队列
		if e, ok := err.(ErrRequeue); ok {
			f.addIfNotPresent(id, item)
			err = e.Err
		}
		return item, err
	}
}
```

## DeltaFIFO

与FIFO的主要区别就是DeltaFIFO处理的Delta。

**FIFO只关心对象的最新状态`items map[string]interface{}`， 而DeltaFIFO中会保存每次变更的信息`items map[string]Deltas`。**

这是为啥呢？因为Watch返回的是变更[type+obj]，而不是Obj。所以Watch返回的数据是靠DeltaFIFO处理；

```go
func NewDeltaFIFO(keyFunc KeyFunc, knownObjects KeyListerGetter) *DeltaFIFO {
	return NewDeltaFIFOWithOptions(DeltaFIFOOptions{
		KeyFunction:  keyFunc,
		KnownObjects: knownObjects,
	})
}

// NewDeltaFIFOWithOptions returns a Queue which can be used to process changes to
// items. See also the comment on DeltaFIFO.
func NewDeltaFIFOWithOptions(opts DeltaFIFOOptions) *DeltaFIFO {
	if opts.KeyFunction == nil {
		opts.KeyFunction = MetaNamespaceKeyFunc
	}

	f := &DeltaFIFO{
		items:        map[string]Deltas{},
		queue:        []string{},
		keyFunc:      opts.KeyFunction,
		knownObjects: opts.KnownObjects,

		emitDeltaTypeReplaced: opts.EmitDeltaTypeReplaced,
		transformer:           opts.Transformer,
	}
	f.cond.L = &f.lock
	return f
}


func newInformer(
	lw ListerWatcher,
	objType runtime.Object,
	resyncPeriod time.Duration,
	h ResourceEventHandler,
	clientState Store,
	transformer TransformFunc,
) Controller {
	// This will hold incoming changes. Note how we pass clientState in as a
	// KeyLister, that way resync operations will result in the correct set
	// of update/delete deltas.
	fifo := NewDeltaFIFOWithOptions(DeltaFIFOOptions{   // 使用方式
		KnownObjects:          clientState,
		EmitDeltaTypeReplaced: true,
		Transformer:           transformer,
	})

	cfg := &Config{
		Queue:            fifo,
		ListerWatcher:    lw,
		ObjectType:       objType,
		FullResyncPeriod: resyncPeriod,
		RetryOnError:     false,

		Process: func(obj interface{}, isInInitialList bool) error {
			if deltas, ok := obj.(Deltas); ok {
				return processDeltas(h, clientState, deltas, isInInitialList)
			}
			return errors.New("object given as Process argument is not Deltas")
		},
	}
	return New(cfg)
}

```

还有个特殊的地方：knownObjects,存储的是Store,即底层是map[string]obj， Delete和Replace都会涉及knownObjects。

## Reflector

从API Server通过ListWatch读取存量资源以及增量变更同步到DeltaFIFO中。

```go
type Reflector struct {
	// name identifies this reflector. By default it will be a file:line if possible.
	name string
	// The name of the type we expect to place in the store. The name
	// will be the stringification of expectedGVK if provided, and the
	// stringification of expectedType otherwise. It is for display
	// only, and should not be used for parsing or comparison.
	typeDescription string

  // expectedType expectedGVK 其实是互斥关系，代表reflector侦听的资源类型
  // 从定义中可以看出，一个reflector只能侦听一种资源类型，比如pods；
	expectedType reflect.Type
	expectedGVK *schema.GroupVersionKind
	// The destination to sync up with the watch source
	store Store
	// listerWatcher is used to perform lists and watches.
	listerWatcher ListerWatcher
	// backoff manages backoff of ListWatch
	backoffManager wait.BackoffManager
	resyncPeriod   time.Duration
	// clock allows tests to manipulate time
	clock clock.Clock
	// paginatedResult defines whether pagination should be forced for list calls.
	// It is set based on the result of the initial list call.
	paginatedResult bool
	// lastSyncResourceVersion is the resource version token last
	// observed when doing a sync with the underlying store
	// it is thread safe, but not synchronized with the underlying store
	lastSyncResourceVersion string
	// isLastSyncResourceVersionUnavailable is true if the previous list or watch request with
	// lastSyncResourceVersion failed with an "expired" or "too large resource version" error.
	isLastSyncResourceVersionUnavailable bool
	// lastSyncResourceVersionMutex guards read/write access to lastSyncResourceVersion
	lastSyncResourceVersionMutex sync.RWMutex
	// Called whenever the ListAndWatch drops the connection with an error.
	watchErrorHandler WatchErrorHandler
	// WatchListPageSize is the requested chunk size of initial and resync watch lists.
	// If unset, for consistent reads (RV="") or reads that opt-into arbitrarily old data
	// (RV="0") it will default to pager.PageSize, for the rest (RV != "" && RV != "0")
	// it will turn off pagination to allow serving them from watch cache.
	// NOTE: It should be used carefully as paginated lists are always served directly from
	// etcd, which is significantly less efficient and may lead to serious performance and
	// scalability problems.
	WatchListPageSize int64
	// ShouldResync is invoked periodically and whenever it returns `true` the Store's Resync operation is invoked
	ShouldResync func() bool
	// MaxInternalErrorRetryDuration defines how long we should retry internal errors returned by watch.
	MaxInternalErrorRetryDuration time.Duration
	// UseWatchList if turned on instructs the reflector to open a stream to bring data from the API server.
	// Streaming has the primary advantage of using fewer server's resources to fetch data.
	//
	// The old behaviour establishes a LIST request which gets data in chunks.
	// Paginated list is less efficient and depending on the actual size of objects
	// might result in an increased memory consumption of the APIServer.
	//
	// See https://github.com/kubernetes/enhancements/tree/master/keps/sig-api-machinery/3157-watch-list#design-details
	UseWatchList bool
}
```



```go
func (r *Reflector) Run(stopCh <-chan struct{}) {
	klog.V(3).Infof("Starting reflector %s (%s) from %s", r.typeDescription, r.resyncPeriod, r.name)
	wait.BackoffUntil(func() {
		if err := r.ListAndWatch(stopCh); err != nil {
			r.watchErrorHandler(r, err)
		}
	}, r.backoffManager, true, stopCh)
	klog.V(3).Infof("Stopping reflector %s (%s) from %s", r.typeDescription, r.resyncPeriod, r.name)
}
```

### bookmarks的作用

比如client-go侦听到pods资源最新的rv 为1000，但k8s中最新资源的rv为1000100，这时候client-go watch重连时。这时候如果没有bookmark的话，client-go要从1000开始消费或者重新拉取所有资源(在1000 rv被淘汰掉),从而导致消耗大量资源来追上当前的版本。

而通过bookmark可以server端可以周期性的给client-go发送最新的rv，这样client-go重连后就可以根据bookmark定义的rv增量的消费数据。

> 直接记录watch到的最新的resource version不就可以了么?
>
> 要知道api-server给client-go同步消息的时候，就只是同步给定gvk的资源。所以如果client-go要记录最新rv的话，就要重复的通过api-server查询。

ps: 此处 `rv=ResourceVersion`

这里特别注意的是client-go重启时，确实会从api-server获取存量资源情况。

## controller/informer

封装了`reflector`，增加了ResourceEventHandler，通过`processLoop`循环调用函数`func (f *DeltaFIFO) Pop(process PopProcessFunc) (interface{}, error)`来响应事件。

```go
func newInformer(
	lw ListerWatcher,
	objType runtime.Object,
	resyncPeriod time.Duration,
	h ResourceEventHandler,
	clientState Store,
	transformer TransformFunc,
) Controller {
	// This will hold incoming changes. Note how we pass clientState in as a
	// KeyLister, that way resync operations will result in the correct set
	// of update/delete deltas.
	fifo := NewDeltaFIFOWithOptions(DeltaFIFOOptions{
		KnownObjects:          clientState,
		EmitDeltaTypeReplaced: true,
		Transformer:           transformer,
	})

	cfg := &Config{
		Queue:            fifo,
		ListerWatcher:    lw,
		ObjectType:       objType,
		FullResyncPeriod: resyncPeriod,
		RetryOnError:     false,

		Process: func(obj interface{}, isInInitialList bool) error {
			if deltas, ok := obj.(Deltas); ok {
				return processDeltas(h, clientState, deltas, isInInitialList)
			}
			return errors.New("object given as Process argument is not Deltas")
		},
	}
	return New(cfg)
}

// Run blocks; call via go.
func (c *controller) Run(stopCh <-chan struct{}) {
	defer utilruntime.HandleCrash()
	go func() {
		<-stopCh
		c.config.Queue.Close()
	}()
	r := NewReflectorWithOptions(
		c.config.ListerWatcher,
		c.config.ObjectType,
		c.config.Queue,
		ReflectorOptions{
			ResyncPeriod:    c.config.FullResyncPeriod,
			TypeDescription: c.config.ObjectDescription,
			Clock:           c.clock,
		},
	)
	r.ShouldResync = c.config.ShouldResync
	r.WatchListPageSize = c.config.WatchListPageSize
	if c.config.WatchErrorHandler != nil {
		r.watchErrorHandler = c.config.WatchErrorHandler
	}

	c.reflectorMutex.Lock()
	c.reflector = r
	c.reflectorMutex.Unlock()

	var wg wait.Group

	wg.StartWithChannel(stopCh, r.Run)

	wait.Until(c.processLoop, time.Second, stopCh)
	wg.Wait()
}
```

## SharedInformer

既然有Informer为什么还需要shared informer呢？

主要是为了实现相同资源类型reflector共享，同时有助于缓解api-server的压力。

比如两个controller都希望侦听pods资源的变更，就没必要分别维护自己的reflector和请求api-server进行listwatch。

### 具体是如何实现的呢？

SharedInformer封装了Informer,主要是通过`ProcessFunc:HandleDeltas`中把事件分发给多个`listener`实现（发到每个listener的addCh)。 然后每个listener会有个循环不停的处理数据(pop从`addCh`中读取数据，放入`nextCh`, run从`nextCh`中取数据并处理)。

```go
// Process: s.HandleDeltas
func (s *sharedIndexInformer) HandleDeltas(obj interface{}, isInInitialList bool) error {
	s.blockDeltas.Lock()
	defer s.blockDeltas.Unlock()

	if deltas, ok := obj.(Deltas); ok {
		return processDeltas(s, s.indexer, deltas, isInInitialList)
	}
	return errors.New("object given as Process argument is not Deltas")
}

// 调用ResourceEventHandler对应的梳理函数，并且更新clientState
func processDeltas(
	// Object which receives event notifications from the given deltas
	handler ResourceEventHandler,
	clientState Store,
	deltas Deltas,
	isInInitialList bool,
) error {
	// from oldest to newest
	for _, d := range deltas {
		obj := d.Object

		switch d.Type {
		case Sync, Replaced, Added, Updated:
			if old, exists, err := clientState.Get(obj); err == nil && exists {
				if err := clientState.Update(obj); err != nil {
					return err
				}
				handler.OnUpdate(old, obj)
			} else {
				if err := clientState.Add(obj); err != nil {
					return err
				}
				handler.OnAdd(obj, isInInitialList)
			}
		case Deleted:
			if err := clientState.Delete(obj); err != nil {
				return err
			}
			handler.OnDelete(obj)
		}
	}
	return nil
}

func (s *sharedIndexInformer) OnAdd(obj interface{}, isInInitialList bool) {
	s.cacheMutationDetector.AddObject(obj)
	s.processor.distribute(addNotification{newObj: obj, isInInitialList: isInInitialList}, false)
}

func (p *sharedProcessor) distribute(obj interface{}, sync bool) {
	p.listenersLock.RLock()
	defer p.listenersLock.RUnlock()

	for listener, isSyncing := range p.listeners {
		switch {
		case !sync:
			listener.add(obj)
		case isSyncing:
			listener.add(obj)
		default:
		}
	}
}

func (p *processorListener) add(notification interface{}) {
	if a, ok := notification.(addNotification); ok && a.isInInitialList {
		p.syncTracker.Start()
	}
	p.addCh <- notification
}
```



### 一些关键代码

```go
func NewSharedIndexInformerWithOptions(lw ListerWatcher, exampleObject runtime.Object, options SharedIndexInformerOptions) SharedIndexInformer {
	realClock := &clock.RealClock{}

	return &sharedIndexInformer{
		indexer:                         NewIndexer(DeletionHandlingMetaNamespaceKeyFunc, options.Indexers),
		processor:                       &sharedProcessor{clock: realClock},
		listerWatcher:                   lw,
		objectType:                      exampleObject,
		objectDescription:               options.ObjectDescription,
		resyncCheckPeriod:               options.ResyncPeriod,
		defaultEventHandlerResyncPeriod: options.ResyncPeriod,
		clock:                           realClock,
		cacheMutationDetector:           NewCacheMutationDetector(fmt.Sprintf("%T", exampleObject)),
	}
}

func (s *sharedIndexInformer) Run(stopCh <-chan struct{}) {
	defer utilruntime.HandleCrash()

	if s.HasStarted() {
		klog.Warningf("The sharedIndexInformer has started, run more than once is not allowed")
		return
	}

	func() {
		s.startedLock.Lock()
		defer s.startedLock.Unlock()

    // 创建DeltaFIFO
		fifo := NewDeltaFIFOWithOptions(DeltaFIFOOptions{
			KnownObjects:          s.indexer,
			EmitDeltaTypeReplaced: true,
			Transformer:           s.transform,
		})

		cfg := &Config{
			Queue:             fifo,
			ListerWatcher:     s.listerWatcher,
			ObjectType:        s.objectType,
			ObjectDescription: s.objectDescription,
			FullResyncPeriod:  s.resyncCheckPeriod,
			RetryOnError:      false,
			ShouldResync:      s.processor.shouldResync,

			Process:           s.HandleDeltas,
			WatchErrorHandler: s.watchErrorHandler,
		}

    // 初始化Informer
		s.controller = New(cfg)
		s.controller.(*controller).clock = s.clock
		s.started = true
	}()

	// Separate stop channel because Processor should be stopped strictly after controller
	processorStopCh := make(chan struct{})
	var wg wait.Group
	defer wg.Wait()              // Wait for Processor to stop
	defer close(processorStopCh) // Tell Processor to stop
	wg.StartWithChannel(processorStopCh, s.cacheMutationDetector.Run)
  // s.processor.run依次启动listeners成员的run和pop的后台循环
	wg.StartWithChannel(processorStopCh, s.processor.run) 

	defer func() {
		s.startedLock.Lock()
		defer s.startedLock.Unlock()
		s.stopped = true // Don't want any new listeners
	}()
	s.controller.Run(stopCh)
}
```

pop函数的代码比较难懂，这里要搞懂的话，要先搞清楚它的作用, 接收addCh的数据发送到nextCh, 如果是这样的话，那应该很简单，为啥pop这么复杂呢？
原因是多个`processorListener`在`distribute`中被依次调用，而对应的`listener.add`会写`addCh`,假如`addCh`阻塞了,就会影响其他的` listener`.
另外如果队列为空的话,就不再走`pendingNotifications`,而是直接发到`p.nextCh`.

```go
func (p *processorListener) pop() {
	defer utilruntime.HandleCrash()
	defer close(p.nextCh) // Tell .run() to stop

	var nextCh chan<- interface{}
	var notification interface{}
	for {
		select {
		case nextCh <- notification:
			// Notification dispatched
			var ok bool
			notification, ok = p.pendingNotifications.ReadOne()
			if !ok { // Nothing to pop
				nextCh = nil // Disable this select case
			}
		case notificationToAdd, ok := <-p.addCh:
			if !ok {
				return
			}
			if notification == nil { // No notification to pop (and pendingNotifications is empty)
				// Optimize the case - skip adding to pendingNotifications
				notification = notificationToAdd
				nextCh = p.nextCh
			} else { // There is already a notification waiting to be dispatched
				p.pendingNotifications.WriteOne(notificationToAdd)
			}
		}
	}
}
```



## 附1：GroupVersionKind Resource

> - The *core* (also called *legacy*) group is found at REST path `/api/v1`. The core group is not specified as part of the `apiVersion` field, for example, `apiVersion: v1`.
> -  The named groups are at REST path `/apis/$GROUP_NAME/$VERSION` and use `apiVersion: $GROUP_NAME/$VERSION` (for example, `apiVersion: batch/v1`). You can find the full list of supported API groups in [Kubernetes API reference](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.28/#-strong-api-groups-strong-).

```yaml
apiVersion: core.openfunction.io/v1beta1 # group: core.openfunction.io version: v1beta1
apiVersion: v1  # group: core version: v1 
```

`Kind` 表示对象的种类或类型，而 `Resource` 是对象 API 操作的目标。 Pod` 对象的 `Kind` 是 `Pod`，而对应的 `Resource` 是 `pods。

在执行 kubectl 命令或调用 Kubernetes API 时，操作的目标是 `Resource`（如 `pods`、`deployments` 或 `services` 等）。

```go
func getExpectedGVKFromObject(expectedType interface{}) *schema.GroupVersionKind {
	obj, ok := expectedType.(*unstructured.Unstructured)
	if !ok {
		return nil
	}

	gvk := obj.GroupVersionKind()
	if gvk.Empty() {
		return nil
	}

	return &gvk
}
```

若有格式问题或者查看后续更新，可见原文。[k8s client-go源码解析 -  1) tools/cache ](https://binnn6.github.io/posts/k8s%20client-go%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90%20-%20toolscache.html)
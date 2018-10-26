# 观察者模式又叫做序列
## 基础  
对理解RxSwift来说观察者模式和普通对列模式是相等是很重要的事。  
每一个观察对列也是一个对列。观察相比于Swift中序对列的优势是它可以异步接收元素。这是RxSwift的核心，这里的说明是我们拓展这个观点的一些方式。  

* 观察等同于对列  
* `ObservableType.subscribe`方法等同于`Sequence.makeIterator`方法 。  
*  `Observer`需要必须传递`ObservableType.subcribe`方法接收序列元素来代替在返回的迭代器上调用`next()`。  

对列是一种容易形象化的简单熟悉的概念。  
人类是拥有庞大外形的生物，当我们可以容易地想象一个概念时，这会比较容易理解它。  

通过在对列中高级操作的每一个Rx操作符上努力模拟状态机可以提升加载认知。  
  
 如果我们不用Rx而是异步模拟系统，这就意味着我们代码中被需要模仿来代替抽象方式状态机和暂时状态的代码充满。  
 一般情况下链表和序列是数学家和编程者首先学的概念。  
 一个数字序列:  
 ```
 --1--2--3--4--5--6--| //正常结束
 ```  
 
 另外一个字符序列：  
 ```
 --a--b--a--a--a---d---X // 错误结束
 ```  
 有些序列是有限的，但也有无限的，像按钮的点击序列：  
 ```
 ---tap-tap-------tap-->
 ```  
 
 它们被叫做`marble diagrams`。这里有更多的在[rxmarbles.com](http://rxmarbles.com/)。  
 如果我们以规律表达式来指定序列语法，它应该就像：  
 **next (error | completed)?**  
 
 这描述了：  
 * 序列中有0或更多个元素。  
 * 一旦错误或完成事件被接收，那序列就不能再生产其它元素。  
 
 在Rx中序列被描述为推入接口（又叫回调）。  
 
 ```
 enum Event<Element> {
 	case next(Element)   // 序列中下一个元素
 	case error(Swift.Error)  // 以错误序列失败
	case completed
 }  
 
 class Observable<Element> {
 	func subscribe(_ oberver: Observer<Element>) -> Disposable
 }
 
 protocol ObserverType {
 	func on(_ event: Event<Element>)
 }
 ```  
 
 当一个序列发送`completed`或`error`事件时所有计算序列元素的内部资源将被释放。  
 想取消序列元素生产和立即释放资源，可以在返回的订阅者上调用`dispose`。  
 如果一个序列在有限的时间消失，不调用`dispose`或不使用`disposed(by: disposeBag)`不会造成永久资源泄露。然而，这些资源一直被使用直到序列完成，另外结束生产或者返回一个错误。  
 
 如果一个序列不能自己结束，例如一连串的button点击，资源将被永久分配，除非`dispose`被手动调用，自动地在`disposeBag`中，后面跟着`takeUntil`操作符，或者其它方式。  
 使用`dispose bags`或`takerUntil`操作符是一种强健的方式来保证资源会被清除。 尽管序列可以在有限的时间内结束我们推荐在生产环境中使用它们。    
 如果你好奇`Swift.Error`为什么不是类，你在[这里](https://github.com/ReactiveX/RxSwift/blob/master/Documentation/DesignRationale.md#why-error-type-isnt-generic)可以找到解释。  
 
## Dispoding 处理  

这是被观察的序列可以停止的另外一种方式。当我们在一个序列中完成且想要释放所有被分配的资源，我们可以在订阅上处理。  

这里有个`Interval`操作符的例子。  

```
let scheduler = SerialDispatchQueueScheduler(qos: .default)
let subcription = Observable<Int>.interval(0.3, scheduler: scheduler)
            .subscribe{ event in
                print(event)
        }
        
Thread.sleep(forTimeInterval: 2.0)
        
subcription.dispose()
```
 这将会打印（我自己跑了一下）：  
 
```
next(0)
next(1)
next(2)
next(3)
next(4)
next(5)
```  
注意你通常不需要手动调用`dispose`；这只是一个教学例子。手动调用`dispose`通常是坏的代码方式。这里有更好的方式来处理订阅，例如`DisposeBag`，`takeUntil`操作符，或者其它技巧。  

 当`dispose`被执行这段代码可以打印出来东西吗？答案是：根据情况而定。  
 
 * 当`schduler`是串行调度器且dispose在相同串行调度器中被执行，就不打印。(就是`MainScheduler.instance`)
 * 其它的都打印。  
 
 你可以在[这里](https://github.com/ReactiveX/RxSwift/blob/master/Documentation/Schedulers.md)找到更多关于`scheduler`。  
 
 你拥有两个平行的进程事件。  
 
 * 一个生产元素
 * 另一个处理订阅 

"之后会打印一些东西吗？"这个问题在不同调度下还没有弄明白。  
下面这种情况：

```swift
let subcription = Observable<Int>.interval(0.3, scheduler: scheduler)
            .observeOn(MainScheduler.instance)
            .subscribe{ event in
                print(event)
        }
        
Thread.sleep(forTimeInterval: 2.0)
subcription.dispose()
```  
执行完dispose后，没有任何数据打印。  
并且，在下面例子:  

```swift
  let subcription = Observable<Int>.interval(0.3, scheduler: scheduler)
            .observeOn(scheduler)
            .subscribe{ event in
                print(event)
        }
        
        Thread.sleep(forTimeInterval: 2.0)
        
        subcription.dispose()
```
 在相同的scheduler中也不会打印。  
 
 
## Dispose Bags  处置包  
处理包用来返回类似ARC的行为给Rx。  
当一个`DisposeBags`被释放的时候，它将清除每一个额外可自由支配的。  
它没有dispose方法，因此也不允许根据目的明确调用明确的dispose。如果需要立即清除，我们只需要创建一个新包。  

```swift
self.disposeBag = DisposeBag()
```
 这将清除旧的接口，并造成资源清除。  
 如果依然想直接手动处置，用`CompositeDisposable`。**它有想要的行为但是一旦dispose被触发，它将立即处置任何新的额外可自由处置的。**   
 
## Take until  

在销毁时另外一种自动处置订阅的途径是使用`takeUntil`操作符。  

```swift
let _ = takeUntilBtn.rx
            .tap
            .takeUntil(self.rx.deallocated)
            .subscribe {
                print("test takeUntil operator \($0)")
                
        }
```  
> `takeUntil`就是将这次订阅资源绑定到另外一个信号上，另外一个信号被释放，它紧跟着释放。  
  
## Implicit `Observable` guarantees  

这里也可以额外保证是因为所有的序列生产者必须尊敬。  
它不是很在意他们在哪个线程生产元素，但是如果它们构造了一个元素并把他们发送给观察者`observer.on(.next(nextElement))`，它们在`observer.on`结束之前不会发送下一个元素。  
生产者在`.next`结束前也不能发送结束`.completed`或`.error`。   

## 创建自己的`Observable` 

这是一件很重要的事来理解observables。  
当一个observable被创建，它不会执行简单地任何工作因为它已经被创建。  

`Observable`可以以很多方式构造元素这是事实。它们中一些也会造成副作用，或者一些在正在跑的进程中触发像点击鼠标事件等。  
然而，如果你仅调用一个返回`Observable`的方法时，没有序列构造被执行，因此没有副作用。`Observable`仅定义了序列如何被构造和在构造中什么元素被使用。序列构造开始于`subcribe`方法什么时候被调用。  

比如，告诉你拥有一个带类似原型的方法：  

```swift
func searchWikipedia(searchTerm: String) -> Observable<Results> {}
```  

```swift
let searchForMe = searchWikipedia("me")
// 没有请求被执行，没有工作被做完，没有url请求被启动  
let cancel = searchForMe.subscribe(onNext: { results in 
	print(results)
})
```  

这里有很多方式可以创建自己的可观察序列，最简单的方式还是使用`create`方法。  
RxSwift提供一个创建返回一个被订阅元素的序列的方法。这个方法叫做`just`。让我们自己实现它。

```swift
func myJust<E>(_ element: E) -> Observable<E> {
        return Observable.create { observer in
            observer.on(.next(element))
            observer.on(.completed)
            return Disposables.create()
        }
 }
 
  let _ = myJust(0).subscribe(onNext: { n in
            print(n)
        })
```  

不错。所以`create`方法是什么？  
它仅是一个便利方法能让你轻松地使用swift中的闭包来实现subscribe方法。像subscrible方法它有一个参数，observer，和返回disposable。  

这种方式实现的序列确实是异步的，它在`subcribe`回调返回disposable订阅之前将构造元素和结束。因为它不在意是哪个disposable返回，构造元素的进程不会被打断。  
当构建异步序列时，通常返回的disposable是`NopDisposable`的单例。  

现在让我们创建一个从数组中返回元素的observable。  

```swift
func myFrom<E>(_ sequence: [E]) -> Observable<E> {
    return Observable.create { observer in
        for element in sequence {
            observer.on(.next(element))
        }

        observer.on(.completed)
        return Disposables.create()
    }
}

let stringCounter = myFrom(["first", "second"])

print("Started ----")

// first time
stringCounter
    .subscribe(onNext: { n in
        print(n)
    })

print("----")

// again
stringCounter
    .subscribe(onNext: { n in
        print(n)
    })

print("Ended ----")
```  

## Creating an Observable that performs work 


 好，现在一些事开始有趣了。让我们在上一个例子中创建`interval`操作符。  
 
 对于分配序列调度器这个等价于实际实现。  
 
 ```swift
 
 ```
在订阅上每一个订阅者通常构建元素它们自己的单独序列。操作者默认没有区别的，相比状态机这里有更多无限制的。  

## Sharing subscription and share operator  

但是从一个订阅上面多个订阅者分享事件怎么办？  

这里有两件事需要被定义：  
* 在一个新的订阅者感兴趣观察之前如何处理传递元素  
* 如何决定什么时间来启动这个分享的订阅  

通常做法是`replay(1).refCount()`的混合操作，又叫做`share(replay: 1)`。  

```swift
let counter = myInterval(0.1).share(replay: 1)
        
        print("Started ---- ")
        
        let subscription = counter.subscribe(onNext: { (n) in
            print("First: \(n)")
        })
        
        let subscription2 = counter.subscribe(onNext: { (n) in
            print("Second: \(n)")
        })
        
        Thread.sleep(forTimeInterval: 0.5)
        
        subscription.dispose()
        
        Thread.sleep(forTimeInterval: 0.5)
        
        subscription2.dispose()
        
        print("Ended ---- ")
``` 

## Operators

在RxSwift中有很多被实现的操作符。 

所有操作符的单子图表可以在[ReactiveX.io](http://reactivex.io/)找到。  

几乎所有操作符在[Playgrounds](https://github.com/ReactiveX/RxSwift/blob/master/Rx.playground)中演示。 

使用playgrounds请先打开`Rx.xcworkspace`，编译`RxSwift-macOS`，然后在结构中打开playgrounds。  

[操作符使用](http://reactivex.io/documentation/operators.html#tree)  

### Custom operators  

这里有两种方式可以创建自定义操作符。  

### Easy way 

所有的内部代码使用操作符的高度优化版本，所以他们不是最好的指导资料。这也是为什么建议使用标准操作符。  

幸运地这里有容易的方法来创建操作符。创建一个新的操作符实际上创建observables和之前章节讲到的如何做它。  

看一下未优化的map操作符如何被实现：

```swift
extension ObservableType {
    func myMap<R>(transform: @escaping (E) -> R) -> Observable<R> {
        return Observable.create { observer in
            let subscription = self.subscribe { e in
                switch e {
                case .next(let value):
                    let result = transform(value)
                    observer.on(.next(result))
                case .error(let error):
                    observer.on(.error(error))
                case .completed:
                    observer.on(.completed)
                }
            }
            return subscription
        }
    }
}
```    

可以这样使用：

```swift
 let subscription = myInterval(0.1).myMap { (e) in
            return "This is simply \(e)"
            }.subscribe(onNext: { n in
                print(n)
            })
        
        Thread.sleep(forTimeInterval: 1)
        
        subscription.dispose()
```  

### Life happens  

如果觉得处理这些自定义操作符的情况比较困难？你可以退出Rx原子操作，在重要块中执行方法，然后使用`Subject`S将结果返回给Rx。  

可以对这件事不熟练，这也是不好的代码，但是你能这样做。  
不建议这样做。  

## Playgrounds  

## Error handling  





 
 
 
	 
	
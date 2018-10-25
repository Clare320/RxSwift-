# 观察者模式又叫做队列
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
 一般情况下链表和队列是数学家和编程者首先学的概念。  
 一个数字队列:  
 ```
 --1--2--3--4--5--6--| //正常结束
 ```  
 
 另外一个字符队列：  
 ```
 --a--b--a--a--a---d---X // 错误结束
 ```  
 有些队列是有限的，但也有无限的，像按钮的点击队列：  
 ```
 ---tap-tap-------tap-->
 ```  
 
 它们被叫做`marble diagrams`。这里有更多的在[rxmarbles.com](http://rxmarbles.com/)。  
 如果我们以规律表达式来指定队列语法，它应该就像：  
 **next (error | completed)?**  
 
 这描述了：  
 * 队列中有0或更多个元素。  
 * 一旦错误或完成事件被接收，那队列就不能再生产其它元素。  
 
 在Rx中队列被描述为推入接口（又叫回调）。  
 
 ```
 enum Event<Element> {
 	case next(Element)   // 队列中下一个元素
 	case error(Swift.Error)  // 以错误队列失败
	case completed
 }  
 
 class Observable<Element> {
 	func subscribe(_ oberver: Observer<Element>) -> Disposable
 }
 
 protocol ObserverType {
 	func on(_ event: Event<Element>)
 }
 ```  
 
 当一个队列发送`completed`或`error`事件时所有计算队列元素的内部资源将被释放。  
 想取消队列元素生产和立即释放资源，可以在返回的订阅者上调用`dispose`。  
 如果一个队列在有限的时间消失，不调用`dispose`或不使用`disposed(by: disposeBag)`不会造成永久资源泄露。然而，这些资源一直被使用直到队列完成，另外结束生产或者返回一个错误。  
 
 如果一个队列不能自己结束，例如一连串的button点击，资源将被永久分配，除非`dispose`被手动调用，自动地在`disposeBag`中，后面跟着`takeUntil`操作符，或者其它方式。  
 使用`dispose bags`或`takerUntil`操作符是一种强健的方式来保证资源会被清除。 尽管队列可以在有限的时间内结束我们推荐在生产环境中使用它们。    
 如果你好奇`Swift.Error`为什么不是类，你在[这里](https://github.com/ReactiveX/RxSwift/blob/master/Documentation/DesignRationale.md#why-error-type-isnt-generic)可以找到解释。  
 
## Dispoding 处理  

这是被观察的队列可以停止的另外一种方式。当我们在一个队列中完成且想要释放所有被分配的资源，我们可以在订阅上处理。  

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

 
 
 
 
	 
	
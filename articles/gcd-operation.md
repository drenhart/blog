## iOS多线程总结

### GCD

- 可用于多核并行运算
- 自动管理线程生命周期

### 同步与异步区别

1. 是否等待队列的任务执行结束,
2. 是否具备开启新线程的能力

### 串行队列与并发队列区别

![serial](https://upload-images.jianshu.io/upload_images/1877784-4faca27116209f35.png)

![concurrent](https://upload-images.jianshu.io/upload_images/1877784-97f3931d1b187b11.png)

### 同步异步与队列组合

| 区别        | 并发队列                     | 串行队列                     | 主队列                     |
| ----------- | ---------------------------- | ---------------------------- | -------------------------- |
| 同步(sync)  | 不开启新线程, 串行执行任务   | 不开启新线程, 串行执行任务   | 死锁卡住不执行             |
| 异步(async) | 可以开启新线程, 并发执行任务 | 开启一条新线程, 串行执行任务 | 不开启新线程, 串行执行任务 |

### 死锁问题

> 追加任务与原有任务互相等待造成程序无法完成的情况称为死锁

**例1**

syncMain和任务1互相等待造成死锁

```swift
    func syncMain() {
        let queue = DispatchQueue.main
        queue.sync {
          	// 任务1
            Thread.sleep(forTimeInterval: 2)
            print("\(Thread.current)")
        }
    }
```

在其他线程中调用就不会死锁


```swift
	 Thread.detachNewThreadSelector(#selector(syncMain), toTarget: self, with: nil)
```

**例2**

串行队列异步套同步。串行队列中原有任务和追加任务互相等待

```swift
				let queue = DispatchQueue(label: "serial")
        queue.async { //  串行队列 异步执行
            queue.sync { // 串行队列 同步执行
                sleep(2);
                print("A-- \(Thread.current)")
            }
        }
```

### 信号量

> 小于0阻塞, 大于等于0通过.

**应用1：同步线程**

```swift
		func semaphoreSynchronize() {
        let semaphore = DispatchSemaphore(value: 0)
        var number = 0
        let queue = DispatchQueue.global()
        queue.async {
            sleep(2)
            print("A-- \(Thread.current)")
            
            number = 100
            semaphore.signal() // +1
        }
        
        semaphore.wait(timeout: .now() + 5) // -1
      	print(number) // 100
    }
```

**应用2：加锁**

```swift
		var ticketCount = 50
    let semaphore = DispatchSemaphore(value: 1)
    func semaphoreThreadSafe() {
        let queue1 = DispatchQueue(label: "concurrent1", attributes: .concurrent)
        queue1.async {
            self.saleTicket()
        }
        let queue2 = DispatchQueue(label: "concurrent2", attributes: .concurrent)
        queue2.async {
            self.saleTicket()
        }
    }
    
    func saleTicket() {
        while (true) {
            semaphore.wait()
            if (ticketCount > 0) {
                ticketCount -= 1;
                print("\(ticketCount)----\(Thread.current)")
                Thread.sleep(forTimeInterval: 0.2)
            } else {
                print("end")
                semaphore.signal()
                break
            }
            semaphore.signal()
        }
    }
```

### barrier

> 分割异步线程

```swift
			  queue.async(flags: .barrier) {
            sleep(2)
            print("barrier-- \(Thread.current)")
        }
```

### Operation

> 虽然Operation是GCD的封装，但是Operation有一些GCD不具备的功能，比如说设置最大并发数，queue的暂停与恢复，Operation的子类化等。

### 面试题

**ABC并发执行, 之后执行D?**

```swift
			  // gcd
				let group = DispatchGroup()
        DispatchQueue.global().async(group: group) {
            sleep(2)
            print("A-- \(Thread.current)")
        }
        DispatchQueue.global().async(group: group) {
            sleep(2)
            print("B-- \(Thread.current)")
        }
        DispatchQueue.global().async(group: group) {
            sleep(2)
            print("C-- \(Thread.current)")
        }
        group.notify(queue: DispatchQueue.main) {
            sleep(2);
            print("D-- \(Thread.current)")
        }
```

```swift
				// Operation
				let queue = OperationQueue()
        let operation = BlockOperation {
            print("A-- \(Thread.current)")
        }
        operation.addExecutionBlock {
            print("B-- \(Thread.current)")
        }
        operation.addExecutionBlock {
            print("C-- \(Thread.current)")
        }
        let operation2 = BlockOperation {
            print("D-- \(Thread.current)")
        }
        operation2.addDependency(operation)
        queue.addOperation(operation)
        queue.addOperation(operation2)
```

**代码的执行顺序？**

> 字节面试真题，顺序应为4->2->1->3

```swift
			DispatchQueue.main.async {
				DispatchQueue.main.async{print(1)}
				print(2)
				DispatchQueue.main.async{print(3)}
			}
			print(4)
```

> 洋钱罐面试真题，顺序应为1->5->2->死锁

```swift
			let queue = DispatchQueue(label: "serial")
			print(1)
			queue.async {
    			print(2)
    			queue.sync {
        			print(3)
    			}
    			print(4)
			}
			print(5)
```

**最大并发数控制?**

> 米可世界面试真题。并发数设置为3，D和E会等待ABC执行完再执行

```swift
        let queue = OperationQueue()
        queue.maxConcurrentOperationCount = 3
        let operation1 = BlockOperation {
            print("A")
        }
        let operation2 = BlockOperation {
            print("B")
        }
        let operation3 = BlockOperation {
            print("C")
        }
        let operation4 = BlockOperation {
            print("D")
        }
        let operation5 = BlockOperation {
            print("E")
        }
        queue.addOperation(operation1)
        queue.addOperation(operation2)
        queue.addOperation(operation3)
        queue.addOperation(operation4)
        queue.addOperation(operation5)
```

**主队列与主线程关系?**

> 陌陌面试真题。

默认情况下，代码都是放在主队列中的，主队列中的代码又都会放到主线程中去执行。

### 参考

1. [GCD详尽总结](https://www.jianshu.com/p/2d57c72016c6)
2. [iOS 多线程编程总结](https://github.com/bestswifter/blog/blob/master/articles/multi-thread-conclusion.md)






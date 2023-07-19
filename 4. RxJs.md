### 什么是 `RxJs`

`RxJs` 使用的是观察者模式，用来编写异步队列和事件处理。

- `Observable` 可观察的物件

- `Subscription` 监听 `Observable`

-  `Operators` 纯函数可以处理管道的数据 如 `map` 、 `filter`、 `concat` 、 `reduce` 等



> `rxjs` 是 `nest` 内置的一个库，如果需要请查看[文档](https://cn.rx.js.org/)。



**发布订阅案例**：

```tsx
import { Observable } from "rxjs";
 
// 类似于迭代器 next 发出通知  complete 通知完成

// 可观察的物件
const observable = new Observable(subscriber=>{
    // 发出通知
    subscriber.next(1)
    subscriber.next(2)
    subscriber.next(3)
 
    setTimeout(()=>{
        subscriber.next(4)
        subscriber.complete()
    },1000)
})

// 订阅通知
observable.subscribe({
    // 每一步都可以检测到
    next:(value)=>{
       console.log(value)
    }
})
```



##### 定时监听案例：

```tsx
import { interval } from 'rxjs';
import { map, filter } from 'rxjs/operators';

// interval 500 毫秒执行一次 pipe (就是管道的意思)
// 管道里面也是可以去调接口, 支持处理异步数据
const subs = interval(500)
  .pipe(
    map((v) => ({ num: v })),
    filter((v) => v.num % 2 == 0),
  )
  // 通过观察者 subscribe 接受回调
  .subscribe((e) => {
    console.log(e);
    // 结束条件
    if (e.num == 10) {
      subs.unsubscribe();
    }
  });
```



##### 处理事件案例：

`Rxjs` 也可以处理事件, 不过我们在 `nest` 里面不用操作 `DOM` , 你如果使用 `Angular` 或 `Vue` 等框架, 可以使用 `fromEvent`

```tsx
import { fromEvent } from "rxjs";
import { map } from 'rxjs/operators'
 
const dom = fromEvent(document,'click').pipe(map(e=>e.target))
dom.subscribe((e)=>{
 
})
```


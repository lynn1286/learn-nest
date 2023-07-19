`Nest` 的实现中最核心的设计思想：

- 控制反转（IOC）
- 依赖注入（DI）



通过简单的代码示例，带大家了解其中的原理。



一场大型的演唱会，邀请了众多的表演嘉宾，使用代码演示，应该是这样的：

```tsx
// 薛之谦 - 绅士
class JokerXue {
  acting() {
    console.log('好久没见了什么角色呢...')
  }
}

// 舞台
class Stage {
	private jokerXue: JokerXue = new JokerXue()

  performance() {
    this.jokerXue.acting()
  }
}

// 直播间
class Studio {
  start() {
    const stage = new Stage()
    stage.performance()
  }
}

const studio = new Studio()
studio.start()
```

首先上场表演的是薛之谦，一首歌结束后，是不是轮到周杰伦上场了，那么我们更改代码如下：

```tsx
// 周杰伦 - 晴天
class Jay {
  acting() {
    console.log(
      '从前从前有个人爱你很久，但偏偏风渐渐把距离吹得好远，好不容易又能再多爱一天，但故事的最后你好像 ...'
    )
  }
}

// 舞台
class Stage {
	private jay: Jay = new Jay()

  performance() {
    this.jay.acting()
  }
}

// 直播间
class Studio {
  start() {
    const stage = new Stage()
    stage.performance()
  }
}

const studio = new Studio()
studio.start()
```

现在我们发现了一个问题，没结束一个演唱，我们都需要更改一下代码，很不友好，我们将代码改进下，变成：

```tsx
// 定义一个演员接口
interface Actor {
  acting: () => void
}

// 薛之谦
class JokerXue implements Actor {
  // 演唱 绅士
  gentleman() {
    console.log('好久没见了什么角色呢...')
  }

  acting() {
    this.gentleman()
  }
}

// 周杰伦
class Jay implements Actor {
  // 演唱 晴天
  sunny() {
    console.log(
      '从前从前有个人爱你很久，但偏偏风渐渐把距离吹得好远，好不容易又能再多爱一天，但故事的最后你好像 ...'
    )
  }
  acting() {
    this.sunny()
  }
}

// 舞台
class Stage {
  // private actor: JokerXue = new JokerXue()
  private actor: Jay = new Jay()

  performance() {
    this.actor.acting()
  }
}

// 直播间
class Studio {
  start() {
    const stage = new Stage()
    stage.performance()
  }
}

const studio = new Studio()
studio.start()
```

我们新增一个抽象接口，它拥有一个 `acting` 方法，每个具体的表演嘉宾实现这个接口的方法，这样，我们就可以实现快速切换，不过这还不是完全遵守了依赖倒置原则，所以，我们还要再对代码进行修改：

```tsx
interface Actor {
  acting: () => void
}

// 薛之谦
class JokerXue implements Actor {
  // 演唱 绅士
  gentleman() {
    console.log('好久没见了什么角色呢...')
  }

  acting() {
    this.gentleman()
  }
}

// 周杰伦
class Jay implements Actor {
  // 演唱 晴天
  sunny() {
    console.log(
      '从前从前有个人爱你很久，但偏偏风渐渐把距离吹得好远，好不容易又能再多爱一天，但故事的最后你好像 ...'
    )
  }
  acting() {
    this.sunny()
  }
}

// 舞台
class Stage {
  private actor: Actor
  // 通过构造函数注入
  constructor(actor: Actor) {
    this.actor = actor
  }

  performance() {
    this.actor.acting()
  }
}

// 直播间
class Studio {
  start() {
    // const actor = new JokerXue()
    const actor = new Jay()
    const stage = new Stage(actor)
    stage.performance()
  }
}

const studio = new Studio()
studio.start()
```

通过构造函数，将依赖注入给`Stage`类， 只需要在演出结束，直播间自动切换就可以换人啦。



### 总结：

因为这些设计模式，可以将类之间的强依赖关系解耦，在`Nest`的设计中遵守了控制反转的思想，使用依赖注入（包括构造函数注入、参数注入、Setter方法注入）解藕了`Controller`与`Provider`之间的依赖。
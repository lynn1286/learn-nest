### 什么是 `metadata`（元数据）

> 元数据在开发中是一个很常见的概念，意思是描述数据的数据（Data that describes other data）。比如拍了一张照片，我们只关心画面好不好看，但是摄影师会关注曝光，光圈，白平衡等信息，这些就是照片的元数据。



开发中元数据通常是和装饰器联系在一起使用的，在 `Nestjs` 中大量使用了装饰器，也会通过元数据给类或方法注入一些信息，然后通过反射器来获取。

> 声明： 本文所有代码都运行在 `Typescript` 环境



### 如何开始`metadata`编程

 `TypeScript` 已经完整的实现了装饰器，只需要开启两个配置`tsconfig.json`: 

```json
{
    "compilerOptions": {
        "target": "ES5",
        "experimentalDecorators": true,
        "emitDecoratorMetadata": true
    }
}
```

接着我们安装 `reflect-metadata` ：

```bash
npm i reflect-metadata --save
```

项目地址： https://github.com/rbuckton/reflect-metadata



基础使用：

```js
import 'reflect-metadata'

@Reflect.metadata('name', 'Person')
class Person {
  @Reflect.metadata('words', 'method speak hello world')
  public speak(): string {
    return 'hello world'
  }
}

const p = Reflect.getMetadata('name', Person)

const m = Reflect.getMetadata('words', new Person(), 'speak')

console.log({ p, m }) // { p: 'Person', m: 'method speak hello world' }

```



### Reflect API 清单

- Reflect.decorate: 定义元数据的装饰器
- Reflect.metadata：定义元数据的装饰器
- Reflect.defineMetadata：定义元数据
- Reflect.hasMetadata：判断元数据是否存在，会沿着原型链查找
- Reflect.hasOwnMetadata：判断 target 自身是否存在元数据
- Reflect.getMetadata：获取元数据，会沿着原型链查找
- Reflect.getOwnMetadata：只从 target 获取元数据
- Reflect.getMetadataKeys：获取所有元数据的 Key
- Reflect.getOwnMetadataKeys：只获取 target 身上的 key
- Reflect.deleteMetadata：删除元数据



#### Reflect.decorate

>  这个方法有多个重载



- 对类的装饰

```js
// 要引入这个库
import 'reflect-metadata'

const classDecorator: ClassDecorator = target => {
  target.prototype.sayName = () => console.log('override name')

  // target.name = 'xing'  -> error: read-only

  // return target 这里可以return也可以不return, 因为target是一个对象引用
}

export class TestClassDecrator {
  constructor(public name = '') {}

  sayName() {
    console.log(this.name)
  }
}

Reflect.decorate([classDecorator], TestClassDecrator) // 对其进行装饰

const t = new TestClassDecrator('lynn')

t.sayName() // 'override name'
```

注意： 在classDecorator中传入的target, 只能修改其prototype的方法, 不能修改其属性, 因为其属性是`read-only`



- 对属性的装饰

```js
const propertyDecorator: PropertyDecorator = (target, propertyKey) => {
  const origin = target[propertyKey]

  target[propertyKey] = () => {
    origin.call(target)
    console.log('added override')
  }
}

class PropertyAndMethodExample {
  static staticProperty() {
    console.log('im static property')
  }

  method() {
    console.log('im one instance method')
  }
}

Reflect.decorate([propertyDecorator], PropertyAndMethodExample, 'staticProperty')
// test property decorator
PropertyAndMethodExample.staticProperty() // im static property \n added override
```



- 对方法的装饰

```js
const methodDecorator: MethodDecorator = (target, propertyKey, descriptor) => {
  // 将其描述改为不可编辑
  descriptor.configurable = false
  descriptor.writable = false
  return descriptor
}

class PropertyAndMethodExample {
  static staticProperty() {
    console.log('im static property')
  }

  method() {
    console.log('im one instance method')
  }
}

// 获取原descriptor
let descriptor = Object.getOwnPropertyDescriptor(PropertyAndMethodExample.prototype, 'method')

// 获取修改后的descriptor
descriptor = Reflect.decorate([methodDecorator], PropertyAndMethodExample, 'method', descriptor)

// 将修改后的descriptor添加到对应的方法上
Object.defineProperty(PropertyAndMethodExample.prototype, 'method', descriptor)

// test method decorator
const example = new PropertyAndMethodExample()

example.method = () => console.log('override') // 报错 Cannot assign to read only property 'method' of object '#<PropertyAndMethodExample>'
```



#### Reflect.metadata

> 默认的元数据装饰器可以被用于类, 类成员以及参数.

**注意**: 如果 `metadataKey` 已经被定义在target或target key, 那么`metadataValue`将会被覆盖。

```js
const nameSymbol = Symbol('lorry')
// 类元数据
@Reflect.metadata('class', 'class')
class MetaDataClass {
  // 实例属性元数据
  @Reflect.metadata(nameSymbol, 'nihao')
  public name = 'origin'
  // 实例方法元数据
  @Reflect.metadata('getName', 'getName')
  public getName() {}
  // 静态方法元数据
  @Reflect.metadata('static', 'static')
  static staticMethod() {}
}

const value = Reflect.getMetadata('name', MetaDataClass)

const metadataInstance = new MetaDataClass()

const name = Reflect.getMetadata(nameSymbol, metadataInstance, 'name')

const methodVal = Reflect.getMetadata('getName', metadataInstance, 'getName')

const staticVal = Reflect.getMetadata('static', MetaDataClass, 'staticMethod')

console.log(value, name, methodVal, staticVal)

```



#### Reflect.defineMetadata

> 该方法是`metadata`的定义版本, 也就是非装饰器版本, 会多传一个参数`target`, 表示待装饰的对象

```js
class DefineMetadata {
  static staticMethod() {}
  static staticProperty = 'static'
  getName() {}
}

const type = 'type'
Reflect.defineMetadata(type, 'class', DefineMetadata)
const t1 = Reflect.getMetadata(type, DefineMetadata)

Reflect.defineMetadata(type, 'staticMethod', DefineMetadata.staticMethod)
const t2 = Reflect.getMetadata(type, DefineMetadata.staticMethod)

// t2 的另外一种写法
// Reflect.defineMetadata(type, 'staticMethos', DefineMetadata, 'staticMethod')
// const t2 = Reflect.getMetadata(type, DefineMetadata, 'staticMethod')

// 这两种语法不能混合使用，否则是取不到 metadataValue 的， 例如：
// Reflect.defineMetadata(type, 'staticMethos', DefineMetadata, 'staticMethod')
// const t2 = Reflect.getMetadata(type, DefineMetadata.staticMethod)

Reflect.defineMetadata(type, 'method', DefineMetadata.prototype.getName)
const t3 = Reflect.getMetadata(type, DefineMetadata.prototype.getName)

Reflect.defineMetadata(type, 'staticProperty', DefineMetadata, 'staticProperty')
const t4 = Reflect.getMetadata(type, DefineMetadata, 'staticProperty')

console.log(t1, t2, t3, t4) // class staticMethod method staticProperty

```



#### Reflect.hasMetadata

> 该方法返回布尔值, 表明该target或其原型链上有没有对应的元数据

```js
const type = 'type'
class HasMetadataClass {
  @Reflect.metadata(type, 'staticProperty')
  static staticProperty = ''
}
Reflect.defineMetadata(type, 'class', HasMetadataClass)
const t1 = Reflect.hasMetadata(type, HasMetadataClass)
const t2 = Reflect.hasMetadata(type, HasMetadataClass, 'staticProperty')
console.log(t1, t2)
```



#### Reflect.hasOwnMetadata

> 跟`Object.prototype.hasOwnProperty`类似, 是只查找对象上的元数据, 而不会继续向上查找原型链上的, 其余的跟`hasMetadata`一致

```js
const type = 'type'
class Parent {
  @Reflect.metadata(type, 'getName')
  getName() {}
}
@Reflect.metadata(type, 'class')
class HasOwnMetadataClass extends Parent {
  @Reflect.metadata(type, 'static')
  static staticProperty() {}
  @Reflect.metadata(type, 'method')
  method() {}
}

const t1 = Reflect.hasOwnMetadata(type, HasOwnMetadataClass)
const t2 = Reflect.hasOwnMetadata(type, HasOwnMetadataClass, 'staticProperty')
const t3 = Reflect.hasOwnMetadata(type, HasOwnMetadataClass.prototype, 'method')
const t4 = Reflect.hasOwnMetadata(type, HasOwnMetadataClass.prototype, 'getName')

// 在原型链上找到了
const t5 = Reflect.hasMetadata(type, HasOwnMetadataClass.prototype, 'getName')

console.log(t1, t2, t3, t4, t5) // true true true false true
```



#### Reflect.getMetadata

> 这个属性在之前验证各个属性的时候就已经使用过了, 就是用于获取target的元数据值, 会往原型链上找



#### Reflect.getOwnMetadata

> 与hasOwnMetadata和hasMetadata的区别一样, 不会往原型链上找



#### Reflect.getMetadataKeys

> 类似Object.keys, 返回该target以及原型链上target的所有元数据的keys

```js
const type = 'type'
@Reflect.metadata('parent', 'parent')
class Parent {
  getName() {}
}
@Reflect.metadata(type, 'class')
class HasOwnMetadataClass extends Parent {
  @Reflect.metadata(type, 'static')
  static staticProperty() {}
  @Reflect.metadata('bbb', 'method')
  @Reflect.metadata('aaa', 'method')
  method() {}
}

const t1 = Reflect.getMetadataKeys(HasOwnMetadataClass)
const t2 = Reflect.getMetadataKeys(HasOwnMetadataClass.prototype, 'method')
console.log(t1, t2) // ["type", "parent"] \n ["design:returntype", "design:paramtypes", "design:type", "aaa", "bbb"]
```

`t2`的输出多了三个`key`, 这是 `Typescript` 开启了 `emitDecoratorMetadata` 后增加的内置元数据。

- `design:type`: 表示被装饰的对象是什么类型, 比如是字符串? 数字?还是函数等
- `design:paramtypes`: 表示被装饰对象的参数类型, 是一个表示类型的数组, 如果不是函数, 则没有该key
- `design:returntype`: 表示被装饰对象的返回值属性, 比如字符串,数字或函数等



#### Reflect.getOwnMetadataKeys

> 跟getMetadataKeys 一样, 只是不向原型链中查找



#### Reflect.deleteMetadata

> 用于删除元数据,该方法返回布尔值, 表明是否删除成功

```
const type = 'type'
@Reflect.metadata(type, 'class')
class DeleteMetadata {
  @Reflect.metadata(type, 'static')
  static staticMethod() {}
}

const res1 = Reflect.deleteMetadata(type, DeleteMetadata)
const res2 = Reflect.deleteMetadata(type, DeleteMetadata, 'staticMethod')
const res3 = Reflect.deleteMetadata(type, DeleteMetadata)
console.log(res1, res2, res3) // true true false
```


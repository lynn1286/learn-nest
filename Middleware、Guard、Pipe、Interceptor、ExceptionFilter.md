### AOP （面向切面编程）

AOP 是什么意思呢？什么是面向切面编程呢？

不侵入业务的场景下，透明的加入业务逻辑，实现日志记录、权限控制、异常处理等，并且可以复用，动态增删。

而 `nest` 实现 `AOP` 的方式一共有五种，

-  `Middleware`
- `Guard`
- `Pipe`
- `Inteceptor`
- `ExceptionFilter`



#### Middleware

默认情况下，`nest` 中间件等同于 `Express` 中间件

以下是从 Express 官方文档中复制的中间件功能列表:

- 执行任何代码。
- 对请求和响应对象进行更改。
- 结束请求-响应周期。
- 调用堆栈中的下一个中间件函数。
- 如果当前中间件功能没有结束请求-响应周期，它必须调用 `next()` 将控制权传递给下一个中间件功能。否则，请求将被搁置。



##### Nest 中间件预览

`nest` 中间件可以是一个函数，也可以是一个带有 `@Injectable()` 装饰器的类，且该类应该实现 `NestMiddleware` 接口，而函数没有任何特殊要求。如下是一个日志中间件的简单示例：

```ts
import { Inject, Injectable, NestMiddleware } from '@nestjs/common';
import { Request, Response } from 'express';

@Injectable()
export class CatsMiddleware implements NestMiddleware {
  use(req: Request, res: Response, next: () => void) {
    console.log('🚀 ~  : CatsMiddleware -> use -> before');
    next();
    console.log('🚀 ~  : CatsMiddleware -> use -> after');
  }
}
```

##### 中间件中的依赖注入

与提供者（Provider）和控制器（Controller）一样，它能够通过构造函数注入属于同一模块的依赖项：

```tsx
import { Inject, Injectable, NestMiddleware } from '@nestjs/common';
import { Request, Response } from 'express';
import { CatsService } from './cats.service';

@Injectable()
export class CatsMiddleware implements NestMiddleware {
  // 依赖注入
  constructor(@Inject(CatsService) private readonly catsService: CatsService) {}

  use(req: Request, res: Response, next: () => void) {
    console.log(
      '🚀 ~  : CatsMiddleware -> use -> this.catsService.findAll()',
      this.catsService.findAll(),
    );

    console.log('🚀 ~  : CatsMiddleware -> use -> before');
    next();
    console.log('🚀 ~  : CatsMiddleware -> use -> after');
  }
}
```

##### 函数中间件

```tsx
import { MiddlewareConsumer, Module, NestModule } from '@nestjs/common';
import { DogsService } from './dogs.service';
import { DogsController } from './dogs.controller';

@Module({
  controllers: [DogsController],
  providers: [DogsService],
  exports: [DogsService],
})
export class DogsModule implements NestModule {
  configure(consumer: MiddlewareConsumer) {
    consumer
      .apply(function logger(req, res, next) {
        console.log(`Request...`);
        next();
      })
      .forRoutes('dogs');
  }
}
```

##### 全局中间件

为了将中间件一次绑定到每个注册的路由，我们可以使用 `INestApplication` 实例中的 `use()` 方法:

```ts
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  // 开始监听关闭钩子
  app.enableShutdownHooks();

  // 这里必须使用函数中间件
  app.use(function logger(req, res, next) {
    console.log(`global Request...`);
    next();
  });

  await app.listen(3000);
}

bootstrap();
```


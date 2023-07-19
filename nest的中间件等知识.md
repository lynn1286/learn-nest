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



#### Guard

守卫有一个单独的责任。它们根据运行时出现的某些条件（例如权限，角色，访问控制列表等）来确定给定的请求是否由路由处理程序处理。这通常称为授权。在传统的 `Express` 应用程序中，通常由中间件处理授权(以及认证)。中间件是身份验证的良好选择，因为诸如 `token` 验证或添加属性到 `request` 对象上与特定路由(及其元数据)没有强关联。

中间件不知道调用 `next()` 函数后会执行哪个处理程序。另一方面，守卫可以访问 `ExecutionContext` 实例，因此确切地知道接下来要执行什么。它们的设计与异常过滤器、管道和拦截器非常相似，目的是让您在请求/响应周期的正确位置插入处理逻辑，并以声明的方式进行插入。这有助于保持代码的简洁和声明性。

> 守卫在每个中间件之后执行，但在任何拦截器或管道之前执行。



守卫要求实现`CanActivate`函数 给定参数`context`执行上下文 , 要求返回布尔值:

```tsx
import { CanActivate, ExecutionContext, Injectable } from '@nestjs/common';
import { Observable } from 'rxjs';

@Injectable()
export class DogsGuard implements CanActivate {
  // 实现 canActivate 接口
  canActivate(
    context: ExecutionContext,
  ): boolean | Promise<boolean> | Observable<boolean> {
    return true;
  }
}

```

在控制器使用路由守卫：

```tsx
import { Controller, Get, UseGuards } from '@nestjs/common';
import { DogsGuard } from './dogs.guard';

@Controller('dogs')
// 使用路由守卫
@UseGuards(DogsGuard)
export class DogsController {
  @Get()
  // 方法绑定守卫
  // @UseGuards(DogsGuard)
  findAll() {
    return 'hello dog';
  }
}

```

##### 全局守卫

1. 如果要注册全局守卫，只需要在 `main.ts` 中注册即可:

```tsx
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';
import { GlobalGuard } from './global/global.guard';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  // 注册全局守卫
  app.useGlobalGuards(new GlobalGuard());

  await app.listen(3000);
}

bootstrap();

```

2. 再任意一个模块中注册守卫，全部控制器（全局）都会经过这个守卫逻辑。

```tsx
import { Module } from '@nestjs/common';
import { AdminService } from './admin.service';
import { AdminController } from './admin.controller';
import { AdminGuard } from './admin.guard';
import { APP_GUARD } from '@nestjs/core';

@Module({
  controllers: [AdminController],
  providers: [
    AdminService,
    {
      provide: APP_GUARD,
      useClass: AdminGuard,
    },
  ],
})
export class AdminModule {}

```

注意： 守卫注册分别有全局、类和方法三种方式，执行过程的优先级为： 全局 -> 控制器 -> 方法。



##### 针对角色控制守卫

注意这里只能装饰 `controller` 的方法，不能全局。

`SetMetadata` 装饰器:  第一个参数为`key`，第二个参数自定义我们的例子是数组存放的权限。

```tsx
import {
  Controller,
  Get,
  Post,
  Body,
  Patch,
  Param,
  Delete,
  SetMetadata,
} from '@nestjs/common';
import { AdminService } from './admin.service';
import { CreateAdminDto } from './dto/create-admin.dto';
import { UpdateAdminDto } from './dto/update-admin.dto';

@Controller('admin')
export class AdminController {
  constructor(private readonly adminService: AdminService) {}

  @Post()
  @SetMetadata('role', ['admin'])
  create(@Body() createAdminDto: CreateAdminDto) {
    return this.adminService.create(createAdminDto);
  }

  @Get()
  @SetMetadata('role', ['admin'])
  findAll() {
    return this.adminService.findAll();
  }

  @Get(':id')
  @SetMetadata('role', ['admin'])
  findOne(@Param('id') id: string) {
    return this.adminService.findOne(+id);
  }

  @Patch(':id')
  @SetMetadata('role', ['admin'])
  update(@Param('id') id: string, @Body() updateAdminDto: UpdateAdminDto) {
    return this.adminService.update(+id, updateAdminDto);
  }

  @Delete(':id')
  @SetMetadata('role', ['admin'])
  remove(@Param('id') id: string) {
    return this.adminService.remove(+id);
  }
}
```

守卫通行逻辑：

```tsx
import { CanActivate, ExecutionContext, Injectable } from '@nestjs/common';
import { Observable } from 'rxjs';
import { Reflector } from '@nestjs/core';
import type { Request } from 'express';

@Injectable()
export class AdminGuard implements CanActivate {
  // 1. 注入 Reflector 依赖
  constructor(private Reflector: Reflector) {}

  canActivate(
    context: ExecutionContext,
  ): boolean | Promise<boolean> | Observable<boolean> {
    // 2. 使用  Reflector 读取 setMetaData 的值
    // context.getHandler() 拿到控制器方法，例如 findAll
    const admin = this.Reflector.get<string[]>('role', context.getHandler());

    // 3. 获取到请求体
    const request = context.switchToHttp().getRequest<Request>();

    // 4. 这里通过判断 query 上面有没有 role 是 admin 的数据 ， 如果有就放行，没有就拒绝通行
    if (admin.includes(request.query.role as string)) {
      console.log('返回true');
      return true;
    } else {
      console.log('返回false');
      return false;
    }
  }
}

```



#### Pipe

`Pipe` 是管道的意思，管道有两个典型的用例：

- 转换：将输入数据转换为所需的形式（例如，从字符串转换为整数）
- 验证：评估输入数据，如果有效，只需原封不动地传递它;否则，引发异常

在这两种情况下，管道对控制器路由处理程序正在处理的参数进行操作。 `Nest` 在调用方法之前插入一个管道，该管道接收指定给该方法的参数并对它们进行操作。任何转换或验证操作都会在此时发生，之后使用任何（可能）转换的参数调用路由处理程序。



##### **9个开箱即用的管道：**

- `ValidationPipe` 一般用于全局的校验管道
- `ParseIntPipe` 转换为整数类型
- `ParseFloatPipe` 转为浮点数
- `ParseBoolPipe` 转布尔值
- `ParseArrayPipe` 转数组
- `ParseUUIDPipe` 转uuid
- `ParseEnumPipe` 转为枚举
- `DefaultValuePipe` 默认值，适用于一些参数可以不传，然后使用这个加入默认值
- `ParseFilePipe` 文件



##### 使用管道

```tsx
import { Controller, Get, Param, ParseIntPipe } from '@nestjs/common';
import { CatsService } from './cats.service';

@Controller('cats')
export class CatsController {
  constructor(private readonly catsService: CatsService) {}

  @Get(':id')
  findOne(@Param('id', ParseIntPipe) id: number) {
    return this.catsService.findOne(id);
  }
}
```

在上述代码中，`url params` 接收到的参数要求必须是数字类型, `ParseIntPipe`会判断是否是可转换的数字类型，如果是字符串会抛出错误，如果是字符串数字自动转换数字。



##### 自定义类型转换管道

```ts
import { ArgumentMetadata, Injectable, PipeTransform } from '@nestjs/common';

@Injectable()
export class PipePipe implements PipeTransform {
  transform(value: any, metadata: ArgumentMetadata) {
    return value;
  }
}
```

每个管道都必须实现该方法 `transform()` 以履行 `PipeTransform` 接口协定。此方法有两个参数：

- `value`: 当前处理的方法参数（在路由处理方法接收之前）
- `metadata`: 当前处理的方法参数的元数据。

`metadata`元数据对象具有以下属性：

```tsx
export interface ArgumentMetadata {
  type: 'body' | 'query' | 'param' | 'custom';
  metatype?: Type<unknown>;
  data?: string;
}
```

| 属性       | 说明                                                         |
| ---------- | ------------------------------------------------------------ |
| `type`     | 告诉我们参数是一个 body `@Body()`，query `@Query()`，param `@Param()` 还是自定义参数 |
| `metatype` | 参数的元类型，例如 `String`。 如果在函数签名中省略类型声明，或者使用原生 JavaScript，则为 `undefined`。 |
| `data`     | 传递给装饰器的字符串，例如 `@Body('string')`。如果您将括号留空，则为 `undefined`。 |



##### 默认参数

```tsx
import {
  Controller,
  Get,
  UseGuards,
  DefaultValuePipe,
  Query,
} from '@nestjs/common';
import { DogsService } from './dogs.service';
import { DogsGuard } from './dogs.guard';

@Controller('dogs')
// 使用路由守卫
@UseGuards(DogsGuard)
export class DogsController {
  constructor(private readonly dogsService: DogsService) {}

  @Get('search')
  findOne(
    @Query('keyword', new DefaultValuePipe('default key')) keyword: string,
  ) {
    console.log('🚀 ~  : DogsController -> findOne -> ', keyword);
    return 'Search keyword:' + keyword;
  }
}
```



##### 验证

管道的另一个作用就是验证，判断用户传过来的参数是否合法。



###### 基于 schema 的验证

安装`joi` 和 `@types/joi` :

```
npm install joi
npm install @types/joi -D
```

使用 `ES` 模块导入的方式导入 `joi` 时需要在 `tsconfig.json` 中启用 `esModuleInterop` 选项。接着使用 `Joi` 模块将三个属性均设置为必填项。

```ts
import Joi from 'joi';

export const createDogSchema = Joi.object({
  name: Joi.string().required(),
  age: Joi.number().required(),
  gender: Joi.bool().required(),
});

```

定义完 `schema` 后可以使用 `nest g pi joi-validation` 创建一个公共的管道，在 `transform` 函数中使用已经注入的`ObjectSchema` 对象提供的 `validate` 函数对请求参数 `value` 做验证，当验证不通过是抛出合理的异常，反之通过。

```tsx
import {
  ArgumentMetadata,
  BadRequestException,
  Injectable,
  PipeTransform,
} from '@nestjs/common';
import { ObjectSchema } from 'joi';

@Injectable()
export class JoiValidationPipe implements PipeTransform {
  constructor(private schema: ObjectSchema) {}

  transform(value: any, metadata: ArgumentMetadata) {
    const { error } = this.schema.validate(value);

    if (error) {
      throw new BadRequestException('Validation failed');
    }
    return value;
  }
}

```

管道定义好后，利用 `@UsePipes()` 装饰器使用到控制器中，并传入通过 `Joi` 定义的 `schema`：

```tsx
import { Controller, Post, Body, UseGuards, UsePipes } from '@nestjs/common';
import { DogsService } from './dogs.service';
import { CreateDogDto } from './dto/create-dog.dto';
import { DogsGuard } from './dogs.guard';
import { JoiValidationPipe } from 'src/joi-validation/joi-validation.pipe';
import { createDogSchema } from './schema/create-dog.schema';

@Controller('dogs')
// 使用路由守卫
@UseGuards(DogsGuard)
export class DogsController {
  constructor(private readonly dogsService: DogsService) {}

  @Post()
  // 使用自定义管道验证
  @UsePipes(new JoiValidationPipe(createDogSchema))
  create(@Body() createDogDto: CreateDogDto) {
    return this.dogsService.create(createDogDto);
  }
}


// 注意，别忘了还要维护 dto 中的 CreateDogDto 类型
// create-dog.dto.ts
export class CreateDogDto {
  name: string;
  age: number;
  gender: boolean;
}
```



###### 基于 dto 的验证

在基于 `schema` 的验证中不仅编写了通用的 `joi-validation` 管道，还用 `Joi` 库编写了一份和 `CreateDogDto` 几乎一样的 `schema` 文件，每当 `DTO` 文件有变更时就需要同步维护 `schema` 文件。

基于 `dto` 的验证就可以利用为已创建的 `CreateDogDto` 增加验证相关的装饰器并配合通过的管道即可完成，从而可以少维护一份文件，避免不一致造成的问题。

首先执行 `npm i --save class-validator class-transformer` 安装必要的模块，接着为 `CreateDogDto` 增加验证相关的装饰器。

```tsx
import { Transform } from 'class-transformer';
import { IsString, IsInt, IsBoolean, IsNotEmpty } from 'class-validator';

export class CreateDogDto {
  @IsNotEmpty({ message: 'name不能为空' })
  @IsString({ message: 'name必须是字符串' })
  name: string;

  // 考虑类型转换
  @Transform(({ value }) => {
    if (typeof value === 'string' && value.trim() !== '') {
      return Number(value);
    }
    return value;
  })
  @IsNotEmpty({ message: 'age不能为空' })
  @IsInt({ message: 'age必须是数字' })
  age: number;

  @IsBoolean()
  @IsNotEmpty()
  gender: boolean;
}
```

接着执行 `nest g pi dto-validation` 创建一个公共的管道，在这个管道中需要做这么几件事情：

1. 解构 `metadata` 参数，获取请求体参数的元类型。
2. 定义私有函数 `toValidation`，跳过非`dto`的类型（非`Javascript`原类型）。
3. 使用 `plainToInstance` 将元类型和请求体参数转为可验证的类型对象。
4. 通过 `validate` 函数执行校验，校验未通过则抛出合理的异常信息。

```tsx
import {
  ArgumentMetadata,
  BadRequestException,
  Injectable,
  PipeTransform,
  Type,
} from '@nestjs/common';
import { plainToInstance } from 'class-transformer';
import { validate } from 'class-validator';

@Injectable()
export class DtoValidationPipe implements PipeTransform {
  async transform(value: any, metadata: ArgumentMetadata) {
    // 解构 metadata 参数，获取请求体参数的元类型
    const { metatype } = metadata;

    // 跳过非DTO的类型（非Javascript原类型）。
    if (!metatype || !this.toValidation(metatype)) {
      return value;
    }

    // 使用 plainToInstance 将元类型和请求体参数转为可验证的类型对象。
    const object = plainToInstance(metatype, value);

    // 通过 validate 函数执行校验，校验未通过则抛出合理的异常信息。
    const errors = await validate(object);

    if (errors.length > 0) {
      throw new BadRequestException('Validation failed');
    }
    return value;
  }

  /**
   * 当 metatype 所指的参数的元类型仅为Javascript原生类型的话则跳过校验，这里只关注了对定义的DTO的校验
   */
  private toValidation(metatype: Type<any>): boolean {
    const types: any[] = [String, Boolean, Number, Array, Object];
    return !types.includes(metatype);
  }
}
```

使用这个管道：

```tsx
import { Controller, Post, Body, UseGuards } from '@nestjs/common';
import { DogsService } from './dogs.service';
import { CreateDogDto } from './dto/create-dog.dto';
import { DogsGuard } from './dogs.guard';
import { DtoValidationPipe } from 'src/dto-validation/dto-validation.pipe';

@Controller('dogs')
// 使用路由守卫
@UseGuards(DogsGuard)
export class DogsController {
  constructor(private readonly dogsService: DogsService) {}

  @Post('dto')
  createDto(@Body(new DtoValidationPipe()) createDogDto: CreateDogDto) {
    return this.dogsService.create(createDogDto);
  }
}
```



⚠️ ：`nest` 内置提供的 `ValidationPipe` 管道完全可以支持上述两种验证方式，我们不必为自定义验证管道花费时间。



##### 全局管道注册

1. 如果要注册全局管道，只需要在 `main.ts` 中注册即可:

```tsx
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';
import { ValidationPipe } from '@nestjs/common';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  // 全局管道注册
  app.useGlobalPipes(new ValidationPipe());

  await app.listen(3000);
}

bootstrap();

```

2. 再任意一个模块中注册管道，全部控制器（全局）都会经过这个管道。

```tsx
import { MiddlewareConsumer, Module, NestModule } from '@nestjs/common';
import { DogsService } from './dogs.service';
import { DogsController } from './dogs.controller';
import { ValidatePipe } from 'src/pipe/validate.pipe';
import { APP_PIPE } from '@nestjs/core';

@Module({
  controllers: [DogsController],
  providers: [
    DogsService,
    {
      provide: APP_PIPE,
      useClass: ValidatePipe,
    },
  ],
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



##### 自定义错误`message`

官方的错误消息可能并不是我们想要的，我们可能需要它在返回的错误信息里告知一下对应的字段名是什么,于是我们可以自定义个校验管道：

```tsx
import { ValidationError, ValidationPipe } from '@nestjs/common';

// 1. 继承 ValidationPipe 类
export class ValidatePipe extends ValidationPipe {
  // 2. 自定义 mapChildrenToValidationErrors 方法
  protected mapChildrenToValidationErrors(
    error: ValidationError,
    parentPath?: string,
  ): ValidationError[] {
    /**
     * error 在传入之前的结构是这样的：
     *  {
     *    target: CreateDogDto { name: 'ads', age: NaN, gender: true },
     *    value: NaN, // age 接受的是number 类型，并且支持类型转换，由于传入的是字符串，导致类型转换成NAN了
     *    property: 'age',
     *    children: [],
     *    constraints: { isInt: 'age必须是数字', isNotEmpty: 'age不能为空' }
     *  }
     */
    // 3. 保证功能不变，super 关键字调用父级的 mapChildrenToValidationErrors 方法拿到结果
    const errors = super.mapChildrenToValidationErrors(error, parentPath);
    /**
     * error 经过 mapChildrenToValidationErrors 方法处理后变成了数组
     * [
     *  ValidationError {
     *    target: CreateDogDto { name: 'ads', age: NaN, gender: true },
     *    value: NaN,
     *    property: 'age',
     *    children: [],
     *    constraints: { isInt: 'age必须是数字!!!' }
     *  }
     * ]
     */

    // 4. 拿到结果后自定义结构体
    errors.forEach((item) => {
      for (const key in item.constraints) {
        item.constraints[key] = `${item.property}-${item.constraints[key]}`;
      }
    });

    /**
     * 我们最终调整为如下结构：
     *  [
     *    ValidationError {
     *      target: CreateDogDto { name: 'ads', age: NaN, gender: true },
     *      value: NaN,
     *      property: 'age',
     *      children: [],
     *      constraints: { isInt: 'age-age必须是数字!!!' }
     *    }
     *  ]
     */

    return errors;
  }
}

```

我们继承`ValidationPipe`管道类，然后自定义它的`mapChildrenToValidationErrors`方法，由于需要保证原来的功能不变，我们使用`super`关键词调用父类的方法，得到结果后再自定义。

由于需要保证`ValidationError`的结构，我们只能调整`constraints`中的错误文本，拼接一个字段`key`，然后再通过后续的自定义过滤器，将`string`转换为对象。



#### Inteceptor

`Interceptor` 是拦截器的意思，拦截器具有一组有用的功能，这些功能的灵感来自面向方面的编程（AOP）技术。它们使以下方面成为可能：

- 在方法执行之前/之后绑定额外的逻辑
- 转换从函数返回的结果
- 转换从函数引发的异常
- 扩展基本函数行为
- 根据特定条件完全覆盖函数（例如，出于缓存目的）



##### 拦截器接口

每个拦截器都需要实现 `NestInterceptor` 接口的`intercept()`方法，该方法接收两个参数。方法原型如下：

```ts
function intercept(context: ExecutionContext, next: CallHandler): Observable<any>
```

- `ExecutionContext` 是执行上下文。
- `CallHandler` 是路由处理函数, 接口定义如下: 

```ts
export interface CallHandler<T = any> {
    handle(): Observable<T>;
}
```

`handle()`函数的返回值也就是对应路由函数的返回值, 以获取用户列表为例：

```ts
@Controller('user')
export class UserController {
  @Get()
  list() {
    return [];
  }
}
```

当访问 `/user/list` 路由时，路由处理函数返回数据`[]`，如果在应用拦截器的场景下，调用`CallHandler`接口的`handle()`方法得到的也是`Observable<[]>`(RxJs包装对象)。

所以，如果在拦截器中调用了`next.handle()`方法就会执行对应的路由处理函数，如果不调用的话就不会执行。



##### 一个请求链路的日志记录拦截器

```tsx
import {
  CallHandler,
  ExecutionContext,
  Injectable,
  Logger,
  NestInterceptor,
} from '@nestjs/common';
import { Observable } from 'rxjs';
import { tap } from 'rxjs/operators';
import { Request } from 'express';
import { format } from 'util';

@Injectable()
// 1. 每个拦截器都需要实现 NestInterceptor 接口的 intercept 方法
export class AppInterceptor implements NestInterceptor {
  // 2. 实例化日志记录器
  private readonly logger = new Logger();

  // 3. 实现 intercept 方法
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    // 4. 记录请求开始时间
    const start = Date.now();

    // 5. 调用完 handle() 后得到 RxJs 响应对象，使用 tap 可以得到路由函数的返回值
    return next.handle().pipe(
      tap((response) => {
        const host = context.switchToHttp();
        const request = host.getRequest<Request>();

        // 6. 打印请求方法，请求链接，处理时间和响应数据
        this.logger.log(
          format(
            '%s %s %dms %s',
            request.method,
            request.url,
            Date.now() - start,
            JSON.stringify(response),
          ),
        );
      }),
    );
  }
}
```

在控制器中使用这个拦截器：

```tsx
import { Controller, Get, UseInterceptors } from '@nestjs/common';
import { DogsService } from './dogs.service';
import { AppInterceptor } from 'src/common/interceptor';

@Controller('dogs')
export class DogsController {
  constructor(private readonly dogsService: DogsService) {}

  @Get()
  // 使用拦截器
  @UseInterceptors(AppInterceptor)
  findAll() {
    return this.dogsService.findAll();
  }
}
```



##### 全局拦截器

1. 在main.ts中使用以下代码即可：

```tsx
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';
import { AppInterceptor } from './common/interceptor';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  // 注册全局响应拦截
  app.useGlobalInterceptors(new AppInterceptor());

  await app.listen(3000);
}

bootstrap();

```

2. 再任意一个模块中注册拦截器，全部控制器（全局）都会经过这个拦截器。

```tsx
import { MiddlewareConsumer, Module, NestModule } from '@nestjs/common';
import { DogsService } from './dogs.service';
import { DogsController } from './dogs.controller';
import { APP_INTERCEPTOR } from '@nestjs/core';
import { AppInterceptor } from 'src/common/interceptor';

@Module({
  controllers: [DogsController],
  providers: [
    DogsService,
    {
      provide: APP_INTERCEPTOR,
      useClass: AppInterceptor,
    },
  ],
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



##### 响应体规范化

```tsx
import {
  Injectable,
  NestInterceptor,
  CallHandler,
  ExecutionContext,
} from '@nestjs/common';
import { map } from 'rxjs/operators';
import { Observable } from 'rxjs';

// 数据
interface data<T> {
  data: T;
}

@Injectable()
// 1. 继承nest的拦截器 NestInterceptor
export class Response<T = any> implements NestInterceptor {
  // 2. 重写 intercept 方法
  intercept(context: ExecutionContext, next: CallHandler): Observable<data<T>> {
    // 3. 规范响应体格式
    return next.handle().pipe(
      map((data) => {
        return {
          data,
          status: 0,
          success: true,
          message: 'success',
        };
      }),
    );
  }
}

```



##### 异常映射

>  后面我们用 `ExceptionFilter` 实现异常响应体。



```tsx
import {
  Injectable,
  NestInterceptor,
  ExecutionContext,
  BadGatewayException,
  CallHandler,
} from '@nestjs/common';
import { Observable, throwError } from 'rxjs';
import { catchError } from 'rxjs/operators';

@Injectable()
export class ErrorsInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    return next.handle().pipe(
      catchError(() => throwError(() => new BadGatewayException())), // catchError用来捕获异常
    );
  }
}
```



##### 重写路由函数逻辑

可以通过拦截器，将路由函数的逻辑改写，这里做一个缓存的案例：

1. 定义一个缓存服务

```tsx
import { Injectable } from '@nestjs/common';

@Injectable()
export class CacheService {
  private cacheData: Map<string, any> = new Map();

  get(key: string): any {
    return this.cacheData.get(key);
  }

  set(key: string, value: any): void {
    this.cacheData.set(key, value);
  }
}
```

2. `CacheInterceptor`的逻辑

```tsx
import {
  Injectable,
  NestInterceptor,
  ExecutionContext,
  CallHandler,
} from '@nestjs/common';
import { Observable, of } from 'rxjs';
import { tap } from 'rxjs/operators';
import { CacheService } from 'src/cache/cache.service';

@Injectable()
export class CacheInterceptor implements NestInterceptor {
  constructor(private cacheService: CacheService) {}

  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const request = context.switchToHttp().getRequest();
    const cacheKey = this.getCacheKey(request);

    // 1. 读取缓存
    const cachedData = this.cacheService.get(cacheKey);

    // 2. 有缓存读缓存数据
    if (cachedData) {
      return of(cachedData);
    }

    return next.handle().pipe(
      tap((response) => {
        this.cacheService.set(cacheKey, response);
      }),
    );
  }

  private getCacheKey(request: Request): string {
    // 根据请求参数、URL 或其他标识生成唯一的缓存键
    // 例如，可以使用请求的 URL 作为缓存键
    return request.url;
  }
}

```



#### ExceptionFilter 

`ExceptionFilter` 可以对抛出的异常做处理，`nest` 带有一个内置的异常层，负责处理整个应用程序中的所有未经处理的异常。当应用程序代码未处理异常时，该层会捕获该异常，然后自动发送适当的用户友好响应。

开箱即用，此操作由内置的全局异常过滤器执行，该过滤器处理 `HttpException` 类型（及其子类）的异常。当异常无法识别时（既不是 `HttpException` 也不是继承自 `HttpException` 的类），内置异常过滤器会生成以下默认 `JSON` 响应：

```tsx
{
  "statusCode": 500,
  "message": "Internal server error"
}
```

> 全局异常过滤器部分支持 `http-errors` 库。基本上，任何包含 `statusCode` 和 `message` 属性的抛出异常都将被正确填充并作为响应发送回来（而不是针对无法识别的异常的默认 `InternalServerErrorException` ）。



##### 抛出标准异常

`nest` 提供了一个内置的 `HttpException` 类，从 `@nestjs/common` 包中公开。对于典型的基于 HTTP REST/GraphQL API 的应用程序，最佳实践是在发生某些错误情况时发送标准 HTTP 响应对象。

例如：

```tsx
import { Controller, Get, HttpException, HttpStatus } from '@nestjs/common';
import { FilterService } from './filter.service';

@Controller('filter')
export class FilterController {
  constructor(private readonly filterService: FilterService) {}

  @Get()
  findAll() {
    // 抛出 nest 内置的标准错误 { "statusCode": 403, "message": "Forbidden" }
    throw new HttpException('Forbidden', HttpStatus.FORBIDDEN);
  }
}
```

`HttpException` 构造函数采用两个必需参数来确定响应：

- `response`: 定义 `JSON` 响应正文。它可以是 `string` 或 `object`。
- `status`： 定义 `HTTP` 状态代码。

所以你还可以这样写，达到自定义异常消息的目的：

```tsx
import { Controller, HttpException, HttpStatus, Post } from '@nestjs/common';
import { FilterService } from './filter.service';

@Controller('filter')
export class FilterController {
  constructor(private readonly filterService: FilterService) {}

  @Post()
  async create() {
    throw new HttpException(
      {
        status: HttpStatus.FORBIDDEN,
        error: 'This is a custom message',
      },
      403,
    );
  }
}
```



##### 内置 HTTP 异常 

`nest` 提供了一组继承自基类 `HttpException` 的标准异常。这些是从 `@nestjs/common` 包中公开的，代表许多最常见的 `HTTP` 异常：

| 内置异常类                    | 表示/含义                                                    |
| ----------------------------- | ------------------------------------------------------------ |
| BadRequestException           | 表示客户端发送了无效的请求，例如缺少必需的参数或格式不正确的参数。 |
| UnauthorizedException         | 表示客户端未经授权访问受保护的资源。                         |
| NotFoundException             | 表示请求的资源不存在。                                       |
| ForbiddenException            | 表示客户端没有访问请求资源的权限。                           |
| NotAcceptableException        | 表示服务器无法提供客户端请求的内容类型。                     |
| RequestTimeoutException       | 表示客户端请求超时。                                         |
| ConflictException             | 表示请求的操作与当前资源状态冲突。                           |
| GoneException                 | 表示请求的资源已经不存在。                                   |
| PayloadTooLargeException      | 表示请求的负载太大，服务器无法处理。                         |
| UnsupportedMediaTypeException | 表示请求的媒体类型不受支持。                                 |
| UnprocessableException        | 表示请求无法处理，因为它包含无效的数据。                     |
| InternalServerErrorException  | 表示服务器内部错误。                                         |
| NotImplementedException       | 表示请求的操作尚未实现。                                     |
| BadGatewayException           | 表示网关或代理服务器从上游服务器接收到无效的响应。           |
| ServiceUnavailableException   | 表示服务当前不可用。                                         |
| GatewayTimeoutException       | 表示网关或代理服务器在等待上游服务器响应时超时。             |

所有内置异常还可以使用 `options` 参数提供错误 `cause` 和错误描述：

```tsx
throw new BadRequestException('Something bad happened', { cause: new Error(), description: 'Some error description' })
```



##### 自定义异常类

创建一个异常过滤器, 它负责捕获作为`HttpException` 类实例的异常,并为它们设置自定义响应逻辑,为此，我们需要访问底层平台 `Request` 和 `Response`,我们将访问`Request`对象，以便提取原始 `url` 并将其包含在日志信息中, 我们将使用 `Response.json()` 方法，使用 `Response` 对象直接控制发送的响应。

```tsx
import {
  ArgumentsHost,
  Catch,
  ExceptionFilter,
  HttpException,
  HttpStatus,
  Logger,
} from '@nestjs/common';

@Catch(HttpException)
export class HttpFilter implements ExceptionFilter {
  // // 如果有日志服务，可以在constructor,中挂载logger处理函数
  // constructor(private readonly logger: Logger) {}

  // 实例化日志记录器
  private readonly logger = new Logger();

  catch(exception: HttpException, host: ArgumentsHost) {
    const ctx = host.switchToHttp(); // 获取请求上下文
    const request = ctx.getRequest(); // 获取请求上下文中的request对象
    const response = ctx.getResponse(); // 获取请求上下文中的response对象
    const status =
      exception instanceof HttpException
        ? exception.getStatus()
        : HttpStatus.INTERNAL_SERVER_ERROR; // 获取异常状态码

    // 设置错误信息
    const message = exception.message
      ? exception.message
      : `${
          status >= 500
            ? '服务器错误（Service Error）'
            : '客户端错误（Client Error）'
        }`;

    const nowTime = new Date().getTime();

    const errorResponse = {
      data: {},
      message,
      status: -1,
      date: nowTime,
      path: request.url,
      success: false,
    };

    // 将异常记录到logger中
    this.logger.error(
      `【${nowTime}】${request.method} ${request.url} query:${JSON.stringify(
        request.query,
      )} params:${JSON.stringify(request.params)} body:${JSON.stringify(
        request.body,
      )}`,
      JSON.stringify(errorResponse),
      'HttpExceptionFilter',
    );

    // 设置返回的状态码， 请求头，发送错误信息
    response.status(status);
    response.header('Content-Type', 'application/json; charset=utf-8');
    response.send(errorResponse);
  }
}

```

在控制器中注册：

```tsx
import {
  Controller,
  HttpException,
  HttpStatus,
  Post,
  UseFilters,
} from '@nestjs/common';
import { FilterService } from './filter.service';
import { HttpFilter } from 'src/common/filter';

@Controller('filter')
export class FilterController {
  constructor(private readonly filterService: FilterService) {}

  @Post()
  @UseFilters(HttpFilter)
  async create() {
    throw new HttpException(
      {
        status: HttpStatus.FORBIDDEN,
        error: 'This is a custom message',
      },
      403,
    );
  }
}

```



##### 全局过滤器注册

1. 如果要注册全局过滤器，只需要在 `main.ts` 中注册即可:

```tsx
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';
import { HttpFilter } from './common/filter';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  // 注册全局异常过滤器
  app.useGlobalFilters(new HttpFilter());

  await app.listen(3000);
}

bootstrap();

```

2. 再任意一个模块中注册过滤器，全部控制器（全局）都会经过这个过滤器。

```tsx
import { Module } from '@nestjs/common';
import { FilterService } from './filter.service';
import { FilterController } from './filter.controller';
import { APP_FILTER } from '@nestjs/core';
import { HttpFilter } from 'src/common/filter';

@Module({
  controllers: [FilterController],
  providers: [
    FilterService,
    {
      provide: APP_FILTER,
      useClass: HttpFilter,
    },
  ],
})
export class FilterModule {}

```



### 几种 AOP 机制的顺序

`Middleware`、`Guard`、`Pipe`、`Interceptor`、`ExceptionFilter` 都可以透明的添加某种处理逻辑到某个路由或者全部路由，这就是 AOP 的好处。

它们之间的执行顺序：

`Middleware` 是 `Express` 的概念，在最外层，到了某个路由之后，会先调用 `Guard`，`Guard` 用于判断路由有没有权限访问，然后会调用 `Interceptor`，对 `Contoller` 前后扩展一些逻辑，在到达目标 `Controller` 之前，还会调用 `Pipe` 来对参数做验证和转换。所有的 `HttpException` 的异常都会被 `ExceptionFilter` 处理，返回不同的响应。








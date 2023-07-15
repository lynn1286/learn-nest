使用`nest-cli`创建一个模板：

```bash
npx nest new nest-learn
```



### useClass

模板是使用了简写模式的注入方式：

```ts
import { Module } from '@nestjs/common';
import { AppController } from './app.controller';
import { AppService } from './app.service';

@Module({
  imports: [],
  controllers: [AppController],
  providers: [
    AppService // 简写模式
  ],
})
export class AppModule {}

```

实际上完整的写法应该是：

```ts
import { Module } from '@nestjs/common';
import { AppController } from './app.controller';
import { AppService } from './app.service';

@Module({
  imports: [],
  controllers: [AppController],
  providers: [
    // 完整的写法
    {
      provide: 'app_service',
      useClass: AppService,
    },
  ],
})
export class AppModule {}

```



### useValue

```ts
import { Module } from '@nestjs/common';
import { AppController } from './app.controller';
import { AppService } from './app.service';

@Module({
  imports: [],
  controllers: [AppController],
  providers: [
   	// useValue 注入一个对象
    {
      provide: 'person',
      useValue: {
        name: 'lynn',
        age: 18,
      },
    },
  ],
})
export class AppModule {}

```



### useFactory

```ts
import { Module } from '@nestjs/common';
import { AppController } from './app.controller';
import { AppService } from './app.service';

@Module({
  imports: [],
  controllers: [AppController],
  providers: [
    // 完整的写法
    {
      provide: 'app_service',
      useClass: AppService,
    },

    // useValue 注入一个对象
    {
      provide: 'person',
      useValue: {
        name: 'lynn',
        age: 18,
      },
    },
    
    // 动态注入一个对象
   	{
      provide: 'dynamic_person',
      useFactory: () => {
        return {
          name: 'dynamic_person -> lynn',
          age: 18,
        };
      },
    },
    
    // 动态注入支持参数传递
    {
      provide: 'dynamic_person_params',
      useFactory: (
        person: { name: string; age: number },
        appService: AppService,
      ) => {
        return {
          name: person.name,
          age: person.age,
          desc: appService.getHello(),
        };
      },
      // 参数传递，传递的是上面定义的 provide
      inject: ['person', 'app_service'],
    },
      
      
    // 动态注入支持异步
    {
      provide: 'dynamic_person_async',
      async useFactory() {
        await new Promise((resolve) => {
          setTimeout(resolve, 5000);
        });

        return {
          name: 'lynn',
          age: 18,
        };
      },
    },
  ],
})
export class AppModule {}

```



### useExisting

```ts
import { Module } from '@nestjs/common';
import { AppController } from './app.controller';
import { AppService } from './app.service';

@Module({
  imports: [],
  controllers: [AppController],
  providers: [
   	// useValue 注入一个对象
    {
      provide: 'person',
      useValue: {
        name: 'lynn',
        age: 18,
      },
    },
    
    // 给其他的token 起别名
    {
      provide: 'person_alias',
      useExisting: 'person', // person 和 person_alias 是同一个 provide
    },
  ],
})
export class AppModule {}

```



使用的方式根据 `provide` 的注入方式有所改变：

```ts
import { Controller, Get, Inject } from '@nestjs/common';
import { AppService } from './app.service';

@Controller()
export class AppController {
  // 构造器注入
  // constructor(private readonly appService: AppService) {}
  
  // 如果是完整的写法，需要指定 token
  @Inject('app_service')
  private readonly appService: AppService;
  
  // 如果是简写，Inject 装饰器不需要传递token
  // @Inject()
  // private readonly appService: AppService;


  @Get()
  getHello(): string {
    return this.appService.getHello();
  }
}

```

可以通过构造器注入的方式，也可以使用属性注入的方式，两者是等价的。












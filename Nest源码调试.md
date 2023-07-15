## 实现源码调试：

1. 拉取 `nest` 仓库到本地安装依赖后，修改 `packages/tsconfig.build.json` 的配置：

```js
{
  "compilerOptions": {
    // ...
	  "sourceMap": true, // 修改为 true , 生成相应的 '.map' 文件
    "inlineSources": true, // 修改为 true , 将代码与 sourcemaps 生成到一个文件中，要求同时设置了 --inlineSourceMap 或 --sourceMap 属性
    // ...
  }
}
```

2. 修改 `gulp` 配置文件,目的是为了将生成的 `.map`输出到 `node_modules`:

```js
// /tools/gulp/tasks/move
const distFiles = src([
  'packages/**/*',
  '!packages/**/*.ts',
  'packages/**/*.d.ts',
  'packages/**/*.map', // 新增
]);
```

3. 根目录执行 `npm run build` 构建产物后检查 `node_modules` 下的 `@nestjs`是否存在`.map`文件
4. 接着配置 `vscode` 的`launch.json`文件 ， 参数解析请看[文章](https://www.jianshu.com/p/d3c6e25ae815)

```json
{
  // 使用 IntelliSense 了解相关属性。
  // 悬停以查看现有属性的描述。
  // 欲了解更多信息，请访问: https://go.microsoft.com/fwlink/?linkid=830387
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Launch via NPM",
      "request": "launch", 
      "runtimeArgs": ["run-script", "start"],
      "runtimeExecutable": "npm",
      "console": "integratedTerminal", 
      "cwd": "${workspaceFolder}/sample/01-cats-app/", 
      "skipFiles": ["<node_internals>/**"],
      "resolveSourceMapLocations": ["${workspaceFolder}/**"],
      "type": "node"
    }
  ]
}

```

5. 最后一步啦，进入`01-cats-app` 项目先执行 `npm install` 安装依赖，接着再次回到根目录执行`npm run move:samples` ，这个命令是通过程序将根目录下的`node_modules/@nestjs`移动到`01-cats-app/node_modules`下，所以这就是为什么我们要先执行`npm install` ，否则会被覆盖哦，除了执行脚本移动你也可以手动复制粘贴的哈。


# webpack插件介绍及自定义插件

### plugin的核心概念

在 Webpack 构建流程中的特定时机会广播出对应的事件，插件可以监听这些事件的发生，在特定时机做对应的事情

### webpack的主要的三个阶段及事件

#### 初始化阶段

| 事件名            | 解释                                                         |
| ----------------- | ------------------------------------------------------------ |
| 初始化参数        | 从配置文件和 Shell 语句中读取与合并参数，得出最终的参数。 这个过程中还会执行配置文件中的插件实例化语句 `new Plugin()`。 |
| 实例化 `Compiler` | 用上一步得到的参数初始化 `Compiler` 实例，`Compiler` 负责文件监听和启动编译。`Compiler` 实例中包含了完整的 `Webpack` 配置，全局只有一个 `Compiler` 实例。 |
| 加载插件          | 依次调用插件的 `apply` 方法，让插件可以监听后续的所有事件节点。同时给插件传入 `compiler` 实例的引用，以方便插件通过 `compiler` 调用 Webpack 提供的 API。 |
| `environment`     | 开始应用 Node.js 风格的文件系统到 compiler 对象，以方便后续的文件寻找和读取。 |
| `entry-option`    | 读取配置的 `Entrys`，为每个 `Entry` 实例化一个对应的 `EntryPlugin`，为后面该 `Entry` 的递归解析工作做准备。 |
| `after-plugins`   | 调用完所有内置的和配置的插件的 `apply` 方法。                |
| `after-resolvers` | 根据配置初始化完 `resolver`，`resolver` 负责在文件系统中寻找指定路径的文件。 |

#### 编译阶段

| 事件名          | 解释                                                         |
| --------------- | ------------------------------------------------------------ |
| `run`           | 启动一次新的编译。                                           |
| `watch-run`     | 和 `run` 类似，区别在于它是在监听模式下启动的编译，在这个事件中可以获取到是哪些文件发生了变化导致重新启动一次新的编译。 |
| `compile`       | 该事件是为了告诉插件一次新的编译将要启动，同时会给插件带上 `compiler` 对象。 |
| `compilation`   | 当 `Webpack` 以开发模式运行时，每当检测到文件变化，一次新的 `Compilation` 将被创建。一个 `Compilation` 对象包含了当前的模块资源、编译生成资源、变化的文件等。`Compilation` 对象也提供了很多事件回调供插件做扩展。 |
| `make`          | 一个新的 `Compilation` 创建完毕，即将从 `Entry` 开始读取文件，根据文件类型和配置的 `Loader` 对文件进行编译，编译完后再找出该文件依赖的文件，递归的编译和解析。 |
| `after-compile` | 一次 `Compilation` 执行完成。                                |
| `invalid`       | 当遇到文件不存在、文件编译错误等异常时会触发该事件，该事件不会导致 Webpack 退出。 |

**在编译阶段中，最重要的要数 `compilation` 事件了，因为在 `compilation` 阶段调用了 Loader 完成了每个模块的转换操作，在 `compilation` 阶段又包括很多小的事件，它们分别是：**

| 事件名                 | 解释                                                         |
| ---------------------- | ------------------------------------------------------------ |
| `build-module`         | 使用对应的 Loader 去转换一个模块。                           |
| `normal-module-loader` | 在用 Loader 对一个模块转换完后，使用 `acorn` 解析转换后的内容，输出对应的抽象语法树（`AST`），以方便 Webpack 后面对代码的分析。 |
| `program`              | 从配置的入口模块开始，分析其 AST，当遇到 require 等导入其它模块语句时，便将其加入到依赖的模块列表，同时对新找出的依赖模块递归分析，最终搞清所有模块的依赖关系。 |
| `seal`                 | 所有模块及其依赖的模块都通过 Loader 转换完成后，根据依赖关系开始生成 Chunk。 |

#### 输出阶段

| 事件名        | 解释                                                         |
| ------------- | ------------------------------------------------------------ |
| `should-emit` | 所有需要输出的文件已经生成好，询问插件哪些文件需要输出，哪些不需要。 |
| `emit`        | 确定好要输出哪些文件后，执行文件输出，可以在这里获取和修改输出内容。 |
| `after-emit`  | 文件输出完毕。                                               |
| `done`        | 成功完成一次完成的编译和输出流程。                           |
| `failed`      | 如果在编译和输出流程中遇到异常导致 Webpack 退出时，就会直接跳转到本步骤，插件可以在本事件中获取到具体的错误原因。 |

**在输出阶段已经得到了各个模块经过转换后的结果和其依赖关系，并且把相关模块组合在一起形成一个个 Chunk。 在输出阶段会根据 Chunk 的类型，使用对应的模版生成最终要要输出的文件内容。**

### 怎么编写webpack插件

`webpack` 插件由以下组成：

- 一个 JavaScript 命名函数。
- 在插件函数的 prototype 上定义一个 `apply` 方法。
- 指定一个绑定到 webpack 自身的事件钩子。
- 处理 webpack 内部实例的特定数据(`Compiler` 或 `Compilation`)
- 功能完成后调用 webpack 提供的回调。

这个 `apply` 方法在安装插件时，会被 webpack compiler 调用一次。`apply` 方法可以接收一个 webpack的 compiler 对象的引用，从而可以在回调函数中访问到 compiler 对象。

简单示例 : 

##### 异步插件

```javascript
class MyPlugin {
    constructor() {
    }

    apply(compiler) {
        compiler.plugin('emit', function (compilcation, callback) {
            // emit是异步事件  , 需要调用callback , 否则程序将会卡这不会继续执行
            callback()
        })
    };
}

module.exports = MyPlugin;
```



##### 同步插件

```javascript
class MyPlugin {
    constructor() {
    }
    apply(compiler) {
        compiler.plugin('entry-option', function (c) {
            // entry-option是同步事件 ,无需callback()
        })
    };
}

module.exports = MyPlugin;
```

复杂点的示例:

```javascript
class MyPlugin {
    constructor() {
    }
    apply(compiler) {
        console.log('开始执行插件')
        compiler.plugin('done', function () {
            console.log('打包完成......')
        })
        compiler.plugin('emit', function (c, cb) {
            console.log('webpack 输出完毕.-----')
            cb()
        })
        compiler.plugin('compile', function () {
            console.log('webpack 编译器开始编译...-----')
        })

        compiler.plugin('compilation', function (compilation) {
            console.log('编译器开始一个新的编译任务...-----')
            compilation.plugin('optimize', function () {
                console.log('编译器开始优化文件...')
            })
        })
        /*
        开始执行插件
            webpack 编译器开始编译...-----
            编译器开始一个新的编译任务...-----
            编译器开始一个新的编译任务...-----
            编译器开始优化文件...
            编译器开始优化文件...
            webpack 输出完毕.-----
            打包完成......
         */
 		// 事件将按照特定的事件顺序执行
    };
}

module.exports = MyPlugin;
```



### Compiler 和 Compilation

在开发 Plugin 时最常用的两个对象就是 Compiler 和 Compilation，它们是 Plugin 和 Webpack 之间的桥梁。 Compiler 和 Compilation 的含义如下：

- Compiler 对象包含了 Webpack 环境所有的的配置信息，包含 `options`，`loaders`，`plugins` 这些信息，这个对象在 Webpack 启动时候被实例化，它是全局唯一的，可以简单地把它理解为 Webpack 实例；
- Compilation 对象包含了当前的模块资源、编译生成资源、变化的文件等。当 Webpack 以开发模式运行时，每当检测到一个文件变化，一次新的 Compilation 将被创建。Compilation 对象也提供了很多事件回调供插件做扩展。通过 Compilation 也能读取到 Compiler 对象。



### emit事件

有些场景下插件需要修改、增加、删除输出的资源，要做到这点需要监听 `emit` 事件，因为发生 `emit` 事件时所有模块的转换和代码块对应的文件已经生成好， 需要输出的资源即将输出，因此 `emit` 事件是修改 Webpack 输出资源的最后时机。

所有需要输出的资源会存放在 `compilation.assets` 中，`compilation.assets` 是一个键值对，键为需要输出的文件名称，值为文件对应的内容

##### 获取资源信息

```javascript
compiler.plugin('emit', (compilation, callback) => {
  // 读取名称为 fileName 的输出资源
  const asset = compilation.assets[fileName];
  // 获取输出资源的内容
  asset.source();
  // 获取输出资源的文件大小
  asset.size();
  callback();
});
```



##### 比如修改文件内容

```javascript
class MyPlugin {
    constructor() {
    }

    apply(compiler) {
        compiler.plugin('emit', function (compilcation, callback) {
            compilcation.assets['main.js'] = {
                // 返回文件内容
                source: () => {
                    return '修改文件内容---';
                },
                // 返回文件大小
                size: () => {
                    return Buffer.byteLength('修改文件内容---', 'utf8');
                }
            };
            callback()
        })
    };
}

module.exports = MyPlugin;
```

![1582373282477](C:\Users\VIVO__~1\AppData\Local\Temp\1582373282477.png)



### after-emit事件中可以做上传资源到服务器. . . 
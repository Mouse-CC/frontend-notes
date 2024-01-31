
脚手架：

框架内，模块 => 实例 => 单页应用中的页面
每个页面都有属于自己的参数，逻辑

跨实例处理数据：`vuex` ，全局数据的存储。（多实例之间的数据流和状态管理）=> 状态机

实例与实例之间的切换：`vue-router` ，

`vue-cli` 生成以上全部

![[Drawing 2023-12-29 17.51.18.excalidraw]]

## 三步 ：
### 1 . 基础 - 用于生成项目文件体系 + 开发者的习惯 / 规范性操作
### 2 . Metadata 元数据配置 [`bin/vue-init`](#^metadata)
### 3 . Parser & render 渲染和解析

## vue-cli 2.x

`package.json`
```json
{
 "name": "vue-cli",
  "version": "2.9.6",
  "description": "A simple CLI for scaffolding Vue.js projects.",
  "preferGlobal": true,
  "bin": {
    "vue": "bin/vue",
    "vue-init": "bin/vue-init",
    "vue-list": "bin/vue-list"
  },
  ...
}
```

入口：`bin` 
指令型执行

### 主入口

`bin/vue`
```js
#!/usr/bin/env node

// 手册性指引 - 描述 cli 有哪些功能
const program = require("commander");

// 提供了哪些 API
program
  .version(require("../package").version) // 版本号
  .usage("<command> [options]") // 能使用哪些命令：command
  .command("init", "generate a new project from a template")
  .command("list", "list available official templates")
  .command("build", "prototype a new project")
  .command("create", "(for v3 warning only)");

program.parse(process.argv);
```

从上可知，主命令为：`vue init` 和 `vue list`

#### `bin/vue-init` : 数据配置的主要内容 ^metadata
```js
#!/usr/bin/env node

// 工具箱 - 第三方工具：

// 下载远程仓库：通过 git-repo 去解析当前赋值的地址对应的仓库或是嵌套的仓库，将需要的文件下载到本地 - 在本地 node 环境读取文件实现某些公功能
const download = require("download-git-repo");
// 命令行处理工具
const program = require("commander");
// node 下文件操作系统：fs，existsSync 用来检测路径是否存在
const exists = require("fs").existsSync;
// node 自带模块，拼接路径
const path = require("path");
// 命令行加载效果：动态加载，进度条...
const ora = require("ora");
// 快速定位用户根目录
const home = require("user-home");
// 绝对路径替换成波浪号的展示
const tildify = require("tildify");
// 高亮打印
const chalk = require("chalk");
// (重点) 用户与脚本命令行的交互工具
const inquirer = require("inquirer");
// rm -rf | js 版本，用于删除文件
const rm = require("rimraf").sync;

// 内部工具

// 全局统一的打印文件，根据颜色区分当前执行状态 ...
const logger = require("../lib/logger");
// 根据模板构建项目
const generate = require("../lib/generate");
// 当前 cli 版本检查，本地 node 版本检查
const checkVersion = require("../lib/check-version");
// 兼容类的警告
const warnings = require("../lib/warnings");
// 路径处理，路径本地化
const localPath = require("../lib/local-path");

const isLocalPath = localPath.isLocalPath;
// 判断是否是绝对路径，若是相对路径，将其做拼接
const getTemplatePath = localPath.getTemplatePath;

/**
 * Usage.
 */

program
  // 模板类型，项目名称
  .usage("<template-name> [project-name]")
  // git clone 远端下载模板
  .option("-c, --clone", "use git clone")
  // 本地缓存的模板
  .option("--offline", "use cached template");

/**
 * Help.
 */

// on 起到监听作用，此处监听 '--help'，提供帮助手册
program.on("--help", () => {
  console.log("  Examples:");
  console.log();
  // 1. 使用官方的模板
  console.log(
    chalk.gray("    # create a new project with an official template")
  );
  console.log("    $ vue init webpack my-project");
  console.log();
  console.log(
    chalk.gray("    # create a new project straight from a github template")
  );
  // 2. 使用自己配置的模板，放在 github 上，创建时将会去 github 上下载
  console.log("    $ vue init username/repo my-project");
  console.log();
});

// 总结，使用下载好的模板或使用自己预设的模板：
// --offline || username/repo

/**
 * Help.
 */

function help() {
  program.parse(process.argv);
  if (program.args.length < 1) return program.help();
}
help();

/**
 * Settings.
 */

// 主要设置模块
// 获取相关信息，组织初始化参数，执行设置
// 如何获取到 => program.args

// 模板名称
let template = program.args[0];
// 模板名是否包含 '/' => 模板是否包含路径层级 => 区分是官方还是自己的
const hasSlash = template.indexOf("/") > -1;
// 项目名称
const rawName = program.args[1];
// 项目名称是否有 => 是否在当前文件夹下创建，以当前文件夹为根目录路径
const inPlace = !rawName || rawName === ".";
// 调整目录，有 inPlace 表示以当前文件夹为根目录路径，向上返回一层
const name = inPlace ? path.relative("../", process.cwd()) : rawName;
// 对当前路径做拼接操作
const to = path.resolve(rawName || ".");
const clone = program.clone || false;

// 本地模板目录文件名拼接，拼接的是此电脑的根目录下（mac 是当前用户，以用户名命名的文件夹）拼接 ".vue-templates/" + 模板名称 "template"
//
// zhouyinan/.vue-templates/webpack-demo
//
const tmp = path.join(home, ".vue-templates", template.replace(/[\/:]/g, "-"));
// 用户是否使用命令 '--offline' 模板从本地取
if (program.offline) {
  console.log(`> Use cached template at ${chalk.yellow(tildify(tmp))}`);
  template = tmp;
}

/**
 * Padding.
 */

// 隔离

console.log();
process.on("exit", () => {
  console.log();
});

// 以当前路径为根目录，且存在此目录：二次确认，是否以当前目录下构建（可能存在风险）=> 覆盖
if (inPlace || exists(to)) {
  inquirer.prompt
    ([
      {
        type: "confirm", // "yes" or "no"
        message: inPlace
          ? "Generate project in current directory?"
          : "Target directory exists. Continue?",
        name: "ok", // 输入 y，按回车，等于 ok
      },
    ])
    .then((answers) => {
      if (answers.ok) {
        run();
      }
    })
    .catch(logger.fatal);
} else {
  run();
}

/**
 * Check, download and generate the project.
 */

// 以上是关于参数的配置，下面是模板配置

function run() {
  // check if template is local
  // 检查本地路径是否存在
  if (isLocalPath(template)) {
    // 获取模板路径本身
    const templatePath = getTemplatePath(template);
    // 判断模板路径有效性 - 本地模板存在
    if (exists(templatePath)) {
      // 模板名称，模板路径，项目路径，根据这些生成项目文件
      generate(name, templatePath, to, (err) => {
        if (err) logger.fatal(err);
        console.log();
        logger.success('Generated "%s".', name);
      });
    } else {
      logger.fatal('Local template "%s" not found.', template);
    }
  } else {
    // 远端模板
    // checkVersion 版本检测
    checkVersion(() => {
      if (!hasSlash) {
        // use official templates - 使用官方模板
        //
        // 统一官方与本地模板的名称格式
        // 路径 + 模板名
        const officialTemplate = "vuejs-templates/" + template;
        // 版本性兼容，初代带 '-2.0'
        //
        // 包含 '#'
        if (template.indexOf("#") !== -1) {
          // 下载并生成官方模板
          downloadAndGenerate(officialTemplate);
        } else {
          // 不包含 '#'，判断是不是 '2.0' 版本
          if (template.indexOf("-2.0") !== -1) {
            warnings.v2SuffixTemplatesDeprecated(template, inPlace ? "" : name);
            return;
          }

          // warnings.v2BranchIsNowDefault(template, inPlace ? '' : name)
          downloadAndGenerate(officialTemplate);
        }
      } else {
        downloadAndGenerate(template);
      }
    });
  }
}

/**
 * Download a generate from a template repo.
 *
 * @param {String} template
 */

function downloadAndGenerate(template) {
  const spinner = ora("downloading template");
  spinner.start();
  // Remove if local template exists
  if (exists(tmp)) rm(tmp);
  // 从远端下载并生成
  download(template, tmp, { clone }, (err) => {
    spinner.stop();
    if (err)
      logger.fatal(
        "Failed to download repo " + template + ": " + err.message.trim()
      );
    generate(name, tmp, to, (err) => {
      if (err) logger.fatal(err);
      console.log();
      logger.success('Generated "%s".', name);
    });
  });
}
```

`inquirer` 做提问和问题梳理，整理用户的配置需求

#### lib 库

`/lib`
##### `.../logger`
```js
// 封装了一个带有提示色彩的命令行打印工具

const chalk = require("chalk");
const format = require("util").format;

/**
 * Prefix.
 */

const prefix = "   vue-cli";
const sep = chalk.gray("·");

/**
 * Log a `message` to the console.
 *
 * @param {String} message
 */

exports.log = function (...args) {
  const msg = format.apply(format, args);
  console.log(chalk.white(prefix), sep, msg);
};

/**
 * Log an error `message` to the console and exit.
 *
 * @param {String} message
 */

exports.fatal = function (...args) {
  if (args[0] instanceof Error) args[0] = args[0].message.trim();
  const msg = format.apply(format, args);
  console.error(chalk.red(prefix), sep, msg);
  process.exit(1);
};

/**
 * Log a success `message` to the console and exit.
 *
 * @param {String} message
 */

exports.success = function (...args) {
  const msg = format.apply(format, args);
  console.log(chalk.white(prefix), sep, msg);
};

```

==疑问，为什么要单独封装一个打印？==

方便全局，后续所有打印相关只要调用封装的这个就可以了，统一，方便使用者查看。

##### `.../check-version`
```js
const request = require("request");
const semver = require("semver");
const chalk = require("chalk");
const packageConfig = require("../package.json");

module.exports = (done) => {
  // Ensure minimum supported node version is used
  // 通过 satisfies 函数，获取到使当前可执行的最低版本 node 是，起到限制最低门槛的作用
  if (!semver.satisfies(process.version, packageConfig.engines.node)) {
    // 显示出来 node 版本需要 大于等于 xx 版本
    // 先至最低版本
    return console.log(
      chalk.red(
        "  You must upgrade node to >=" +
          packageConfig.engines.node +
          ".x to use vue-cli"
      )
    );
  }

  // 服务，判断当前最近一次版本号 与 本地版本 对比
  request(
    {
      url: "https://registry.npmjs.org/vue-cli",
      timeout: 1000,
    },
    (err, res, body) => {
      if (!err && res.statusCode === 200) {
        // 最近一次 版本号
        const latestVersion = JSON.parse(body)["dist-tags"].latest;
        const localVersion = packageConfig.version;
        if (semver.lt(localVersion, latestVersion)) {
          // lt 小于，当本地版本落后与线上版本，黄色文本，提示有新版本可同用 ...
          console.log(
            chalk.yellow("  A newer version of vue-cli is available.")
          );
          console.log();
          console.log("  latest:    " + chalk.green(latestVersion));
          console.log("  installed: " + chalk.red(localVersion));
          console.log();
        }
      }
      done();
    }
  );
};

```

一眼看出是一个请求，`vue` 在 `github` 上提供了 `registry` 服务，通过这个服务，可以去判断当前工具的版本号：通过 `const latestVersion = JSON.parse(body)["dist-tags"].latest` 拿到最近一次的版本号和本地的 `package.json` 的版本号做对比，在 node 下 `lt` 表示小于，当前本地版本落后于线上最近一次版本，将会通过粉笔提示。

本函数主要用于版本升级提示 ...

##### `.../generate`
```js
const chalk = require('chalk')
// 两大工具
// 静态网页文件组织生成器 - 用于生成静态网页文件组织
const Metalsmith = require('metalsmith')
// 模板引擎 - 关联文件并处理渲染
const Handlebars = require('handlebars')
// 拼接元数据，并将元数据传递给这两个工具，在不同的回调钩子里去完成解析和渲染的工作

const async = require('async')
const render = require('consolidate').handlebars.render
const path = require('path')
// 多条件匹配
const multimatch = require('multimatch')
const getOptions = require('./options')
const ask = require('./ask')
const filter = require('./filter')
const logger = require('./logger')

// register handlebars helper
Handlebars.registerHelper('if_eq', function (a, b, opts) {
  // 如果 a 和 b 相等，执行函数处理
  return a === b
    ? opts.fn(this)
    : opts.inverse(this)
})

Handlebars.registerHelper('unless_eq', function (a, b, opts) {
  return a === b
    ? opts.inverse(this)
    : opts.fn(this)
})

/**
 * Generate a template given a `src` and `dest`.
 *
 * @param {String} name
 * @param {String} src
 * @param {String} dest
 * @param {Function} done
 */

module.exports = function generate(name, src, dest, done) {
  // 获取配置，根据 src 去获 metadata，并在元数据的基础上新增一些默认信息
  // name，模板名称，传入模板名称和模板路径到 getOptions 返回所有配置项组成的对象
  const opts = getOptions(name, src)
  // src => templatePath，`静态网页文件组织生成器` 通过模板路径返回静态网页文件组织。
  const metalsmith = Metalsmith(path.join(src, 'template'))
  // 初始化模板元数据： 基于 metalsmith 的 metadata，再做一次加工 ...
  const data = Object.assign(metalsmith.metadata(), {
    destDirName: name,
    inPlace: dest === process.cwd(),
    noEscape: true
  })

  opts.helpers && Object.keys(opts.helpers).map(key => {
    // 通过模板引擎处理，将 helpers 中的每一项通过上述操作符做处理，[if_eq, unless_eq] ...
    // 注册配置对象 - 动态组件配置可以学习这种方式
    Handlebars.registerHelper(key, opts.helpers[key])
  })

  const helpers = { chalk, logger }

  // 注册配置对象
  //
  // before hook 中 做预处理
  if (opts.metalsmith && typeof opts.metalsmith.before === 'function') {
    // opts.metalsmith.before 存在，说明有预处理要求。那么传入内容做更新 ...
    opts.metalsmith.before(metalsmith, opts, helpers)
  }

  // before hook 之后，增加全局插件，质询问题
  //
  // prompts：模板中预置的和用户的交互，用 use 去调起这些交互
  //
  // filters：文件过滤
  metalsmith.use(askQuestions(opts.prompts))
    .use(filterFiles(opts.filters))
    .use(renderTemplateFiles(opts.skipInterpolation))

  if (typeof opts.metalsmith === 'function') {
    // 满足嵌套情况 metalsmith 内嵌 metalsmith
    opts.metalsmith(metalsmith, opts, helpers)
  } else if (opts.metalsmith && typeof opts.metalsmith.after === 'function') {
    opts.metalsmith.after(metalsmith, opts, helpers)
  }

  // 收尾，清空执行栈，防止不幂等
  metalsmith.clean(false)
    .source('.') // start from template root instead of `./src` which is Metalsmith's default for `source`
    .destination(dest)
    .build((err, files) => {
      done(err)
      if (typeof opts.complete === 'function') {
        const helpers = { chalk, logger, files }
        opts.complete(data, helpers)
      } else {
        logMessage(opts.completeMessage, data)
      }
    })

  return data
}

/**
 * Create a middleware for asking questions.
 *
 * @param {Object} prompts
 * @return {Function}
 */

function askQuestions(prompts) {
  return (files, metalsmith, done) => {
    ask(prompts, metalsmith.metadata(), done)
  }
}

/**
 * Create a middleware for filtering files.
 *
 * @param {Object} filters
 * @return {Function}
 */

function filterFiles(filters) {
  return (files, metalsmith, done) => {
    filter(files, filters, metalsmith.metadata(), done)
  }
}

/**
 * Template in place plugin.
 *
 * @param {Object} files
 * @param {Metalsmith} metalsmith
 * @param {Function} done
 */

function renderTemplateFiles(skipInterpolation) {
  skipInterpolation = typeof skipInterpolation === 'string'
    ? [skipInterpolation]
    : skipInterpolation
  return (files, metalsmith, done) => {
    // 由模板文件对象返回 key 组成的数组
    const keys = Object.keys(files)
    const metalsmithMetadata = metalsmith.metadata()
    async.each(keys, (file, next) => {
      // skipping files with skipInterpolation option
      if (skipInterpolation && multimatch([file], skipInterpolation, { dot: true }).length) {
        return next()
      }
      const str = files[file].contents.toString()
      // do not attempt to render files that do not have mustaches
      if (!/{{([^{}]+)}}/g.test(str)) {
        return next()
      }
      render(str, metalsmithMetadata, (err, res) => {
        if (err) {
          err.message = `[${file}] ${err.message}`
          return next(err)
        }
        files[file].contents = new Buffer(res)
        next()
      })
    }, done)
  }
}

/**
 * Display template complete message.
 *
 * @param {String} message
 * @param {Object} data
 */

function logMessage(message, data) {
  if (!message) return
  render(message, data, (err, res) => {
    if (err) {
      console.error('\n   Error when rendering template complete message: ' + err.message.trim())
    } else {
      console.log('\n' + res.split(/\r?\n/g).map(line => '   ' + line).join('\n'))
    }
  })
}
```

### 总结 

`init` 阶段保证了数据的组装是完整的，它严格的把所有模板路径查找到，确保在当前阶段能拿到所有的模板，和用户输入的参数。将这些组合成元数据，并以同样的模式层层传递（模板有嵌套）。在开始之处将传递的数据和 `metalsmith` 默认的数据做整合，最终生成 `opts` ，有了 `opts` ，就根据其去渲染。

## vue-cli dev (最新)

基本架构：monorepo

### 主要

#### create
`vue-cli/packages/@vue/cli/lib/create.js`
```js
const fs = require('fs-extra')
const path = require('path')
const inquirer = require('inquirer')
const Creator = require('./Creator')
const { clearConsole } = require('./util/clearConsole')
const { getPromptModules } = require('./util/createTools')
const { chalk, error, stopSpinner, exit } = require('@vue/cli-shared-utils')
const validateProjectName = require('validate-npm-package-name')

async function create(projectName, options) {
  // options 用户所有的输入
  if (options.proxy) {
    process.env.HTTP_PROXY = options.proxy
  }

  // 对标 2.x 版本的路径判断
  const cwd = options.cwd || process.cwd()
  // 当前文件名存在
  const inCurrent = projectName === '.'
  const name = inCurrent ? path.relative('../', cwd) : projectName
  const targetDir = path.resolve(cwd, projectName || '.')

  const result = validateProjectName(name)
  if (!result.validForNewPackages) {
    console.error(chalk.red(`Invalid project name: "${name}"`))
    result.errors && result.errors.forEach(err => {
      console.error(chalk.red.dim('Error: ' + err))
    })
    result.warnings && result.warnings.forEach(warn => {
      console.error(chalk.red.dim('Warning: ' + warn))
    })
    exit(1)
  }

  if (fs.existsSync(targetDir) && !options.merge) {
    if (options.force) {
      await fs.remove(targetDir)
    } else {
      await clearConsole()
      // 二次确认，在当前目录操作
      if (inCurrent) {
        const { ok } = await inquirer.prompt([
          {
            name: 'ok',
            type: 'confirm',
            message: `Generate project in current directory?`
          }
        ])
        if (!ok) {
          return
        }
      } else {
        // 存在相同目录
        const { action } = await inquirer.prompt([
          {
            name: 'action',
            type: 'list',
            message: `Target directory ${chalk.cyan(targetDir)} already exists. Pick an action:`,
            choices: [
              { name: 'Overwrite', value: 'overwrite' }, // 覆盖
              { name: 'Merge', value: 'merge' }, // 合并
              { name: 'Cancel', value: false } // 取消
            ]
          }
        ])
        if (!action) {
          return
        } else if (action === 'overwrite') {
          console.log(`\nRemoving ${chalk.cyan(targetDir)}...`)
          await fs.remove(targetDir)
        }
      }
    }
  }

  // 以创建器的形式进行封装，通过创建者类，创建实例，通过这样的方式做封装。好处是每次创建都不会相互影响，伴随着实例的销毁，实例下的所有配置都会自动回收
  const creator = new Creator(name, targetDir, getPromptModules())
  await creator.create(options)
}

module.exports = (...args) => {
  return create(...args).catch(err => {
    stopSpinner(false) // do not persist
    error(err)
    if (!process.env.VUE_CLI_TEST) {
      process.exit(1)
    }
  })
}
```

#### creator

`vue-cli/packages/@vue/cli/lib/Creator.js`
#### generator

`vue-cli/packages/@vue/cli/lib/Generator.js`
#### generatorAPI

`vue-cli/packages/@vue/cli/lib/GeneratorAPI.js`

### 相对于 cli 2.x 提升了什么？

1. 创建流程细化
2. 文件结构上以模块、类、加上实例创建的方式 => 暴露实例 / 单例的方式更好的做组装
3. 办事不求人，所有模板平摊，向外界提供自身能力，让外界书写插件。调用时初始化插件，并将插件交给外部应用。

>[!tip] 提示：
>我们在实现一个大型工具的时候，是否提供了 `common` 的能力（基础能力），假设已提供，在 `common` 之上，可以让用户不断的拓展需要的能力。能力以函数插件 `plugin` 的形式装入。底座 `common` 不关心插件功能是如何实现的，其只负责暴露自身的 `API` 给插件 `plugin`。辅助插件某些功能的完成。这样的架构叫 => ==微内核==

常见的微内核架构工具：`webpack`、`babel`

面试重点：

三步 => 每一步做了什么
1. 获取数据
2. 元数据配置
3. 渲染和生成

不同版本的 cli 工具实现过程中的区别
1. 新版架构的提升
2. 创建流程更多细化

自建脚手架
深入脚手架的概念 => 
通过 `node.js` 打通 `js` 和 `cmdline`

## Q & A

### 洋葱模型

设计模式：职责链和迭代器

![[vue cli 2024-01-12 14.41.22.excalidraw]]

上图有三个元，彼此之间互相独立，每个元是处理某一事务的工具或函数。通过一定的配置去配置元与元之间的顺序，直到最后产出。

正向：config -> output => 配置通过事物转为输出
反向：output -> config => 解析输出反转结果转回配置

洋葱模型，形象的展示了职责相互依存的关系

迭代器：前后陆续执行，生成一条迭代器。
职责链：上面的每一环互相负责不同的职责，以链条的形式做职责传递。

### async 和 defer 区别

脚本的加载方式：什么都不写、async、defer
![[screenshot-20240116-105404.png]]
若多个 script 加载 async 是加载完就执行，看 script 加载速度，速度快得先执行，因此 async 是无序的。而 defer 由于执行在主流程之后，那么多个执行将嵌套，以下用执行 1，2 来做对比：
`... 下载 1 ... 下载 2 ... 主流程执行完毕 ... 执行 2 ... 执行 1` 可以看出 defer 是有序的，如果两个 script 有依赖，比如 1 依赖 2 并不会产生问题

### src 和 href 区别

src: `script` 中表示文件下载，src 能加载任意文家
href: 页面跳转 => 加载带渲染
## npm

CLI文档：[https://docs.npmjs.com/cli/v8/commands/npm](https://docs.npmjs.com/cli/v8/commands/npm)

`npm`是nodejs内置的包管理器，类似于Python的pip，文档所使用的版本为`8.15.1`

<br />

### npm init - 项目初始化

文档：[https://docs.npmjs.com/cli/v8/commands/npm-init](https://docs.npmjs.com/cli/v8/commands/npm-init)

别名：`create`, `innit`

::: warning

npm init用来项目初始化 ，这会生成一个`package.json`文件。但是通常我们不会使用这种方式初始化项目，

而是使用像`Vue CLI`、`Vite`等封装得更高级的脚手架工具来初始化

:::

```bash
C:\Users\Administrator\Desktop\demo>npm init -y
npm WARN config global `--global`, `--local` are deprecated. Use `--location=global` instead.
Wrote to C:\Users\Administrator\Desktop\demo\package.json:

{
  "name": "demo",
  "version": "1.0.0",
  "description": "",
  "main": "index.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "keywords": [],
  "author": "",
  "license": "ISC"
}
```

<br />

### npm config - 管理npm配置文件

文档：[https://docs.npmjs.com/cli/v8/commands/npm-config](https://docs.npmjs.com/cli/v8/commands/npm-config)

别名：`c`

子命令：

```bash
npm config set <key>=<value> [<key>=<value> ...]
npm config get [<key> [<key> ...]]
npm config delete <key> [<key> ...]
npm config list [--json]
npm config edit
```

查看默认的NPM源

```bash
C:\Users\Administrator>npm config get registry
npm WARN config global `--global`, `--local` are deprecated. Use `--location=global` instead.
https://registry.npmjs.org/
```

设置为淘宝NPM源

```bash
C:\Users\Administrator>npm config set registry https://registry.npmmirror.com
npm WARN config global `--global`, `--local` are deprecated. Use `--location=global` instead.

C:\Users\Administrator>npm config get registry
npm WARN config global `--global`, `--local` are deprecated. Use `--location=global` instead.
https://registry.npmmirror.com/
```

<br />

### npm install - 安装包/升级NPM

文档：[https://docs.npmjs.com/cli/v8/commands/npm-install](https://docs.npmjs.com/cli/v8/commands/npm-install)

别名：`add`, `i`, `in`, `ins`, `inst`, `insta`, `instal`, `isnt`, `isnta`, `isntal`, `isntall` （PS：为了手残党真是下足了功夫）

常用参数：

| 参数                                   | 说明                                                       |
| -------------------------------------- | ------------------------------------------------------------ |
| `--global`  or `-g` | 安装到全局（默认会安装到`当前目录/node_modules`） |
| `--save-prod` or `-P` or `--save` or `-S` | 作为项目依赖安装，写入包名到`package.json`中的`dependencies`区域（这是默认选项） |
| `--save-dev` or `-D` | 作为开发依赖安装，写入包名到`package.json`中的`devDependencies`区域中 |
| `--no-save`                 | 仅安装，不修改`package.json`                                 |

升级NPM：	

```bash
# 升级npm版本
npm install -g npm

# 安装指定npm版本
npm install -g npm@8.15.0
```

安装第三方包

```bash
npm install eslint --save-dev
npm install husky --save-dev
npm install echarts --save
```

<br />

### npm uninstall - 卸载包

文档：[https://docs.npmjs.com/cli/v8/commands/npm-uninstall](https://docs.npmjs.com/cli/v8/commands/npm-uninstall)

别名：`unlink`, `remove`, `rm`, `r`, `un`

<br />

### npm ls - 查看包

查看全局都安装了哪些包

文档：[https://docs.npmjs.com/cli/v8/commands/npm-ls](https://docs.npmjs.com/cli/v8/commands/npm-ls)

别名：`list`

```bash
C:\Users\Administrator>npm ls -g
npm WARN config global `--global`, `--local` are deprecated. Use `--location=global` instead.
C:\Users\Administrator\AppData\Roaming\npm
+-- npm-check-updates@12.5.11
+-- npm@8.15.1
+-- nrm@1.2.5
+-- pnpm@7.7.0
`-- yarn@1.22.17
```

<br />

## yarn

## pnpm（推荐）
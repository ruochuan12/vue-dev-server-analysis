# vue-dev-server-analysis

## 环境准备

```json
{
  "name": "@vue/dev-server",
  "version": "0.1.1",
  "description": "Instant dev server for Vue single file components",
  "main": "middleware.js",
  "bin": {
    "vue-dev-server": "./bin/vue-dev-server.js"
  },
  "scripts": {
    "test": "cd test && node ../bin/vue-dev-server.js"
  }
}
```

## 源码

### 整体结构

### vue-dev-server.js

```js
// vue-dev-server/bin/vue-dev-server.js

#!/usr/bin/env node

const express = require('express')
const { vueMiddleware } = require('../middleware')

const app = express()
const root = process.cwd();

app.use(vueMiddleware())

app.use(express.static(root))

app.listen(3000, () => {
  console.log('server running at http://localhost:3000')
})
```

### vueMiddleware

```js
// vue-dev-server/middleware.js

const vueMiddleware = (options = defaultOptions) => {
  // 省略
  return async (req, res, next) => {
    // 省略
    // 对 .vue 结尾的文件进行处理
    if (req.path.endsWith('.vue')) {
    // 对 .js 结尾的文件进行处理
    } else if (req.path.endsWith('.js')) {
    // 对 /__modules/ 开头的文件进行处理
    } else if (req.path.startsWith('/__modules/')) {
    } else {
      next()
    }
  }
}
exports.vueMiddleware = vueMiddleware
```

接着我们来看下具体是怎么处理的。

### 对 .vue 结尾的文件进行处理

```js
const key = parseUrl(req).pathname
let out = await tryCache(key)

if (!out) {
  // Bundle Single-File Component
  const result = await bundleSFC(req)
  out = result
  cacheData(key, out, result.updateTime)
}

send(res, out.code, 'application/javascript')
```

### parseUrl

### tryCache

### cacheData

### bundleSFC

### send

```js
async function bundleSFC (req) {
  const { filepath, source, updateTime } = await readSource(req)
  const descriptorResult = compiler.compileToDescriptor(filepath, source)
  const assembledResult = vueCompiler.assemble(compiler, filepath, {
    ...descriptorResult,
    script: injectSourceMapToScript(descriptorResult.script),
    styles: injectSourceMapsToStyles(descriptorResult.styles)
  })
  return { ...assembledResult, updateTime }
}
```

```js
const path = require('path')
const fs = require('fs')
const readFile = require('util').promisify(fs.readFile)
const stat = require('util').promisify(fs.stat)
const parseUrl = require('parseurl')
const root = process.cwd()

async function readSource(req) {
  const { pathname } = parseUrl(req)
  const filepath = path.resolve(root, pathname.replace(/^\//, ''))
  return {
    filepath,
    source: await readFile(filepath, 'utf-8'),
    updateTime: (await stat(filepath)).mtime.getTime()
  }
}

exports.readSource = readSource
```

### 对 .js 结尾的文件进行处理

```js
const key = parseUrl(req).pathname
let out = await tryCache(key)

if (!out) {
  // transform import statements
  const result = await readSource(req)
  out = transformModuleImports(result.source)
  cacheData(key, out, result.updateTime)
}

send(res, out, 'application/javascript')
```

### 对 /__modules/ 开头的文件进行处理

```js
const key = parseUrl(req).pathname
const pkg = req.path.replace(/^\/__modules\//, '')

let out = await tryCache(key, false) // Do not outdate modules
if (!out) {
  out = (await loadPkg(pkg)).toString()
  cacheData(key, out, false) // Do not outdate modules
}

send(res, out, 'application/javascript')
```

### loadPkg 加载包

```js
const fs = require('fs')
const path = require('path')
const readFile = require('util').promisify(fs.readFile)

async function loadPkg(pkg) {
  if (pkg === 'vue') {
    const dir = path.dirname(require.resolve('vue'))
    const filepath = path.join(dir, 'vue.esm.browser.js')
    return readFile(filepath)
  }
  else {
    // TODO
    // check if the package has a browser es module that can be used
    // otherwise bundle it with rollup on the fly?
    throw new Error('npm imports support are not ready yet.')
  }
}

exports.loadPkg = loadPkg
```

## 总结
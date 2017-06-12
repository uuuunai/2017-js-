# HyperApp
## 简介
>HyperApp 是一款基于 Docker 的自动化部署 Linux 应用的 iPhone 工具，适合敢于尝试的 Linux 零基础用户，无需敲入命令，即可安装那些原本需要命令行才能搞定的 Linux 程序。

## 背景介绍
>我们可以将 HyperApp 理解成可视化的 SSH 工具，它能够看到你的 VPS 信息，包括 CPU 使用率、内存和磁盘占用，以及运行时间，并且还能控制 VPS 重启、关机和查看历史记录。

>通过 Docker虚拟化技术，HyperApp可以很容易的部署一些常见应用。

## 使用流程

* Name- 为你的 VPS 取个名字，比如 My First VPS。
* Host- VPS 的 IP 地址
* Port- 端口号，一般为 22
* User- VPS 登录用户名，一般为 root
* Password- 密码

## 使用介绍
### Download
使用npm.

```
npm i hyperapp
```
### Usage
嵌入到文档中.
```
<script src="hyperapp.js"></script>
```
试试看
```
const { app, html } = hyperapp

app({
    model: "Hi.",
    view: model => html`<h1>${model}</h1>`
})
```
在ES6中
```
import { app, html } from "hyperapp"
```
CommonJS中.
```
const { app, html } = require("hyperapp")
```
Browserify
```
browserify index.js -t hyperxify -g uglifyify | uglifyjs > bundle.js
```
Webpack
```
browserify index.js -t hyperxify -g uglifyify | uglifyjs > bundle.js
```
### 示例
Hello world
```
app({
    model: "Hi.",
    view: model => html`<h1>${model}</h1>`
})
```
Counter
```
app({
    model: 0,
    update: {
        add: model => model + 1,
        sub: model => model - 1
    },
    view: (model, msg) => html`
        <div>
            <button onclick=${msg.add}>+</button>
            <h1>${model}</h1>
            <button onclick=${msg.sub} disabled=${model <= 0}>-</button>
        </div>`
})
```
Input
```
app({
    model: "",
    update: {
        text: (_, value) => value
    },
    view: (model, msg) => html`
        <div>
            <h1>Hi${model ? " " + model : ""}.</h1>
            <input oninput=${e => msg.text(e.target.value)} />
        </div>`
})
```
Drag & Drop
```
const model = {
    dragging: false,
    position: {
        x: 0, y: 0, offsetX: 0, offsetY: 0
    }
}

const view = (model, msg) => html`
    <div
        onmousedown=${e => msg.drag({
            position: {
                x: e.pageX, y: e.pageY, offsetX: e.offsetX, offsetY: e.offsetY
            }
        })}
        style=${{
            userSelect: "none",
            cursor: "move",
            position: "absolute",
            padding: "10px",
            left: `${model.position.x - model.position.offsetX}px`,
            top: `${model.position.y - model.position.offsetY}px`,
            backgroundColor: model.dragging ? "gold" : "deepskyblue"
        }}
    >Drag Me!
    </div>`

const update = {
    drop: model => ({ dragging: false }),
    drag: (model, { position }) => ({ dragging: true, position }),
    move: (model, { x, y }) => model.dragging
        ? ({ position: { ...model.position, x, y } })
        : model
}

const subs = [
    (_, msg) => addEventListener("mouseup", msg.drop),
    (_, msg) => addEventListener("mousemove", e =>
        msg.move({ x: e.pageX, y: e.pageY }))
]

app({ model, view, update, subs })
```

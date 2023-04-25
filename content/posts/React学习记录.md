---
title: "React学习记录"
date: 2019-09-02 00:00:00.0
draft: false
type: "post"
showTableOfContents: true
tags: ["React"]
---

---
layout:     post
title:      "React学习记录"
date:       2019-09-02
author:     "马赛琦"
tags:
    - React
---

> 不会前端的java工程师不是好java工程师

## 前言 

俗话说的好，`不会前端的java工程师不是好java工程师`。笔者在之前已经学习过Vue的相关技术栈，事实上笔者的毕业设计也是用Vue做的~除了Vue，React和Flutter如今也是如日中天般的火爆，今天这篇文章记录一下学React的过程。后续应该随着学习的深入还会不断更新，所以这里排个update log，方便记录~

- 2019-09-02：开篇，根据技术胖的React教学文章进行学习并记录，记录学习过程中遇到的一些困惑
- 2019-09-03：总计6800字了，目前初步完成了基于React16的简单学习记录，后续可能会根据官方文档进行扩展或补充相关的React技术栈

## React环境准备

1. 准备Node.js环境
2. 安装React的脚手架工具
````
npm install -g create-react-app
````
`create-react-app`是React官方出的脚手架工具，你可以选择其他第三方的脚手架工具
3. 创建一个React的demo项目
````
create-react-app demo
````

至此，我们已经初步搭建了一个React的demo工程，我们了解一下**目录结构**

- README.md :这个文件主要作用就是对项目的说明，已经默认写好了一些东西，你可以简单看看。如果是工作中，你可以把文件中的内容删除，自己来写这个文件，编写这个文件可以使用Markdown的语法来编写。

- package.json: 这个文件是webpack配置和项目包管理文件，项目中依赖的第三方包（包的版本）和一些常用命令配置都在这个里边进行配置，当然脚手架已经为我们配置了一些了，目前位置，我们不需要改动。如果你对webpack了解，对这个一定也很熟悉。

- package-lock.json：这个文件用一句话来解释，就是锁定安装时的版本号，并且需要上传到git，以保证其他人再npm install 时大家的依赖能保证一致。

- gitignore : 这个是git的选择性上传的配置文件，比如一会要介绍的node_modules文件夹，就需要配置不上传。

- node_modules :这个文件夹就是我们项目的依赖包，到目前位置，脚手架已经都给我们下载好了，你不需要单独安装什么。

- public ：公共文件，里边有公用模板和图标等一些东西。

- src ： 主要代码编写文件，这个文件夹里的文件对我们来说最重要，都需要我们掌握。

**public文件夹介绍**

这个文件都是一些项目使用的公共文件，也就是说都是共用的，我们就具体看一下有那些文件吧。

- favicon.ico : 这个是网站或者说项目的图标，一般在浏览器标签页的左上角显示。

- index.html : 首页的模板文件，我们可以试着改动一下，就能看到结果。

- mainifest.json：移动端配置文件。

**src文件夹介绍**

这个目录里边放的是我们开放的源代码，我们平时操作做最多的目录。

- index.js : 这个就是项目的入口文件，视频中我们会简单的看一下这个文件。 

- index.css ：这个是index.js里的CSS文件。

- app.js : 这个文件相当于一个方法模块，也是一个简单的模块化编程。

- serviceWorker.js: 这个是用于写移动端开发的，PWA必须用到这个文件，有了这个文件，就相当于有了离线浏览的功能。

了解完目录结构，接下来我们删除`src`下所有文件，开始我们的旅程。

## 定义组件

> 组件名称必须以大写字母开头。
>
> React 会将以小写字母开头的组件视为原生 DOM 标签。例如，`<div />` 代表 HTML 的 div 标签，而 `<Welcome />` 则代表一个组件，并且需在作用域内使用 `Welcome`。

1. 在`src`目录下创建一个文件`index.js`，写入以下代码 
````js
import React from 'react'
import ReactDOM from 'react-dom'
import ComponentDemo from './ComponentDemo'
ReactDOM.render(<ComponentDemo />, document.getElementById('root'))
````
前两个import用来导入React的必要文件，后面一个import用来导入我们的自定义组件`ComponentDemo`
这样我们就将一个名为`ComponentDemo`的组件挂载在了名为`root`的DOM节点上

2. 在`src`目录下创建一个文件`ComponentDemo.js`，写入以下代码：
````jsx
import React, { Component } from 'react';
class ComponentDemo extends Component {
    constructor(props) {
        super(props);
        this.state = {  };
    }
    render() {
        return (  
        	<div>Hello, World!</div>
        );
    }
}
export default ComponentDemo;
````
这里涉及到一个ES6的语法-`解构赋值`，第一行的写法等同于
````js
import React from 'react'
const Component = React.Component
````
这样我们就有了属于自己的组件，我们采用的是定义`class`去继承React.Component的方式实现我们的组件定义，但是ES6中的`class`本质上是一个语法糖，根本上还是定义了一个`function`，因此我们也可以通过函数的方式定义组件。  
通过函数的方式定义的组件称为`函数组件`，但是，class组件比`函数组件`拥有一些额外特性。(官网描述，个人感觉两者可以完全互换，`函数组件`可以通过`prototype`为添加方法和属性，只是class组件很多写法可能更加优雅)  
扩展：[理解 es6 class 中 constructor 方法 和 super 的作用](https://www.jianshu.com/p/fc79756b1dc0)

3. 打开终端，输入以下命令，查看效果
````
npm start
````
浏览器应该会自动打开相应页面，你也可以自己输入`127.0.0.1:3000`进行查看
![](https://img.masaiqi.com/2019-09-02-084706.png)

PS：`root`节点的位置在`./public/index.html`中，打开这个文件我们可以看到我们的`root`节点其实是一个id为`root`的DOM节点，如果需要额外使用Jquary等js框架也可以在这个页面上进行引入使用，但是不推荐，技术栈最好还是能统一一致

````html
<body> 
    <noscript>You need to enable JavaScript to run this app.</noscript>
    <div id="root"></div>
</body>
````

## JSX语法

> JSX就是Javascript和XML结合的一种格式。React发明了JSX，可以方便的利用HTML语法来创建虚拟DOM，当遇到`<`，JSX就当作HTML解析，遇到`{`就当JavaScript解析.

在之前的流程中，我们创建了属于我们自己的组件`ComponentDemo`，我们发现我们在我们的js代码中写HelloWorld时是这样写的

````jsx
render() {
  return (  
    <div>Hello, World!</div>
  );
}
````

这就是一个简单的JSX语法的运用，如果我们用传统的js写法上面的语句要换成这样

````jsx
render() {
  return return React.createElement('div', null, 'Hello, World!');
}
````

可以看出，借助JSX语法大大简化了我们的React开发，就好像在写简单html一样简单高效  

在JSX中，我们可以通过`{}`使用js语法，这点类似于Vue中的插值表达式，如下代码中使用了JSX的三元运算符

````jsx
render() {
  return (
    <div>{false ? '' : "Hello, World!"}</div>
  );
}
````

## 组件外层包裹原则
React的组件最外层必须进行包裹，如以下代码就会报错

```JSX
render() {
  return (
    <div>我是一号</div>
    <div>我是二号</div>
  );
}
```

正确的写法为

````jsx
render() {
  return (
    <div>
      <div>我是一号</div>
      <div>我是二号</div>
    </div>
  );
}
````

这种写法最终生成的html代码中是最外层包围有<div>的

![](https://img.masaiqi.com/2019-09-02-091724.png)

某些情况下，我们希望最外层不要包裹但是又要能通过React语法，我们可以使用`<Fragment>`标签，如下(记得引入React.Fragment组件)

````jsx
//记得引入Fragment组件
import React,{Component,Fragment } from 'react'
//......省略若干代码
render() {
  return (
    <Fragment>
      <div>我是一号</div>
      <div>我是二号</div>
    </Fragment>
  );
}
````

这样生成的html代码中就不会有外层的包裹

![](https://img.masaiqi.com/2019-09-02-092335.png)

## 数据绑定与事件处理

之前的流程中我们都是在render函数中硬编码我们的页面，这里我们尝试做一些简单的响应式的交互设计  

### 数据绑定

尝试将render中的一些数据剥离到state中
````jsx
class ComponentDemo extends Component {
    constructor(props) {
        super(props);
        this.state = {
            inputValue: '初始值', //input中的值
        };
    }
    render() {
        return (
            <Fragment>
            		<input value={this.state.inputValue}/>
                <div>{this.state.inputValue}</div>
            </Fragment>
        );
    }
}
````
这里我们将div的内容提取到了构造函数中的`state`中，这个属性是定义在父类Component中的readOnly属性，可以理解为这个组件的`状态`，我们通过以上代码完成了`state`的数据与标签的绑定。这小节的最后我会列出官网中写出的一些特性，我们暂时不要去关心它。  
通过上面的代码，打开浏览器，我们可以看到页面上已经正常读取并显示我们在state中设置的inputValue的值，在控制台空你应该看到有如下错误：

````
Warning: Failed prop type: You provided a `value` prop to a form field without an `onChange` handler. This will render a read-only field. If the field should be mutable use `defaultValue`. Otherwise, set either `onChange` or `readOnly`.
````
这个错误的意思是如果我们使用input标签的value值进行数据绑定的同时，必须同时写上onChange方法，否则这个value的值将变成一个readOnly的域，如果你希望继续保持可变性，你需要将数据绑定到defaultValue上  
**事实上，到此为止我们已经实现了React数据绑定**。

### 事件处理

我们的需求是实现简单交互，即用户输入什么，我们显示什么，因此我们下面要做的是绑定onChange事件，通过事件触发`state`的改变，实现响应式。  

我们参考数据绑定的操作，有如下代码

````jsx
import React, { Component, Fragment } from 'react';
class ComponentDemo extends Component {
    constructor(props) {
        super(props);
        this.state = {
            inputValue: '初始值', //input中的值
        };
    }
    render() {
        return (
            <Fragment>
                <input value={this.state.inputValue} onChange={this.handleChange} />
                <div>{this.state.inputValue}</div>
            </Fragment>
        );
    }
    handleChange(e) {
        this.state = {
            inputValue: e.target.value
        }
    }
}
````

打开浏览器，随便输入个内容，控制台报错

![](https://img.masaiqi.com/2019-09-02-101916.png)

这是因为在handleChange方法中的this没有定义，没能拿到当前组件的上下文信息。  

我们需要这么一个方法：**改变方法的this并保留或扩展方法的参数**  

这也就是我们的`bind`方法，有两种方式可以进行数据绑定

- 第一种 在使用方法的地方调用bind
````jsx
render() {
  return (
    <Fragment>
      <input value={this.state.inputValue} onChange={this.handleChange.bind(this)} />
      <div>{this.state.inputValue}</div>
    </Fragment>
  );
}
````

- 第二种 在构造函数中使用bind(推荐，原因后续深究，技术胖文章中说有效率方面的考虑)
````jsx
constructor(props) {
        super(props);
        this.state = {
            inputValue: '初始值', //input中的值
        };
        this.handleChange = this.handleChange.bind(this)
}
````
使用了bind方法后，再次尝试改变输入框值，发现还是没有任何变化，这是因为我们错误的使用了this.state=XXXX进行`state`的修改，对于`state`，我们只允许使用this.setState方法进行修改，这个方法是个`浅合并`方法，只会修改你传的内容。
最终我们有如下代码：
````jsx
class ComponentDemo extends Component {
    constructor(props) {
        super(props);
        this.state = {
            inputValue: '初始值', //input中的值
        };
        this.handleChange = this.handleChange.bind(this)
    }
    render() {
        return (
            <Fragment>
                <input value={this.state.inputValue} onChange={this.handleChange} />
                <div>{this.state.inputValue}</div>
            </Fragment>
        );
    }
    handleChange(e) {
        this.setState({
            inputValue: e.target.value
        })
    }
}
````
改变输入的值，页面同步改变，说明我们成功通过事件修改了`state`。  

**关于band方法**，MDN给出的定义是

> The bind() methods creates a new function that, when called, has its 
> `this` keyword set to the provided value, with a given sequence of 
> arguments preceding any provided when the new function is called.

简单翻译一下，就是说bind方法本质上是创建了一个新的函数，这个新函数的this被bind的第一个参数指定，其余参数将作为新函数的参数供调用时使用。  

**你也可以使用箭头函数（它会绑定当前scope的this引用）**

笔者查阅了相关资料，找到简书上有个作者给出了基于apply方法的bind方法的实现。
您可以参考以下文章：  
[bind方法的ES6+ES5实现](https://www.jianshu.com/p/75b26254dea3)  
[Js apply方法和call方法使用区别](https://www.cnblogs.com/chenhuichao/p/8493095.html)  
[三言两语之js事件、事件流以及target、currentTarget、this那些事](https://www.cnblogs.com/54td/p/5936816.html)  
[react-native 之"Cannot update during an exitsting state transition" 与 函数方法bind()/箭头函数有关](https://blog.csdn.net/junhuahouse/article/details/80623686)

### `state`总结(摘自官网)

[官网参考](https://react.docschina.org/docs/state-and-lifecycle.html)

- state 是私有的，并且完全受控于当前组件。除了拥有并设置了它的组件，其他组件都无法访问。
- 构造函数是唯一可以给 `this.state` 赋值的地方，修改`state`必须使用`this.setState`
- `state`的更新可能是异步的  
可以通过setState方法提供的回调函数(setState的第二个参数)保证执行完state更新后执行想要的代码
````jsx
this.setState({
       //state的内容
    },()=>{
        //异步回调
    })
````
- `state`的更新会被合并(浅合并)(比如`state`中有两个属性A、B，通过setState方法只写了A，那么只修改A，不会修改B，B任在`state`中)
- 数据是向下流动的。组件可以选择把它的`state`作为`props`向下传递到它的子组件中。不管是父组件或是子组件都无法知道某个组件是有状态的还是无状态的，并且它们也并不关心它是函数组件还是 class 组件。(这一点其实就论述了React单向数据流，后文有描述)

### 优化：使用ref引用组件实例

上文中我们获得input标签的值是使用了`e.target.value`这样的方式，以此为例，我们可以采用ref指向input实例进而让我们的代码拥有更好的语义化
````jsx
class ComponentDemo extends Component {
    constructor(props) {
        super(props);
        this.state = {
            inputValue: '初始值', //input中的值
        };
        this.handleChange = this.handleChange.bind(this)
    }
    render() {
        return (
            <Fragment>
                <input value={this.state.inputValue} onChange={this.handleChange} ref={(input) => { this.inputComponet = input }} />
                <div>{this.state.inputValue}</div>
            </Fragment>
        );
    }

    handleChange(e) {
        this.setState({
            //使用ref的语义化方式
            inputValue: this.inputComponet.value
        })
    }
}
````
在上述代码中，我们使用ref获得了input实例并赋值给this.inputComponet，属性名我们可以自行选择，语义化得以实现。

## 列表渲染

这次我们尝试通过React展现列表数据
````jsx
class ComponentDemo extends Component {
    constructor(props) {
        super(props);
        this.state = {
            inputValue: '初始值', //input中的值
            list: ['我是小一', '我是小二', '我是小三']
        };
        this.handleChange = this.handleChange.bind(this)
    }
    render() {
        return (
            <Fragment>
                <input value={this.state.inputValue} onChange={this.handleChange} ref={(input) => { this.inputComponet = input }} />
                <div>{this.state.inputValue}</div>
                <ul>
                    {
                        this.state.list.map(
                            (item, index) => <li>{item}</li>
                        )
                    }
                </ul>
            </Fragment>
        );
    }
    handleChange(e) {
        this.setState({
            inputValue: this.inputComponet.value
        })
    }
}
````
打开浏览器，已经可以正常显示列表数据，但是会报以下错误
````
Each child in a list should have a unique "key" prop.
````
这是因为没有对list设置key值，通过下面代码设置key值

````JSX
class ComponentDemo extends Component {
    constructor(props) {
        super(props);
        this.state = {
            inputValue: '初始值', //input中的值
            list: ['我是小一', '我是小二', '我是小三']
        };
        this.handleChange = this.handleChange.bind(this)
    }
    render() {
        return (
            <Fragment>
                <input value={this.state.inputValue} onChange={this.handleChange} ref={(input) => { this.inputComponet = input }} />
                <div>{this.state.inputValue}</div>
                <ul>
                    {
                        this.state.list.map(
                          	//设置key值 暂时设置为index 你也可以设置为index + item
                            (item, index) => <li key={index}>{item}</li>
                        )
                    }
                </ul>
            </Fragment>
        );
    }
    handleChange(e) {
        this.setState({
            inputValue: this.inputComponet.value
        })
    }
}
````

在之前写好的handleChange方法中增加对`state`中list数据的处理便可以实现动态添加列表数据。  

同时，我们也可以顺便完成点击删除的方法，最终代码如下：

````JSX
class ComponentDemo extends Component {
    constructor(props) {
        super(props);
        this.state = {
            inputValue: '初始值', //input中的值
            list: ['我是小一', '我是小二', '我是小三']
        };
        this.handleChange = this.handleChange.bind(this)
    }
    render() {
        return (
            <Fragment>
                <input value={this.state.inputValue} onChange={this.handleChange} ref={(input) => { this.inputComponet = input }} />
                <div>{this.state.inputValue}</div>
                <ul>
                    {
                        this.state.list.map(
                            //设置key值
                            (item, index) => (
                                <li
                                    key={index}
                                    onClick={this.deleteItem.bind(this, index)}
                                >
                                    {item}
                                </li>
                            )
                        )
                    }
                </ul>
            </Fragment>
        );
    }

    handleChange(e) {
        this.setState({
            //使用ref的语义化方式
            inputValue: this.inputComponet.value,
            //ES6 Spread syntax
            list: [...this.state.list, this.inputComponet.value]
        })
    }
    deleteItem(index) {
        //这里注意不要直接操作state
        let list = this.state.list
        list.splice(index, 1)
        this.setState({
            list: list
        })
    }
}
````

这里有一个稍微注意的地方就是

````
list: [...this.state.list, this.inputComponet.value]
````

这是ES6的**Spread syntax**(展开运算符)，[参考链接](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Operators/Spread_syntax)  

拓展：[React 构造函数绑定事件如何传参](https://segmentfault.com/q/1010000015186121)[React 构造函数绑定事件如何传参](https://segmentfault.com/q/1010000015186121)

## 条件渲染

在render方法中写js的逻辑语句就可以实现条件渲染

### if语句

````jsx
render() {
        //条件渲染
        let divContent;
        if(this.state.list.length % 2 === 0) {
            divContent = <div>list长度为偶数</div>
        }else {
            divContent = <div>list长度为奇数</div>
        }
        return (
            <Fragment>
                <input value={this.state.inputValue} onChange={this.handleChange} ref={(input) => { this.inputComponet = input }} />
                <div>{this.state.inputValue}</div>
                <ul>
                    {
                        this.state.list.map(
                            //设置key值
                            (item, index) => (
                                <li
                                    key={index}
                                    onClick={this.deleteItem.bind(this, index)}
                                >
                                    {item}
                                </li>
                            )
                        )
                    }
                </ul>
                {divContent}
            </Fragment>
        );
    }
````

### 与运算符&&

> 在 JavaScript 中，`true && expression` 总是会返回 `expression`, 而 `false && expression` 总是会返回 `false`。
>
> 因此，如果条件是 `true`，`&&` 右侧的元素就会被渲染，如果是 `false`，React 会忽略并跳过它。

这里笔者直接摘取React官网文档的例子

````jsx
function Mailbox(props) {
  const unreadMessages = props.unreadMessages;
  return (
    <div>
      <h1>Hello!</h1>
      {unreadMessages.length > 0 &&
        <h2>
          You have {unreadMessages.length} unread messages.
        </h2>
      }
    </div>
  );
}

const messages = ['React', 'Re: React', 'Re:Re: React'];
ReactDOM.render(
  <Mailbox unreadMessages={messages} />,
  document.getElementById('root')
);
````

## 组件提取

之前流程中，我们将很多功能混杂在了一个组件中，很明显这不适合`高内聚，低耦合`的开发理念，我们需要将组件`解耦`，争取开发出`高可用` `高内聚`的组件。换句话说，我们需要对组件进行合适合理的提取拆分。  

在React中，我们需要了解如何拆分使用父子组件的模式，如何进行父子之间的传值。

我们抽离ComponetDemo的代码到ChildComponetDemo中，有如下代码

````jsx
class ComponentDemo extends Component {
    constructor(props) {
        super(props);
        this.state = {
            inputValue: '初始值', //input中的值
            list: ['我是小一', '我是小二', '我是小三']
        };
        this.handleChange = this.handleChange.bind(this)
    }
    render() {
        return (
            <Fragment>
                <input value={this.state.inputValue} onChange={this.handleChange} ref={(input) => { this.inputComponet = input }} />
                <div>{this.state.inputValue}</div>
                <ul>
                    {/* 子组件写法 */}
                    {
                        this.state.list.map(
                            (item, index) => (
                                <ChildComponetDemo content={item}/>
                            )
                        )
                    }
                </ul>
                {divContent}
            </Fragment>
        );
    }

    handleChange(e) {
        this.setState({
            //使用.target方法
            //inputValue: e.target.value
            
            //使用ref的语义化方式
            inputValue: this.inputComponet.value,
            //ES6 Spread syntax
            list: [...this.state.list, this.inputComponet.value]
        })
    }
    deleteItem(index) {
        //这里注意不要直接操作state
        let list = this.state.list
        list.splice(index, 1)
        this.setState({
            list: list
        })
    }
}
````

````jsx
class ChildComponetDemo extends Component {
    constructor(props) {
        super(props);
    }
    render() {
        return (
            <li>{this.props.content}</li>
        );
    }
}
````

在子组件ChildComponetDemo中，我们使用`props`属性接受了父组件ComponentDemo的参数  

在父组件ComponentDemo中，我们将item传到了子组件ChildComponetDemo的content中(其实是props.content)

**这样，通过`props`属性，我们就完成了父子组件传值问题**

同样地，我们也可以通过`props`传递方法实现我们之前的点击删除

在父组件ComponentDemo中增加函数传递`deleteItem={this.deleteItem.bind(this)}`

````JSX
{
  this.state.list.map(
    (item, index) => (
      <ChildComponetDemo
        key = {index}
        deleteItem={this.deleteItem.bind(this)}
        content={item}
        index={index}
        />
    )
  )
}
````

子组件ChildComponetDemo这样写

````jsx
class ChildComponetDemo extends Component {
    render() {
        return (
            <li onClick={this.handleClcik.bind(this)}>{this.props.content}</li>
        );
    }
    handleClcik() {
        this.props.deleteItem(this.props.index)
    }
}
````

你可能会想，把handleClcik方法放到JSX中让代码更简洁，甚至只要一句话

````jsx
return (
  {/*注意一定要加箭头函数或bind。*/}
  <li onClick={() => {this.props.deleteItem(this.props.index)}}>{this.props.content}</li>
);
````

或者这样

````jsx
return (
  {/*注意一定要加箭头函数或bind。*/}
  <li onClick={this.props.deleteItem.bind(this.props.index)}>{this.props.content}</li>
);

````

**关于Props**

> 组件无论是使用[函数声明还是通过 class 声明](https://react.docschina.org/docs/components-and-props.html#function-and-class-components)，都决不能修改自身的 props。
>
> React 非常灵活，但它也有一个严格的规则：
>
> **所有 React 组件都必须像纯函数一样保护它们的 props 不被更改。**
>
> [“纯函数”](https://en.wikipedia.org/wiki/Pure_function)，该函数不会尝试更改入参，且多次调用下相同的入参始终返回相同的结果。
>
> 与`state`不同，`state`允许 React 组件随用户操作、网络响应或者其他变化而动态更改输出内容。

扩展：  

[react-native 之"Cannot update during an exitsting state transition" 与 函数方法bind()/箭头函数有关](https://blog.csdn.net/junhuahouse/article/details/80623686)  
[React官网对于Props的叙述](https://react.docschina.org/docs/components-and-props.html)

## React单向数据流

这个单独拉出来一个标题可能有点不充实，但是确是我认为很重要的对于React理解的一节

有过Vue开发的经验的程序员应该都知道，Vue是双向数据绑定的，我们只需要改变我们双向数据绑定的变量，相应的页面也会动态刷新，实现即时渲染DOM

那么，React的单向数据流是不是就不是双向数据绑定了，进而不是`MVVM`框架了？

回答这个问题前，我们先回顾一下React数据流走向

父组件中我们定义和维护`state`，同时通过`props`传递给子组件，数据到达子组件这里。子组件并不知道他的父组件有哪些`state`，只知道有哪些父组件向下传递给它的`props`；父组件并不知道子组件的`state`，只知道向子组件传递`props`。这也就是我们所说的，**React的数据流向是从上至下的单向数据流**。

但是React中子组件也可以通过父组件传递的`props`中的方法去改变父组件的`state`，这也就实现了“双向数据绑定”

扩展：  

[[前端MVVM模式及其在Vue和React中的体现](https://www.cnblogs.com/lalalagq/p/9900849.html)](https://www.cnblogs.com/lalalagq/p/9900849.html)

## 通过PropTypes校验参数值

这个比较简单，直接看代码应该就明白了，这里主要举例类型检验，必填校验，默认值，官网有更多校验项。

记得导入`prop-types`

````jsx
import React, { Component } from 'react';
import PropTypes from 'prop-types'

class ChildComponetDemo extends Component {
    render() {
        return (
            <li onClick={this.props.deleteItem.bind(this.props.index)}>{this.props.content}</li>
        );
    }
    handleClcik() {
        this.props.deleteItem(this.props.index)
    }
}

// 参数校验
ChildComponetDemo.propTypes = {
    // 必须是数字类型
    index: PropTypes.number,
    //必须是函数类型且必传(必填项)
    deleteItem: PropTypes.func.isRequired
}

//参数设置默认值
ChildComponetDemo.defaultProps = {
    index : -1
}
````

## 生命周期(部分已过时)

> 生命周期函数指在某一个时刻组件会自动调用执行的函数

先贴一张图(水印见图)

![](https://img.masaiqi.com/2019-09-03-090814.jpg)

`constructor`不算生命周期函数，而是叫构造函数，它是ES6的基本语法。虽然它和生命周期函数的性质一样，但不能认为是生命周期函数。

### Mounting阶段

Mounting阶段叫挂载阶段，伴随着整个虚拟DOM的生成，它里边有三个小的生命周期函数。

- `componentWillMount`：在组件即将被挂载到页面的时刻执行。
- 
- `render` : 页面state或props发生变化时执行。
- 
- `componentDidMount`：组件挂载完成时被执行。

ps：`componentWillMount`和`componentDidMount`这两个生命周期函数，只在页面刷新时执行一次，而render函数是只要有state和props变化就会执行。

### Updation阶段

组件发生改变的更新阶段，这是React生命周期中比较复杂的一部分，它有两个基本部分组成，一个是props属性改变，一个是state状态改变（这个在生命周期的图片中可以清楚的看到）。

- `shouldComponentUpdate`：在组件更新之前被执行。它要求返回一个布尔类型的结果，必须有返回值。返回true代表组件允许更新，false代表组织组件更新。

- `componentWillUpdate`：在组件更新前，在`shouldComponentUpdate`执行后执行，如果`shouldComponentUpdate`返回了false，则这个函数也不会被执行。

- `componentDidUpdate`：组件更新之后执行

- `componentWillReceiveProps`：子组件接收到父组件传递过来的参数或父组件render函数重新被执行条件下会执行

### Unmounting阶段

- `componentWillUnmount`：当组件从页面上被删除时执行

**以上有部分方法已经out of date，后续补充**

## 利用生命周期函数优化性能

在我们之前的父子组件代码中其实存在一个性能问题，我们用React提供的浏览器插件进行分析。   
打开React浏览器插件，切换到profiler标签，点击录制，同时在我们的输入框中输入任意一个字符后点击停止录制，结果如下：  
![](https://img.masaiqi.com/2019-09-03-093541.png)
从Ranked图中我们可以看出，尽管我们只输入了一个字符，增加了一个子组件显示在我们的页面上，但是实际上React处理时是直接重新渲染了整个子组件列表，用上图来说，多浪费了2ms的性能在无用处理上。  

我们可以采用`shouldComponentUpdate`生命周期函数进行控制，`shouldComponentUpdate`接收两个参数：  

- nextProps：变化后的属性

- nextState：变化后的状态

在我们的子组件中加入如下代码
````jsx
shouldComponentUpdate(nextProps,nextState){
    if(nextProps.content !== this.props.content){
        return true
    }else{
        return false
    }
}
````

这就实现了按照前后变化属性是否相同为依据的“按需加载”，优化后我们再次通过浏览器插件进行验证：

![](https://img.masaiqi.com/2019-09-03-094234.png)

可以看到，这里我们的列表渲染值渲染了一个子组件，而且只用了不到0.1ms，比之前的3ms相比快了太多。

## React发起Axios请求

1. 安装Axios
````shell
npm install -save axios
````

2. 在需要Axios的组件上方导入
````js
import axios from 'axios'
````

3. 尝试在`componentDidMount`中发送Axios请求
````js
componentDidMount(){
    axios.post('https://web-api.juejin.im/v3/web/wbbr/bgeda')
        .then((res)=>{console.log('axios 获取数据成功:'+JSON.stringify(res))  })
        .catch((error)=>{console.log('axios 获取数据失败'+error)})
}
````

PS：这里有可能存在跨域问题，后续有时间回来补坑。

扩展：

- npm install xxx: 安装项目到项目目录下，不会将模块依赖写入devDependencies或dependencies。
  
- npm install -g xxx: -g的意思是将模块安装到全局，具体安装到磁盘哪个位置，要看 npm cinfig prefix的位置
  
- npm install -save xxx：-save的意思是将模块安装到项目目录下，并在package文件的dependencies节点写入依赖。
  
- npm install -save-dev xxx：-save-dev的意思是将模块安装到项目目录下，并在package文件的devDependencies节点写入依赖。

## 参考资料

- [技术胖的React教程](http://jspang.com/posts/2019/05/04/new-react-base.html)（很感谢胖哥的React文章~我一个后端被安排的明明白白的哈哈）
- [React官方文档](https://zh-hans.reactjs.org/docs/hello-world.html)(看完胖哥教程入了门后必看)
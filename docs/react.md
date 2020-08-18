##### 组件

vue组件定义方式：<font color="#8B0000">template</font>结构、<font color="#8B0000">script</font>行为、<font color="#8B0000">style</font>样式。

<template>
    <div class="form-list form-list-two">
        <el-form :model="data">
            <el-form-item>
                <el-col :span="11" class="form-title">&nbsp;</el-col>
            </el-form-item>
        </el-form>
    </div>
</template>
<script>
    import i18n from '@scripts/i18n.js';
    export default {
        name:"LocationBPCfg",
        data(){
            return {
            }
        },
        props: {
            dataSource: {}//父组件中使用子组件，传递的参数
        },
        created:function(){
            //组件初始化钩子事件
        },
        mounted:function(){
            //组件初始化钩子事件
        },
        methods:{
            // 定义组件内事件方法
        },
        components:{//引用子组件
             "TSKU": resolve => { require(['@warehouseselector/skuselector.vue.html'], resolve) },
        },
    }
</script>
<style scoped>
react组件定义方式：

1. 函数定义

   ~~~javascript
   function HelloMessage(props) {
       return <h1>Hello World!</h1>;
   }
   ~~~

2. ES6class定义

   ~~~javascript
   class Welcome extends React.Component {
     render() {
       return <h1>Hello World!</h1>;
     }
   }
   ~~~

~~~javascript
function Name(props) {
    return <h1>网站名称：{props.name}</h1>;
}
function Url(props) {
    return <h1>网站地址：{props.url}</h1>;
}
function Nickname(props) {
    return <h1>网站小名：{props.nickname}</h1>;
}
function App() {
    return (
    <div>
        <Name name="菜鸟教程" />
        <Url url="http://www.runoob.com" />
        <Nickname nickname="Runoob" />
    </div>
    );
}
 
ReactDOM.render(
     <App />,
    document.getElementById('example')
);
~~~

##### react核心概念

- <font color="#1E90FF">虚拟DOM</font>
  - dom的本质：浏览器中的概念，用js对象来标识页面上的元素，并提供了操作dom对象的api；
  - 什么是react中的虚拟dom：是框架中的概念，开发者用js对象来模拟页面上的dom和dom嵌套；
  - 虚拟dom的目的：为了实现页面中，dom元素的高效更新；
- <font color="#1E90FF">Diff算法</font>
  - tree diff：新旧两棵dom树逐层对比的过程，找出所有需要被按需更新的元素；
  - component diff：dom树中每一层会有很多组件component ，组件各自进行对比，找出差异组件，移除旧组件，更新为新组件；
  - element diff：组件中的元素element(例如：<p></p>)，如果两个组件类型相同，则需要进行元素级别的对比；

##### 应用

~~~c#
在vscode中无法识别npm指令？//以管理员身份运行vscode
系统不支持脚本执行？//以管理员身份运行shell窗体执行：set-executionpolicy remotesigned
~~~

1. 安装node.js，并配置国内镜像cnpm：

   ~~~java
   npm install -g cnpm --registry=https://registry.npm.taobao.org
   ~~~

2. 安装webpack

   ~~~java
   cnpm install -g webpack
   cnpm install -g webpack-cli //4.x版本将cli分离出来
   cnpm install -g webpack-dev-server //支持自动检测修改，重新编译，生成的main.js文件在根目录下，托管在内存中(在项目根目录中看不到)
    //同时需要在配置package.json中配置dev
   "scripts": {
       "test": "echo \"Error: no test specified\" && exit 1",
       "dev":"webpack-dev-server --open --prot 3000 --hot"
   }
   ~~~

   ~~~java
   //html-webpack-plugin插件，用于将html生成到内存中，加快编译速度
   ~~~

3. 安装react

   ~~~java
   react//专门用于创建组件和虚拟DOM的，同时组件的生命周期都在这个包中
   react-dom//专门进行DOM操作的，最主要应用场景：ReactDOM.render()
   cnpm i react react-dom -S //-s表示生产环境，-D表示开发环境
   ~~~

4. 创建元素

   ~~~javascript
   //1.导入包
   import React from 'react' //创建组件、虚拟dom元素，生命周期
   import ReactDOM from 'react-dom'//把创建好的组件和虚拟dong放到页面显示
   
   //2.创建dom元素
   //参数：1元素名称str，2元素属性class，3子节点(包括其它虚拟dom)n...
   const myh1 = React.createElement('h1',{id:'myh1'},'你好呀h11111')
   
   //3.渲染到页面上
   //参数：1要渲染的那个虚拟dom元素，2指定页面上的一个容器
   ReactDOM.render(myh1,document.getElementById('app'))
   ~~~

5. 创建元素JSX语法

   即在js中混合写入类似HTML的语法，其本质也是被转换成`React.createElement()`，被渲染。

   ~~~java
   // 1.安装babel插件，支持jsx语法
   cnpm i -g babel-core babel-loader@7 babel-plugin-transform-runtime -D
   cnpm i -g babel-preset-env babel-preset-stage-0 -D
   //安装能够识别jsx语法的包
   cnpm i -g babel-preset-react -D
   ~~~

   ~~~json
   //2.在webpack.config.js文件中配置rule，自定义：.js|.jsx文件处理方式，使用babel
   module:{
           //所有第三方模块的配置规则
           rules:[
               {text:/\.js|jsx$/,use:'babel-loader',exclude:/node_modues/}
           ]
       }
   ~~~

   ~~~json
   //3.根目录下新增.babelrc文件
   {
       "presets":["env","stage-0","react"],
       "plugins":["transform-runtime"]
   }
   ~~~

6. 编写组件

   ~~~javascript
   import React from 'react';
   export default function Hello(props){
       return <div>{props.name}<div>;
   }
   ~~~

   ~~~javascript
   // 父组件引用
   import Hello from '@/components/hello';
   //使用 <Hello name={}></Hello>; 
   ~~~

   ~~~javascript
   //在webpack.config.js文件中module同级新增节点：
   resolve:{
           extensions:['.js','jsx'],//表示这几个文件的后缀可以不写，自动补全
           alias:{
               '@':path.join(__dirname,'./src')//表示根目录到src这一层路径
           }
       }
   ~~~

7. class使用

   ~~~javascript
   // 静态方法只能类直接访问，实例方法通过实例化对象访问
   class Animal{
       constructor(name,age){// 构造器
           this.name = name;
           this.age = age;// 实例属性（react常用）
       }
       static color = "red" // 静态属性
       static say(){
           // 静态方法
       }
       func(){
           // 实例方法（react常用）
       }
   }
   // 子类继承父类
   class Dog extends Animal{
       // 语法规范：子类自定义构造器时，必须调用super()
        constructor(name,age,height){// 构造器
          super(name,age);
          this.height = height;
       }
   }
   
   let mydog = new Dog('',4);// 子类使用父类构造函数
   mydog.func();// 实例方法
   ~~~

8. class创建组件

   ~~~javascript
   //1.继承React.Component
   //2.类中必须包含render实例方法，返回值为jsx
   import React from 'react';
   class MyComponent extends React.Component{
       render(){// 渲染当前虚拟dom
           return <div>{this.props.name}<div>;
       }
       constructor(){
          super();
          this.state = {
              // 定义组件内自己的数据，类似vue中的data
          }
       }
   }
   ~~~

   ~~~javascript
   const user = {name:'张三',age:10,six:'男'}
   <MyComponent {...user}><MyComponent> //{...user}方式传参，表示属性全部展开传入
   ~~~

   <font color="red">注意：对class创建的组件传参时，class不需要手动接收，直接通过`this.props`使用</font>

   <font color="#1E90FF">优点</font>：

   1. 有自己的私有属性、方法和生命周期
   2. 有状态组件，即有<font color="#8B0000">state</font>属性

9. 生命周期

   ~~~javascript
   /*vue生命周期钩子函数*/：
   created（此时组件的data和method已经可用，但页面还未渲染，可向服务器发起ajax请求）
   mounted（组件创建阶段最后一个生命周期函数，页面已经渲染完成，组件进入运行中阶段）
   ~~~

   ~~~javascript
   /*react生命周期钩子函数*/：
   //创建阶段
   componentWillMount：//组件将要挂载
   render：//在内存中创建虚拟DOM
   componentDidMount：//完成组件挂载
   //运行阶段
   componentWillReceiveProps：//props改变时，触发：组件将要接收新的属性props
   shouldComponentUpdate：//组件是否需要被更新(方法返回true时更新组件，false不更新，但是数据最新)
   componentUpdate：//组件将要更新
   render：//在内存中重新渲染虚拟DOM
   componentDidUpdate://完成组件更新（state和组件都是最新的）
   //销毁阶段
   componentWillUnmount://组件将要被卸载
   ~~~

   
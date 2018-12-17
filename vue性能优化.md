# vue性能优化  
# 组件优化
一般来说，你不需要太关心vue的运行时性能，它在运行时非常快，但付出的代价是初始化时相对较慢。先看一下常见的vue写法：在html里放一个app组件，app组件里又引用了其他的子组件，形成一棵以app为根节点的组件树。  
```
 <div id="app">
    <router-view></router-view>
  </div>  
  ```
  而正是这种做法引发了性能问题，要初始化一个父组件，必然需要先初始化它的子组件，而子组件又有它自己的子组件。那么要初始化根标签，就需要从底层开始冒泡，将页面所有组件都初始化完。所以我们的页面会在所有组件都初始化完才开始显示。

这个结果显然不是我们要的，更好的结果是页面可以从上到下按顺序流式渲染，这样可能总体时间增长了，但首屏时间缩减，在用户看来，页面打开速度就更快了。
## 异步组件  
```
new Vue({
    components: {
        A: { /*component-config*/ },
        B (resolve) {
            setTimeout(() => {
                resolve({ /*component-config*/ })
            }, 0);
        }
    }
})
```
或者是
```
const Foo = resolve => {
  // require.ensure 是 Webpack 的特殊语法，用来设置 code-split point
  // （代码分块）
  require.ensure(['./Foo.vue'], () => {
    resolve(require('./Foo.vue'))
  })
}  
const Foo = resolve => require(['./Foo.vue'], resolve)   
const router = new VueRouter({
  routes: [
    { path: '/foo', component: Foo }
  ]
})   
```
把不同路由对应的组件分割成不同的代码块，然后当路由被访问的时候才加载对应组件，从而实现路由懒加载  
## 把组件按组分块  
有时候我们想把某个路由下的所有组件都打包在同个异步 chunk 中。只需要 给 chunk 命名，提供 require.ensure 第三个参数作为 chunk 的名称:
```
const Foo = r => require.ensure([], () => r(require('./Foo.vue')), 'group-foo')
const Bar = r => require.ensure([], () => r(require('./Bar.vue')), 'group-foo')
const Baz = r => require.ensure([], () => r(require('./Baz.vue')), 'group-foo')  
```
## 利用v-if和terminal  
```
 <head>
   <!--some component -->
  <div v-if="showB">
   <!--some component -->
    </div>
     <div v-if="showC">
   <!--some component -->
    </div>
</head>
 data: {
        showB: false,
        showC: false
    },
    created () {
        // 显示B
        setTimeout(() => {
            this.showB = true;
        }, 0);
        // 显示C
        setTimeout(() => {
            this.showC = true;
        }, 0);
    }  
 ```
这个示例写起来略显啰嗦，但它已经实现了我们想要的顺序渲染的效果。页面会在组件初始化完后显示，然后再按顺序渲染其余的组件，整个页面渲染方式看起来是流式的。

有些人可能会担心v-if存在一个编译/卸载过程，会有性能影响。但这里并不需要担心，因为v-if是惰性的，只有当第一次值为true时才会开始初始化。  
## 组件keep-alive  
如果你做用一个大型web的spa的时候，你有很多router，对应的是很多个页面。在页面的快速切换中（如常见的tab页），为了保证页面加载的效率，除了缓存机制之外，vue的keep-alive组件可以帮的上忙。它会把组件保存在浏览器内存当中，方便你快速切换。
# 基础优化
## v-if v-show
权限问题，只要涉及到权限相关的展示无疑要用 v-if，没有权限限制下根据用户点击的频次选择，频繁切换的使用 v-show，不频繁切换的使用 v-if，这里要说的优化点在于减少页面中 dom 总数，我比较倾向于使用 v-if，因为减少了 dom 数量，加快首屏渲染。
不要在模板里面写过多的表达式与判断 v-if="isShow && isAdmin && (a || b)"，这种表达式虽说可以识别，但是不是长久之计，当看着不舒服时，适当的写到 methods 和 computed 里面封装成一个方法，这样的好处是方便我们在多处判断相同的表达式，其他权限相同的元素再判断展示的时候调用同一个方法即可。  

循环尽可能在使用 v-for 时提供 key，除非遍历输出的 DOM 内容非常简单，或者是刻意依赖默认行为以获取性能上的提升。引用vue文档：当 Vue.js 用 v-for 正在更新已渲染过的元素列表时，它默认用“就地复用”策略。如果数据项的顺序被改变，Vue 将不会移动 DOM 元素来匹配数据项的顺序， 而是简单复用此处每个元素，并且确保它在特定索引下显示已被渲染过的每个元素。这个类似 Vue 1.x 的 track-by="$index" 。

这个默认的模式是高效的，但是只适用于不依赖子组件状态或临时 DOM 状态 (例如：表单输入值) 的列表渲染输出。

为了给 Vue 一个提示，以便它能跟踪每个节点的身份，从而重用和重新排序现有元素，你需要为每项提供一个唯一 key 属性。理想的 key 值是每项都有的唯一 id。
watch 和 computed 用哪个的问题看官网的例子，计算属性主要是做一层 filter 转换，切忌加一些调用方法进去，watch 的作用就是监听数据变化去改变数据或触发事件如 this.$store.dispatch('update', { ... })
数据请求在哪个时候尽量根据需求考虑后，多多利用promise的并发请求。
# 其他
如打包优化：在打包时可将一些静态模块排除，如ue、vuex、vue-router、axios 等，换用国内的 bootcdn 直接引入到根目录的 index.html 中。在 webpack 里有个 externals，可以忽略不需要打包的库
```
externals: {
  'vue': 'Vue',
  'vue-router': 'VueRouter',
  'vuex': 'Vuex',
  'axios': 'axios'
}
```
或者是利用webpack的动态链接库DllPlugin，打包会输出一个类dll包（dll包源于windows的动态链接库），这些代码本身不会执行，主要是提供给我们的业务代码引用。将静态资源文件（运行依赖包）与源文件分开打包，先使用DllPlugin给静态资源打包，再使用DllReferencePlugin让源文件引用资源文件。dll在打包一次后即可，下次业务代码修改也不会重新打包，然后在html里分模块引入
```
   <script src="./static/js/vendor.dll.js"></script>
    <script src="/dist/build.js"></script>
```
关于webpack的使用建议阅读《深入浅出webpack》一书。
其他的还有样式优化、减少组件耦合性等。

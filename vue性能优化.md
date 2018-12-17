# vue性能优化  
一般来说，你不需要太关心vue的运行时性能，它在运行时非常快，但付出的代价是初始化时相对较慢。先看一下常见的vue写法：在html里放一个app组件，app组件里又引用了其他的子组件，形成一棵以app为根节点的组件树。  
 <div id="app">
    <router-view></router-view>
  </div>  
  而正是这种做法引发了性能问题，要初始化一个父组件，必然需要先初始化它的子组件，而子组件又有它自己的子组件。那么要初始化根标签，就需要从底层开始冒泡，将页面所有组件都初始化完。所以我们的页面会在所有组件都初始化完才开始显示。

这个结果显然不是我们要的，更好的结果是页面可以从上到下按顺序流式渲染，这样可能总体时间增长了，但首屏时间缩减，在用户看来，页面打开速度就更快了。
## 异步组件  

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
或者是
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
把不同路由对应的组件分割成不同的代码块，然后当路由被访问的时候才加载对应组件，从而实现路由懒加载  
## 把组件按组分块  
有时候我们想把某个路由下的所有组件都打包在同个异步 chunk 中。只需要 给 chunk 命名，提供 require.ensure 第三个参数作为 chunk 的名称:
const Foo = r => require.ensure([], () => r(require('./Foo.vue')), 'group-foo')
const Bar = r => require.ensure([], () => r(require('./Bar.vue')), 'group-foo')
const Baz = r => require.ensure([], () => r(require('./Baz.vue')), 'group-foo')  

## 利用v-if和terminal  
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
    这个示例写起来略显啰嗦，但它已经实现了我们想要的顺序渲染的效果。页面会在组件初始化完后显示，然后再按顺序渲染其余的组件，整个页面渲染方式看起来是流式的。

有些人可能会担心v-if存在一个编译/卸载过程，会有性能影响。但这里并不需要担心，因为v-if是惰性的，只有当第一次值为true时才会开始初始化。  


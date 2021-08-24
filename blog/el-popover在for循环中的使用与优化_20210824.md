## el-popover 在 v-for 中的 基础使用
```javascript
<div v-for='item in 100'>
    <el-popover
        placement="bottom"
        title="标题"
        width="200"
        trigger="click"
        content="这是一段内容,这是一段内容,这是一段内容,这是一段内容。">
        <el-button slot="reference">click 激活</el-button>
    </el-popover>
</div>
```
这样使用的缺点是pop-over内嵌的内容回多次重复渲染，在数据量不大的时候影响不大，如果pop-over嵌套table等其他组件会导致效率低下，多次重复渲染。当然为了完成需求可以首先使用这种方法

## 如何优化
优化思路就是将el-popover提出来，不参与循环，让el-popover只渲染一次，这样在首屏渲染时，速度就会大大提升。这样就有两个问题需要解决:
1. 如何将popover slot中的reference与for循环中的button关联起来，用来确定popover的出现位置。
2. 如何触发popover显示与关闭


el-popover有几种激活方式，分click与v-model等。
1. click模式下，需要将button作为reference slot，只有点击reference才会显示popover，点击非popover区域会自动关闭popover，但是我们需要将popover提出来，而reference在v-for中渲染，自然不可能使用slot。 
2. 使用v-model时可以很方便控制显示与关闭但是点击非popover区域无法自动关闭。

### 查看el-popover 源代码寻找解决办法
#### element-ui 2.15 其他版本自行查看代码是否一致
```javascript
props: {
    // 有reference prop 但是官方文档未写这个参数
    reference: {}
}


mounted()
{
    // 这块是确定 reference的逻辑 首先获取props参数 没有就获取refs，所以我们可以将refence当作props传递给el-popover组件
    let reference = this.referenceElm = this.reference || this.$refs.reference;
    ...
    // 可以看到当trigger为click时监听了点击事件 当点击不是popover的区域时自动隐藏popover以及点击reference自身进行toggle操作
    if (this.trigger === 'click') {
        on(reference, 'click', this.doToggle);
        on(document, 'click', this.handleDocumentClick);
    }
}

methods: {
    // 这几个方法主要是控制popover 显示与隐藏的 我们可以使用这几个方法
    doToggle()
    {
        this.showPopper = !this.showPopper;
    },
    doShow()
    {
        this.showPopper = true;
    },
    doClose()
    {
        this.showPopper = false;
    }
}
```
#### 确定思路
1. 使用reference prop将v-for中的button当作参数传递给el-popover，这样就可以不使用slot而将reference与el-popover分离开了
2. trigger使用click实现点击其他区域自动隐藏popover以及点击reference进行toggle操作


#### talk is cheap,show you the code
```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <link href="https://www.unpkg.com/element-ui@2.15.3/lib/theme-chalk/index.css" rel="stylesheet">
</head>
<body>
<script src="https://www.unpkg.com/vue/dist/vue.js"></script>
<script src="https://www.unpkg.com/element-ui@2.15.3/lib/index.js"></script>
<div id="app">
</div>
<template id="test">
    <div>
        <div v-for='item in 100'>
            <el-button @click='clickPop(item)' :ref='`bt`+item'>click 激活</el-button>
        </div>
        <el-popover
                v-if='showPop'
                ref='pop'
                :reference='reference'
                placement="bottom"
                title="标题"
                width="200"
                trigger="click"
        >
            <el-button @click='$refs.pop.doClose()'>自定义关闭按钮</el-button>
        </el-popover>
    </div>
</template>
<script>
    var Main = {
        template:'#test',
        beforeCreate(){
            console.time('per')
        },
        mounted(){
            let time = console.timeEnd('per')
        },
        data() {
            return {
                reference:{},
                // 控制渲染条件 如果不用v-if会报错 具体可自行尝试
                showPop: false,
                // 保存当前激活的refrence id
                activeId:'',
            };
        },
        methods:{
            clickPop(item){
                // 这个操作是为了避免与源码中的点击reference doToggle方法冲突
                if (this.activeId === item && this.showPop) return
                this.showPop = false
                this.activeId = item
                // 因为reference是需要获取dom的引用 所以需要是$el
                this.reference = this.$refs['bt'+item][0].$el
                this.$nextTick(() => {
                    // 等待显示的popover销毁后再 重新渲染新的popover
                    this.showPop = true
                    this.$nextTick(() => {
                        // 此时才能获取refs引用
                        this.$refs.pop.doShow()
                    })
                })
            }
        }
    };
    var Ctor = Vue.extend(Main)
    new Ctor().$mount('#app')
</script>
</body>
</html>
```

### 性能分析

1. 首先el-popover不参与循环肯定会大大缩短首次渲染时间，但是如果循环次数少或者el-popover嵌套的内容不复杂，其实也没必要提出来
2. 缺点就是 点击的时候再渲染popover等待时间会相对比for循环渲染好要长。
3. 如果想减少进页面的首次渲染时间可以提出popover，如果想减少点击显示popover的时间就在for循环中提前渲染好

#### 首屏渲染时间对比

在本地打开html，分别在beforeCreate 和 mounted中打印，比较渲染时间，具体时间要看各自电脑性能，下面写的是在i7的情况下
```javascript
        beforeCreate(){
            console.time('per')
        },
        mounted(){
            let time = console.timeEnd('per')
        },
```

1. 上述代码将popover参与循环的情况下为 per: 80.890869140625 ms
2. 不参与循环的情况下为 per: 30.35498046875 ms

对比下时间快了两倍多，如果popover嵌套el-table等复杂内容时首屏渲染时间提升效果会更明显

[在线运行](https://codepen.io/chanbaihai/pen/abwbEzQ?editors=1111) 或者直接复制上面的html本地打开
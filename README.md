## Vue 组件间通信

------

#### 1.`props`/`$emit`

##### 1-1.父组件向子组件传值(parent to child)

```html
<div id="app">
    <template>
        <com-article :articles="articleList"></com-article>
    </template>
</div>
<script>
    let vm = new Vue({
        el: "#app",
        data() {
            return {
                articleList:['红楼梦', '西游记','三国演义']
            }
        },
        methods:{

        },
        components:{
            ComArticle:{
                template:`<div><span v-for="(item,index) of articles" :key="index" v-text="item"></span></div>`,
                props:{
                    articles:Array
                }
            }
        }
    })
</script>
```

> 总结: `prop`只可以从上一级组件传递到下一级组件（父子组件），即所谓的单向数据流。而且 `prop`只读，不可被修改，所有修改都会失效并警告。

##### 1-2.子组件向父组件传值(child to parent)

```html
<div id="app">
    <template>
        <com-article :articles="articleList" @onemitarg="emitReceive"></com-article>
        <span>传递的值为:{{itemIndex}}</span>
    </template>
</div>
<script>
    let vm = new Vue({
        el: "#app",
        data() {
            return {
                itemIndex:null,
                articleList:['红楼梦', '西游记','三国演义']
            }
        },
        methods:{
            emitReceive(arg){
                this.itemIndex = arg;
            }
        },
        components:{
            ComArticle:{
                template:`<div><span v-for="(item,index) of articles" :key="index" v-text="item" @click="emitArg(index)"></span></div>`,
                props:{
                    articles:Array
                },
                methods:{
                    emitArg(arg){
                        this.$emit("onemitarg",arg);
                    }
                }
            }
        }
    })
</script>
```

> 子组件标签`com-article`中`@onemitarg`只能为小写或者`on-emit-arg`(kebab-case),不能还有大写字母,因为标签的大小写敏感.

------

#### 2.`$parent`/`$child`[^1]

示例见`<com-a>`:

```html
<div id="app">
    <template>
        <com-article :articles="articleList" @onemitarg="emitReceive"></com-article>
        <span>传递的值为:{{itemIndex}}</span>\
        <com-a></com-a>
        <button @click="changeMsg">更改子组件msg</button>
    </template>
</div>

<script>
    let vm = new Vue({
        el: "#app",
        data() {
            return {
                itemIndex:null,
                articleList:['红楼梦', '西游记','三国演义'],
                msg:"this is from parent"
            }
        },
        methods:{
            emitReceive(arg){
                this.itemIndex = arg;
            },
            changeMsg(){
                // console.log(this.$children);
                this.$children[1].parentMsg = "this is changed for child";
            }
        },
        components:{
            ComArticle:{
                template:`<div><span v-for="(item,index) of articles" :key="index" v-text="item" @click="emitArg(index)"></span></div>`,
                props:{
                    articles:Array
                },
                methods:{
                    emitArg(arg){
                        this.$emit("onemitarg",arg);
                    }
                }
            },
            ComA:{
                template:"<div><span>{{parentMsg}}</span></div>",
                data(){
                    return {
                        parentMsg:this.$parent.msg
                    }
                }
            }
        }
    })
</script>
```

> **但是**,当存在多层parent组件是会出现`this.$parent.$parent...`的情况,这样很容易out of hand,因此要向更深层级的后代后代组件(descendent components)建议用依赖注入(dependency injection)来代替.

------

#### 3.依赖注入(dependency injection)[^2]

`provide`/ `inject` 是`vue2.2.0`新增的api, 简单来说就是父组件中通过`provide`来提供变量, 然后再子组件中通过`inject`来注入变量。

> 注意: 这里不论子组件嵌套有多深, 只要调用了`inject` 那么就可以注入`provide`中的数据，而不局限于只能从当前父组件的props属性中回去数据.

示例见`<com-a>`和`<com-b>`:

```html
<div id="app">
    <template>
        <com-article :articles="articleList" @onemitarg="emitReceive"></com-article>
        <span>传递的值为:{{itemIndex}}</span>\
        <com-a></com-a>
        <button @click="changeMsg">更改子组件msg</button>
        <com-b></com-b>
    </template>
</div>

<script>
    let vm = new Vue({
        el: "#app",
        data() {
            return {
                itemIndex:null,
                articleList:['红楼梦', '西游记','三国演义'],
                msgForA:"this is from parent",
                msgForBC:"this is msg from root com for B and C"
            }
        },
        provide(){
            return {
                msgForBC:this.msgForBC
            }
        },
        methods:{
            emitReceive(arg){
                this.itemIndex = arg;
            },
            changeMsg(){
                // console.log(this.$children);
                this.$children[1].parentMsg = "this is changed for child";
            }
        },
        components:{
            ComArticle:{
                template:`<div><span v-for="(item,index) of articles" :key="index" v-text="item" @click="emitArg(index)"></span></div>`,
                props:{
                    articles:Array
                },
                methods:{
                    emitArg(arg){
                        this.$emit("onemitarg",arg);
                    }
                }
            },
            ComA:{
                template:"<div><span>组件A:{{parentMsg}}</span></div>",
                data(){
                    return {
                        parentMsg:this.$parent.msgForA
                    }
                }
            },
            ComB:{
                template:`<div><div>组件B:{{msgForBC}}</div><com-c></com-c></div>`,
                inject:["msgForBC"],
                components:{
                    ComC:{
                        template:`<div><div>组件C:{{msg}}</div></div>`,
                        inject:["msgForBC"],
                        data(){
                            return {
                                msg:this.msgForBC
                            }
                        }
                    }
                }
            }
        }
    })
</script>
```

> **然而**，依赖注入还是有负面影响的。它将你应用程序中的组件与它们当前的组织方式耦合起来，使重构变得更加困难。同时所提供的 property 是非响应式的。这是出于设计的考虑，因为使用它们来创建一个中心化规模化的数据跟[使用 `$root`](https://cn.vuejs.org/v2/guide/components-edge-cases.html#访问根实例)做这件事都是不够好的。如果你想要共享的这个 property 是你的应用特有的，而不是通用化的，或者如果你想在祖先组件中更新所提供的数据，那么这意味着你可能需要换用一个像 [Vuex](https://github.com/vuejs/vuex) 这样真正的状态管理方案了。

------

#### 4.ref[^3]/$refs[^4]

`ref` 被用来给元素或子组件注册引用信息。引用信息将会注册在父组件的 `$refs` 对象上。如果在普通的 DOM 元素上使用，引用指向的就是 DOM 元素；如果用在子组件上，引用就指向组件实例(参考`<com-a>`)：

```html
<div id="app">
    <template>
        <com-article :articles="articleList" @onemitarg="emitReceive"></com-article>
        <span>传递的值为:{{itemIndex}}</span>\
        <com-a ref="coma"></com-a>
        <button @click="changeMsg">更改子组件msg</button>
        <com-b></com-b>
        <p v-if="msgFromRefComa">来自com-a的msg:{{msgFromRefComa}}</p>
    </template>
</div>

<script>
    let vm = new Vue({
        el: "#app",
        data() {
            return {
                itemIndex:null,
                articleList:['红楼梦', '西游记','三国演义'],
                msgForA:"this is from parent",
                msgForBC:"this is msg from root com for B and C",
                msgFromRefComa:null
            }
        },
        provide(){
            return {
                msgForBC:this.msgForBC
            }
        },
        methods:{
            emitReceive(arg){
                this.itemIndex = arg;
            },
            changeMsg(){
                // console.log(this.$children);
                this.$children[1].parentMsg = "this is changed for child";
            }
        },
        components:{
            ComArticle:{
                template:`<div><span v-for="(item,index) of articles" :key="index" v-text="item" @click="emitArg(index)"></span></div>`,
                props:{
                    articles:Array
                },
                methods:{
                    emitArg(arg){
                        this.$emit("onemitarg",arg);
                    }
                }
            },
            ComA:{
                template:"<div><span>组件A:{{parentMsg}}</span></div>",
                data(){
                    return {
                        parentMsg:this.$parent.msgForA,
                        msg:"this is from <com-a>"
                    }
                }
            },
            ComB:{
                template:`<div><div>组件B:{{msgForBC}}</div><com-c></com-c></div>`,
                inject:["msgForBC"],
                components:{
                    ComC:{
                        template:`<div><div>组件C:{{msg}}</div></div>`,
                        inject:["msgForBC"],
                        data(){
                            return {
                                msg:this.msgForBC
                            }
                        }
                    }
                }
            }
        },
        mounted(){
            this.msgFromRefComa = this.$refs.coma.msg;
        }
    })
</script>
```

> 关于 `ref` 注册时间的重要说明：因为 `ref `本身是作为渲染结果被创建的，在初始渲染的时候你不能访问它们 - 它们还不存在！`$refs` 只会在组件渲染完成之后生效，并且它们不是响应式的。这仅作为一个用于直接操作子组件的“逃生舱”——你应该避免在模板或计算属性中访问 `$refs`,不应该试图用它在模板中做数据绑定。

------

#### 5.eventBus[^5]

eventBus 又称为事件总线，在vue中可以使用它来作为沟通桥梁的概念, 就像是所有组件共用相同的事件中心，可以向该中心注册发送事件或接收事件， 所以组件都可以通知其他组件。

eventBus也有不方便之处, 当项目较大,就容易造成难以维护的灾难

在Vue的项目中怎么使用eventBus来实现组件之间的数据通信呢?具体通过下面几个步骤(vue.js开发模式演示):

##### 5-1:初始化:首先需要创建一个事件总线并将其导出, 以便其他模块可以使用或者监听它:

```javascript
// event-bus.js

const eventBus = new Vue();//其实eventBus为Vue的一个实例
```

##### 5-2:创建两个组件: `additionNum` 和 `showNum`为兄弟组件:

```html
<div id="app">
    <template>
        <show-num-com></show-num-com>
        <add-num-com></add-num-com>
    </template>
</div>

<script>
    Vue.prototype.eventBus = eventBus;//Vue的原型对象中创建eventBus等于event_bus.js中创建的eventBus实例,这样任何Vue实例都能调用eventBus实例中的方法
    let vm = new Vue({
        el: "#app",
        components:{
            showNumCom:{
                name:"show",
                template:`<div><ul><li v-for="(item,i) of numArray" :key="i" v-text="item"></li></ul></div>`,
                data(){
                    return {
                        numArray:[2,4,5]
                    }
                },
                methods:{
                    add(num){
                        this.numArray.push(num);
                    }
                },
                mounted(){
                    this.eventBus.$on("addItem",this.add.bind(this));//给eventBus添加事件addItem
                }
            },
            addNumCom:{
                name:"add",
                template:`<div><input type="text"placeholder="请输入你要添加的数字" v-model="num"><button @click="add">添加数字</button></div>`,
                data(){
                    return {
                        num:null
                    }
                },
                methods:{
                    add(){
                        this.eventBus.$emit("addItem",this.num);//触发eventBus实例里的事件addItem并给绑定的函数传参
                        this.num = null;
                    }
                }
            }
        }	
    })
</script>
```

这样就能实现兄弟组件的传参,`eventBus`作为兄弟组件沟通的桥梁,在`show-num-com`组件中在`eventBus`实例中添加`addItem`事件绑定`add`函数,在`add-num-com`组件中向`addItem`事件所绑定的`add`函数传参,最终实现为数组push元素.

#### 6.Vuex

Vuex 是一个专为 Vue.js 应用程序开发的**状态管理模式**。它采用集中式存储管理应用的所有组件的状态，并以相应的规则保证状态以一种可预测的方式发生变化。

它解决了"单向数据流"容易产生的问题:

- 多个视图依赖于同一状态。
- 来自不同视图的行为需要变更同一状态。

对于问题一，传参的方法对于多层嵌套的组件将会非常繁琐，并且对于兄弟组件间的状态传递无能为力。

对于问题二，我们经常会采用父子组件直接引用或者通过事件来变更和同步状态的多份拷贝。以上的这些模式非常脆弱，通常会导致无法维护的代码。

通过定义和隔离状态管理中的各种概念并通过强制规则维持视图和状态间的独立性，我们的代码将会变得更结构化且易维护,将开发者的精力聚焦于数据的更新而不是数据在组件之间的传递上。

如果您的应用够简单，您最好不要使用 Vuex。一个简单的 [store 模式](https://cn.vuejs.org/v2/guide/state-management.html#简单状态管理起步使用)就足够您所需了。但是，如果您需要构建一个中大型单页应用，您很可能会考虑如何更好地在组件外部管理状态，Vuex 将会成为自然而然的选择。



[^1]:Vue官方文档 [https://cn.vuejs.org/v2/guide/components-edge-cases.html#%E8%AE%BF%E9%97%AE%E7%88%B6%E7%BA%A7%E7%BB%84%E4%BB%B6%E5%AE%9E%E4%BE%8B](https://cn.vuejs.org/v2/guide/components-edge-cases.html#访问父级组件实例)
[^2]:Vue官方文档 [https://cn.vuejs.org/v2/guide/components-edge-cases.html#%E4%BE%9D%E8%B5%96%E6%B3%A8%E5%85%A5](https://cn.vuejs.org/v2/guide/components-edge-cases.html#依赖注入)
[^3]:Vue官方文档 https://cn.vuejs.org/v2/api/#ref
[^4]:Vue官方文档 [https://cn.vuejs.org/v2/guide/components-edge-cases.html#%E8%AE%BF%E9%97%AE%E5%AD%90%E7%BB%84%E4%BB%B6%E5%AE%9E%E4%BE%8B%E6%88%96%E5%AD%90%E5%85%83%E7%B4%A0](https://cn.vuejs.org/v2/guide/components-edge-cases.html#访问子组件实例或子元素)

[^5]:掘金文章 [https://juejin.im/search?query=%E5%85%84%E5%BC%9F%E7%BB%84%E4%BB%B6%E4%BC%A0%E5%8F%82&type=all](https://juejin.im/search?query=兄弟组件传参&type=all)
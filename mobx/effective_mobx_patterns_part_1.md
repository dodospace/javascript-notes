# 高效的Mobx模式 - （Part 1）

MobX提供了一种简单而强大的方法来管理客户端状态。 它使用一种称为透明功能反应编程（TFRP-`Transparent Functional Reactive Programming`）的技术，其中如果任何相关值发生变化，它会自动计算派生值。 也就是通过设置跟踪值更改的依赖关系图来完成的。

MobX导致思维方式的转变（`For the better`），并改变您的心理模型围绕管理客户端状态。

## Part 1 - 构建可观察者

当我们使用**Mobx**时，建立客户端状态模型是第一步， 这是最有可能被客户端上呈现你的域模型的直接体现。实际上客户端状态也就是我们通常说的`Store`，了解redux的对此都很熟悉，虽然你只有一个**Store**，但是它是由多个子**Store**组件的，每一个子**Store**用来处理应用程序的各种功能。

最简单的入门方法是注释**Store**的属性，这些属性将随着`@observable`而不断变化。 请注意，我使用的是装饰器语法，但使用简单的ES5函数包装器可以实现相同的功能。

```javascript
import { observable } from 'mobx'
class MyStore {
	@observable name
	@observable description
	@observable author

	@observable photos = [] 
}
```

### 修剪可观察属性
通过将对象标记为`@observable`，您将自动观察其所有嵌套属性。 现在这可能是你想要的东西，但很多时候它更能限制可观察性。 你可以使用一些MobX修饰符来做到这一点：

#### `asReference`
当某些属性永远不会改变值时，这是非常有用的。 请注意，如果您确实更改了引用本身，它将触发更改。
```javascript
let address = new Address();
let contact = observable({
    person: new Person(),
    address: asReference(address)
});

address.city = 'New York'; // 不会触发通知任何

// 将触发通知，因为这是属性引用更改
contact.address = new Address();
```
在上面的示例中，address属性将不可观察。 如果您更改地址详细信息，则不会收到通知。 但是，如果您更改地址引用本身，您将收到通知。

一个有趣的消息是一个可观察对象的属性，其值具有原型（类实例）将自动使用`asReference()`注释。 此外，这些属性不会被进一步递归。
<hr>

#### `asFlat`

这比`asReference`略宽一些。 `asFlat`允许属性本身可观察，但不允许其任何子节点。 典型用法适用于您只想观察数组实例而不是其项目的数组。 请注意，对于数组，**length**属性仍然是可观察的，因为它在数组实例上。 但是，对子属性的任何更改都不会被观察到。

![](http://m.qpic.cn/psb?/V12JcZNk0IdDlN/Zgx*SGhcK1Qc47G3q6VQNWfvv1YJOqWVCf5D8DzU*EM!/b/dDUBAAAAAAAA&bo=HAXCAwAAAAADB*o!&rf=viewer_4)

> 首先创建`@observable`所有内容，然后应用`asReference`和`asFlat`修饰符来修剪可观察属性。

当你深入实现应用程序的各种功能时，你会发现这种修剪的好处。且当你开始时可能并不明显，这完全很正常。当你识别出不需要深度可观察性的属性时，请确保重新检查你的`Store`， 它可以对您的应用程序的性能产生积极影响。
```javascript
import {observable} from 'mobx';

class AlbumStore {
    @observable name;
    
    // 这里不需要观察
    @observable createdDate = asReference(new Date()); 
    
    @observable description;
    @observable author;
    
    // 只观察照片数组，而不是单独的照片
    @observable photos = asFlat([]); 
}
```
<hr>

### 扩展可观察属性

和修剪可观察属性相反，你可以扩展对象上可观察性的范围/行为，而不是删除可观察性。 这里有三个可以控制它的修饰符：


#### `asStructure`
这会修改将新值分配给`observable`时完成相等性检查的方式。 默认情况下，仅将引用更改视为更改。 如果您希望基于内部结构进行比较，则可以使用此修饰符。 这主要是针对值类型（也称为结构），只有在它们的值匹配时才相等。如下图：
![](http://m.qpic.cn/psb?/V12JcZNk0IdDlN/OuOPNCCZ42IRmX.Js38lSicPe6ttyZRupmRQcfmoIbQ!/b/dDYBAAAAAAAA&bo=*gPGAwAAAAADBxo!&rf=viewer_4)

```javascript
const { asStructure, observable } = require('mobx');

let address1 = {
    zip: 12345,
    city: 'New York'
};

let address2 = {
    zip: 12345,
    city: 'New York'
};

let contact = {
    address: observable(address1)
};

// 将被视为一种变化，因为它是一个新的引用
contact.address = address2;

// 使用 asStructure() 修饰
let contact2 = {
    address: observable(asStructure(address1)) 
};

// 不会被视为一种变化，因为它具有相同的价值
contact.address = address2;
```
<hr>

#### `asMap`
默认情况下，将对象标记为可观察对象时，它只能跟踪最初在对象上定义的属性。 如果添加新属性，则不会跟踪这些属性。 使用`asMap`，您甚至可以使新添加的属性可观察。 在内部，MobX将创建一个类似ES6的Map，它具有与原生Map类似的API。

除了使用此修饰符，您还可以通过从常规可观察对象开始来实现类似的效果。 然后，您可以使用`extendObservable()`API添加更多可观察的属性。 当您想要延迟添加可观察属性时，此API非常有用。
<hr>

#### `computed`
这是一个如此强大的概念，其重要性无法得到足够的重视。 计算属性不是域的真实属性，而是使用实际属性派生（也称为计算）。 一个典型的例子是person实例的fullName属性。 它派生自firstName和lastName属性。 通过创建简单的计算属性，您可以简化域逻辑。 例如，您可以只创建一个计算的hasLastName属性，而不是检查一个人是否在任何地方都有lastName
```javascript
class Person {
    @observable firstName;
    @observable lastName;

    @computed get fullName() {
        return `${this.firstName}, ${this.lastName}`;
    }

    @computed get hasLastName() {
        return !!this.lastName;
    }
}
```
![](http://m.qpic.cn/psb?/V12JcZNk0IdDlN/vpuiQ.9SYiE7R6Ay9y4WFyip5MZjJOsnW5zjLbEFtcQ!/b/dDUBAAAAAAAA&bo=GAQ2AwAAAAADBws!&rf=viewer_4)

构建可观察树是使用MobX的一个重要方面，这使MobX开始跟踪您的`store`中有趣且值得改变的部分！

[原文链接](https://blog.pixelingene.com/2016/10/effective-mobx-patterns-part-1/)

# 高效的Mobx模式 - （Part 2）

在上一部分中，我们研究了如何设置MobX状态树并使其可观察。 有了这个，下一步就是开始对变化作出反应。 坦率地说，这就是有趣的开始！

MobX保证只要您的响应数据图发生变化，依赖于可观察属性的部分就会自动同步。 这意味着您现在可以专注于对变化做出反应并引起的副作用，而不是担心数据同步。

让我们深入研究一下可以引起副作用的各种方法。

### 使用`@action`作为入口点

默认情况下，当您修改observable时，MobX将检测并保持其他依赖的可观察对象同步。 这是同步发生的。 但是，有时您可能希望在同一方法中修改多个observable。 这可能会导致多个通知被触发，甚至可能会降低您的应用速度。
![action](http://m.qpic.cn/psb?/V12JcZNk0IdDlN/m8ntGHg8MiOrkNEnRDykuLZ33eoqM*CqpQAS37zm3K8!/b/dDMBAAAAAAAA&bo=agXyA2oF8gMDCSw!&rf=viewer_4)

更好的方法是`action()`中包装要调用的方法。 这会在您的方法周围创建一个事务边界，并且所有受影响的`observable`将在您执行操作后保持同步。 请注意，此延迟通知仅适用于当前函数范围中的`observable`。 如果您具有修改更多可观察对象的异步操作，则必须将它们包装在`runInAction()`中。
```javascript
class Person {
    @observable firstName;
    @observable lastName;

    // 因为我们在@action中包装了此方法，所以只有在changeName()成功执行后，fullName才会更改
    @action changeName(first, last) {
        this.firstName = first;
        this.lastName = last;
    }

    @computed get fullName() {
        return `${this.firstName}, ${this.lastName}`;
    }
}

const p = new Person();
p.changeName('Pavan', 'Podila');

```
`Actions`是改变**Store**的切入点。 通过使用`Actions`，您可以将多个observable更新为原子操作。

> 尽可能避免直接从外部操纵`observable`并公开`@action`方法为你做这个改变。 实际上，可以通过设置`useStrict(true)`来强制执行此操作。
<hr>


### 使用`@autorun`触发副作用
MobX确保可观察图形始终保持一致。 但如果这个世界只是关于可观察的东西，那就不好玩了。 我们需要他们的同行：观察者使事情变得有趣。

实际上，UI是`mobx store`的美化观察者。 使用`mobx-react`，您将获得一个绑定库，使您的React组件可以观察存储并在存储更改时自动呈现。

但是，UI不是系统中唯一的观察者。 您可以向`store`添加更多观察者以执行各种有趣的事情。 一个非常基本的观察者可能是一个控制台记录器，它只是在可观察的变化时将当前值记录到控制台。

![](http://m.qpic.cn/psb?/V12JcZNk0IdDlN/diic7jbUYIKj7SCuB*KP1w8qsgn6mji2Hom4mK1qvUo!/b/dDEBAAAAAAAA&bo=cgIsA3ICLAMDGTw!&rf=viewer_4)

通过`autorun`，我们可以非常轻松地设置这些观察者。 最快的方法是提供`autorun`功能。 MobX会自动跟踪您在此函数中使用的任何可观察对象。 每当它们改变时，你的功能都会重新执行（也就是自动运行）！

```javascript
class Person {

    @observable firstName = 'None';
    @observable lastName = 'None';

    constructor() {

        // A simple console-logger
        autorun(()=>{
            console.log(`Name changed: ${this.firstName}, ${this.lastName}`);
        });

        // 这里会导致autorun()运行
        this.firstName = 'Mob';

        // autorun()再一次运行
        this.lastName = 'X';
    }
}

// Will log: Name changed: None, None
// Will log: Name changed: Mob, None
// Will log: Name changed: Mob, X
```

正如您在上面的日志中所看到的，自动运行将立即运行，并且每次跟踪的可观察量发生变化时也会运行。 如果您不想立即运行，而是仅在发生更改时运行，该怎么办？ 请继续阅读。
<hr>

### 首次更换后使用`reaction`触发副作用

与`autorun`相比，`reaction`提供了更细粒度的控制。 首先，它们不会立即运行并等待对跟踪的可观察量的第一次更改。 API也与`autorun`略有不同。 在最简单的版本中，您提供两个输入参数：
```javascript
reaction(()=> data, data => { /* side effect */})
```
第一个函数（跟踪函数 `tracking function`）应该返回将用于跟踪的数据。 然后将该数据传递给第二个函数（效果函数 `effect function`）。 不跟踪效果函数，您可以在此处使用其他可观察对象。

![](http://m.qpic.cn/psb?/V12JcZNk0IdDlN/9*jtUQMyx50fhnbYgxUXjHZ8Og5tcjGG.QNx9uweDX8!/b/dDEBAAAAAAAA&bo=aAMaA2gDGgMDCSw!&rf=viewer_4)

默认情况下，`reaction`将不会在第一次运行，并将等待追踪函数的变更。 只有当`tracking function`返回的数据发生变化时，才会执行副作用。 通过将原始自动运行分解为`tracking function `+`effect function`，您可以更好地控制实际导致副作用的内容。
```javascript
import {reaction} from 'mobx';

class Router {

    @observable page = 'main';

    setupNavigation() {
        reaction(()=>this.page, (page)=>{
            switch(page) {
                case 'main':
                    this.navigateToUrl('/');
                    break;

                case 'profile':
                    this.navigateToUrl('/profile');
                    break;

                case 'admin':
                    this.navigateToUrl('/admin');
                    break;
            }
        });
    }

    navigateToUrl(url) { /* ... */ }
}
```
在上面的示例中，我在加载“main”页面时不需要导航。 一个`reaction`使用的完美案例。 仅当路由器的页面属性发生更改时，才会导航到特定URL。

以上是一个非常简单的路由器，具有固定的页面集。 您可以通过向URL添加页面地图来使其更具可扩展性。 使用这种方法，路由（使用URL更改）会成为更改**Store**某些属性的副作用。

### 使用`when`触发一次性的副作用

`autorun`和`reaction`是持续的副作用。 初始化应用程序时，您将创建此类副作用，并期望它们在应用程序的生命周期内运行。

我之前没有提到的一件事是这两个函数都返回一个处理器函数。 您可以随时调用该处理器函数并取消副作用。

```javascript
const disposer = autorun(()=>{ 
    /* side-effects based on tracked observables */ 
});

// .... At a later time
disposer(); // Cancel the autorun 
```

现在我们构建的应用程序有各种用例。 您可能希望某些副作用仅在您到达应用程序中的某个点时运行。 此外，您可能希望这些副作用只运行一次，然后再也不会运行。

![](http://m.qpic.cn/psb?/V12JcZNk0IdDlN/Z1fY8CDpu7jA.NYzBvRMMdIQFcefdG**.GQ*Xb9fuEQ!/b/dDYBAAAAAAAA&bo=QAP0AkAD9AIDGTw!&rf=viewer_4)

让我们举一个具体的例子：比如说，当用户到达应用程序中的某个里程碑时，您希望向用户显示一条消息。 此里程碑仅对任何用户发生一次，因此您不希望设置持续运行的副作用，如`autorun`或`reaction`。 现在是时候拿出**`when`** 这个API来完成这项工作了。

当拿出两个参数时，就像`reaction`一样。 第一个（跟踪器函数）应该返回一个布尔值。 当这变为真时，它将运行效果函数，第二个参数为`when`。 最棒的部分是它会在运行后自动处理副作用。 因此，无需跟踪处理器并手动调用它。

```javascript
when(()=>this.reachedMilestone, ()=>{
    this.showMessage({ 
	    title: 'Congratulations', 
	    message: 'You did it!'
	});
})
```
<hr>

到目前为止，我们已经看到了各种技术来跟踪对象图上的变化，并对这些变化做出反应。 MobX提高了抽象级别，以便我们可以在更高级别进行思考，而不必担心跟踪和对变化做出反应的意外复杂性。

我们现在有了一个基础，可以构建依赖于域模型更改的强大系统。 通过将域模型之外的所有内容视为副作用，我们可以提供视觉反馈（UI）并执行许多其他活动，如监控，分析，日志记录等。

[原文链接](https://blog.pixelingene.com/2016/10/effective-mobx-patterns-part-2/)
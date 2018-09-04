# 高效的Mobx模式 - （Part 3）

前两部分侧重于MobX的基本构建块。 有了这些块，我们现在可以通过MobX的角度开始解决一些真实场景。 这篇文章将是一系列应用我们迄今为止所见概念的例子。

当然，这不是一个详尽的清单，但应该让你体会到应用MobX角度所需要的心理转变。 所有示例都是在没有@decorator （装饰器）语法的情况下创建的。 这允许您在Chrome控制台，**Node REPL**或支持临时文件的WebStorm等IDE中进行尝试。

### 改变思维方式

当您学习某些库或框架背后的理论并尝试将其应用于您自己的问题时，您最初可能会画一个空白。 它发生在像我这样的普通人身上，甚至是最好的人。 写作界称之为**“Writer’s block”**，而在艺术家的世界里，它就是**“Painter’s block”**。

我们需要的是从简单到复杂的例子来塑造我们的思维方式。 只有看到应用程序，我们才能开始想象解决我们自己的问题的方法。

对于MobX，它首先要了解您有一个`reactive object-graph`这一事实。 树的某些部分可能依赖于其他部分。 当树变异时，连接的部分将作出反应并更新以反映变化。

> 思维方式的转变是将手头的系统设想为一组反应性变动 **+** 一组相应的结果。

效果可以是由于反应性变化而产生输出的任何事物。 让我们探索各种现实世界的例子，看看我们如何用MobX建模和表达它们。
<hr>

#### `Example 1: 发送重要操作的分析`

**`问题：`**我们在应用程序中有一些必须记录到服务器的一次性操作。 我们希望跟踪执行这些操作的时间并发送分析。

**`1、`**是对建立状态模型。 我们的行为是有限的，我们只关心它执行一次。 我们可以使用动作方法的名称建立对应的布尔值类型状态， 这是我们可观察到的状态。

```javascript
const actionMap = observable({
    login: false,
    logout: false,
    forgotPassword: false,
    changePassword: false,
    loginFailed: false
});
```

**`2、`**接下来，我们必须对这些行动状态发生的变化作出反应。 因为它们只在生命周期中发生过一次，所以我们不会使用长期运行的效果，如`autorun()`或`reaction()`。 我们也不希望这些效果在执行后存在。 好吧，这给我们留下了一个选择：.... 
....
....
#### `when`
```javascript
Object.keys(actionMap)
    .forEach(key => {
        when(
            () => actionMap[key],
            () => reportAnalyticsForAction(key)
        );
    });

function reportAnalyticsForAction(actionName) {
    console.log('Reporting: ', actionName);

    /* ... JSON API Request ... */
}
```

在上面的代码中，我们只是循环遍历actionMap中的键并为每个键设置`when()`副作用。 当tracker-function（第一个参数）返回true时，副作用将运行。 运行效果函数（第二个参数）后，`when()`将自动处理。 因此，没有从应用程序发送多个报告的问题！

**`3、`**我们还需要一个MobX动作来改变可观察状态。 请记住：永远不要直接修改您的observable。 始终通过`action`来做到这一点。
对上面的例子来说，如下：
```javascript
const markActionComplete = action((name) => {
    actionMap[name] = true;
});

markActionComplete('login');
markActionComplete('logout');

markActionComplete('login');

// [LOG] Reporting:  login
// [LOG] Reporting:  logout
```
请注意，即使我将登录操作标记触发两次，也没有发送日志报告。 完美，这正是我们需要的结果。
它有两个原因：
1.  `login`标记已经为`true`，因此值没有变化
2.  此外，`when()`副作用已被触发执行，因此不再发生追踪。
<hr>


#### `Example 2: 作为工作流程的一部分启动操作`

**`问题：`**我们有一个由几个状态组成的工作流程。 每个状态都映射到某些任务，这些任务在工作流到达该状态时执行。

**`1、`**从上面的描述中可以看出，唯一可观察的值是工作流的状态。 需要为每个状态运行的任务可以存储为简单映射。 有了这个，我们可以模拟我们的工作流程：
```javascript
class Workflow {
    constructor(taskMap) {
        this.taskMap = taskMap;
        this.state = observable({
            previous: null,
            next: null
        });

        this.transitionTo = action((name) => {
            this.state.previous = this.state.next;
            this.state.next = name;
        });

        this.monitorWorkflow();
    }

    monitorWorkflow() {
        /* ... */
    }
}

// Usage
const workflow = new Workflow({
    start() {
        console.log('Running START');
    },

    process(){
        console.log('Running PROCESS');
    },

    approve() {
        console.log('Running APPROVE');
    },

    finalize(workflow) {
        console.log('Running FINALIZE');

        setTimeout(()=>{
            workflow.transitionTo('end');
        }, 500);
    },

    end() {
        console.log('Running END');
    }
});
```
请注意，我们正在存储一个名为`state`的实例变量，该变量跟踪工作流的当前和先前状态。 我们还传递`state->task`的映射，存储为`taskMap`。

**`2`、**现在有趣的部分是关于监控工作流程。 在这种情况下，我们没有像前一个例子那样的一次性操作。 工作流通常是长时间运行的，可能在应用程序的生命周期内。 这需要`autorun`或`reaction()`。

只有在转换到状态时才会执行状态任务。 因此我们需要等待对`this.state.next`进行更改才能运行任何副作用（任务）。 等待更改表示使用`reaction()`因为它仅在跟踪的可观察值更改值时才会运行。 所以我们的监控代码如下所示：
```javascript
class Workflow {
    /* ... */
    monitorWorkflow() {
        reaction(
            () => this.state.next,
            (nextState) => {
                const task = this.taskMap[nextState];
                if (task) {
                    task(this);
                }
            }
        )
    }
}
```

`reaction()`第一个参数是跟踪函数，在这种情况下只返回`this.state.next`。 当跟踪功能的返回值改变时，它将触发效果功能。 效果函数查看当前状态，从`this.taskMap`查找任务并简单地调用它。

请注意，我们还将工作流的实例传递给任务。 这可用于将工作流转换为其他状态。
```javascript
workflow.transitionTo('start');

workflow.transitionTo('finalize');

// [LOG] Running START
// [LOG] Running FINALIZE
/* ... after 500ms ... */
// [LOG] Running END
```
有趣的是，这种存储一个简单的`observable`的技术，比如`this.state.next`和使用`reaction()`来触发副作用，也可以用于：

1. 通过`react-router`进行路由
2. 在演示应用程序中导航
3. 基于模式在不同视图之间切换
<hr>


#### `Example 3: 输入更改时执行表单验证`

**`问题：`**这是一个经典的Web表单用例，您需要验证一堆输入。 如果有效，允许提交表单。

**`1、`**让我们用一个简单的表单数据类对其进行建模，其字段必须经过验证。
```javascript
class FormData {
    constructor() {
        extendObservable(this, {
            firstName: '',
            lastName: '',
            email: '',
            acceptTerms: false,

            errors: {},

            get valid() { // this becomes a computed() property
                return (this.errors === null);
            }
        });

        this.setupValidation(); // We will look at this below
    }
}
```
`extendObservable()`API是我们以前从未见过的。 通过在我们的类实例（this）上应用它，我们得到一个ES5相当于创建一个@observable类属性。
```javascript
class FormData {
    @observable firstName = '';
    /* ... */
}
```

**`2、`**接下来，我们需要监视这些字段何时发生变化并运行一些验证逻辑。 如果验证通过，我们可以将实体标记为有效并允许提交。 使用计算属性跟踪有效性本身：有效。

由于验证逻辑需要在FormData的生命周期内运行，因此我们将使用`autorun()`。 我们也可以使用`reaction()`但我们想立即运行验证而不是等待第一次更改。
```javascript
class FormData {
    setupValidation() {
        autorun(() => {
            // Dereferencing observables for tracking
            const {firstName, lastName, email, acceptTerms} = this;
            const props = {
                firstName,
                lastName,
                email,
                acceptTerms
            };

            this.runValidation(props, {/* ... */})
                .then(result => {
                    this.errors = result;
                })
        });
    }

    runValidation(propertyMap, rules) {
        return new Promise((resolve) => {
            const {firstName, lastName, email, acceptTerms} = propertyMap;

            const isValid = (firstName !== '' && lastName !== '' && email !== '' && acceptTerms === true);
            resolve(isValid ? null : {/* ... map of errors ... */});
        });
    }

}
```
在上面的代码中，`autorun()`将在跟踪的`observables`发生更改时自动触发。 请注意，要使MobX正确跟踪您的observable，您必须使用解除引用。

`runValidation()`是一个异步调用，这就是我们返回一个promise的原因。 在上面的示例中，它并不重要，但在现实世界中，您可能会调用服务器进行一些特殊验证。 当结果返回时，我们将设置错误`observable`，这将反过来更新有效的计算属性。

如果你有一个耗时较大的验证逻辑，你甚至可以使用`autorunAsync()`，它有一个参数可以延迟执行去抖动。

**`2、`**好吧，让我们的代码付诸行动。 我们将设置一个简单的控制台记录器（通过`autorun()`）并跟踪有效的计算属性。
```javascript
const instance = new FormData();

// Simple console logger
autorun(() => {
    // input的每一次输入，结果都会触发error变更，autorun随即执行
    const validation = instance.errors;

    console.log(`Valid = ${instance.valid}`);
    if (instance.valid) {
        console.log('--- Form Submitted ---');
    }

});

// Let's change the fields
instance.firstName = 'Pavan';
instance.lastName = 'Podila';
instance.email = 'pavan@pixelingene.com';
instance.acceptTerms = true;

// 	输出日志如下
// 	Valid = false
//	Valid = false
//	Valid = false
//	Valid = false
//	Valid = false
//	Valid = true
//	--- Form Submitted ---
```
由于`autonrun()`立即运行，您将在开头看到两个额外的日志，一个用于`instance.errors`，一个用于`instance.valid`，第1-2行。 其余四行（3-6）用于现场的每次更改。


每个字段更改都会触发`runValidation()`，每次都会在内部返回一个新的错误对象。 这会导致`instance.errors`的引用发生更改，然后触发我们的`autorun()`以记录有效标志。 最后，当我们设置了所有字段时，`instance.errors`变为null（再次更改引用）并记录最终的“Valid = true”。

**`4、`**简而言之，我们通过使表单字段可观察来进行表单验证。 我们还添加了额外的errors属性和有效的计算属性来跟踪有效性。 `autorun()`通过将所有内容捆绑在一起来节省时间。
<hr>

#### `Example 4: 跟踪所有已注册的组件是否已加载`

**`问题：`** 我们有一组已注册的组件，我们希望在所有组件都加载后跟踪。 每个组件都将公开一个返回 promise的`load()`方法。 如果promise解析，我们将组件标记为已加载。 如果它拒绝，我们将其标记为失败。 当所有这些都完成加载时，我们将报告整个集是否已加载或失败。

**`1、`**我们先来看看我们正在处理的组件。 我们正在创建一组随机报告其负载状态的组件。 另请注意，有些是异步的。
```javascript
const components = [
    {
        name: 'first',
        load() {
            return new Promise((resolve, reject) => {
                Math.random() > 0.5 ? resolve(true) : reject(false);
            });
        }
    },
    {
        name: 'second',
        load() {
            return new Promise((resolve, reject) => {
                setTimeout(() => {
                    Math.random() > 0.5 ? resolve(true) : reject(false);
                }, 1000);
            });
        }
    },
    {
        name: 'third',
        load() {
            return new Promise((resolve, reject) => {
                setTimeout(() => {
                    Math.random() > 0.25 ? resolve(true) : reject(false);
                }, 500);
            });
        }
    },
];
```

**`2、`**下一步是为`Tracker`设计可观察状态。 组件的`load()`不会按特定顺序完成。 所以我们需要一个可观察的数组来存储每个组件的加载状态。 我们还将跟踪每个组件的报告状态。

当所有组件都已报告时，我们可以通知组件集的最终加载状态。 以下代码设置了可观察量。
```javascript
class Tracker {
    constructor(components) {
        this.components = components;

        extendObservable(this, {

            // Create an observable array of state objects,
            // one per component
            states: components.map(({name}) => {
                return {
                    name,
                    reported: false,
                    loaded: undefined
                };
            }),

            // computed property that derives if all components have reported
            get reported() {
                return this.states.reduce((flag, state) => {
                    return flag && state.reported;
                }, true);
            },

            // computed property that derives the final loaded state 
            // of all components
            get loaded() {
                return this.states.reduce((flag, state) => {
                    return flag && !!state.loaded;
                }, true);
            },

            // An action method to mark reported + loaded
            mark: action((name, loaded) => {
                const state = this.states.find(state => state.name === name);

                state.reported = true;
                state.loaded = loaded;
            })

        });

    }
}
```

我们回到使用`extendObservable()`来设置我们的可观察状态。 `reported`和`load`的计算属性跟踪组件完成其加载的时间。 `mark()`是我们改变可观察状态的动作方法。

顺便说一句，建议在需要从您的`observables`派生值的任何地方使用`computed`。 将其视为产生价值的可观察物。 计算值也会被缓存，从而提高性能。 另一方面，`autorun`和`reaction`不会产生价值。 相反，它们提供了创建副作用的命令层。

**`3、`**为了启动跟踪，我们将在`Tracker`上创建一个`track()`方法。 这将触发每个组件的`load()`并等待返回的Promise解析/拒绝。 基于此，它将标记组件的负载状态。

`when()`所有组件都已`reported`时，跟踪器可以报告最终加载的状态。 我们在这里使用，因为我们正在等待条件变为真（this.reported）。 报告的副作用只需要发生一次，非常适合`when()`。

以下代码负责以上事项：
```javascript
class Tracker {
    /* ... */ 
    track(done) {
        when(
            () => this.reported,
            () => {
                done(this.loaded);
            }
        );

        this.components.forEach(({name, load}) => {
            load()
                .then(() => {
                    this.mark(name, true);
                })
                .catch(() => {
                    this.mark(name, false);
                });
        });
    }

    setupLogger() {
        autorun(() => {
            const loaded = this.states.map(({name, loaded}) => {
                return `${name}: ${loaded}`;
            });

            console.log(loaded.join(', '));
        });
    }
}
```

`setupLogger()`实际上不是解决方案的一部分，但用于记录报告。 这是了解我们的解决方案是否有效的好方法。

**`4、`**现在我们来测试一下：
```javascript
const t = new Tracker(components);
t.setupLogger();
t.track((loaded) => {
    console.log('All Components Loaded = ', loaded);
});

// first: undefined, second: undefined, third: undefined
// first: true, second: undefined, third: undefined
// first: true, second: undefined, third: true
// All Components Loaded =  false
// first: true, second: false, third: true
```
记录的输出显示其按预期工作。 在组件报告时，我们记录每个组件的当前加载状态。 当所有人报告时，this.reported变为true，我们看到“All Components Loaded”消息。

希望上面的一些例子让你体会到在MobX中的思考。

1. 设计可观察状态
2. 设置变异动作方法以更改可观察状态
3. 放入跟踪功能（when，autorun，reaction）以响应可观察状态的变化

上述公式应该适用于需要在发生变化后跟踪某些内容的复杂场景，这可能导致重复1-3步骤。

![](http://m.qpic.cn/psb?/V12JcZNk0IdDlN/W21MQ949ZBd0HWL*cPqGIAYzLaE8xzPPWfiIwz7sxXU!/b/dDcBAAAAAAAA&bo=.AWsAwAAAAADB3A!&rf=viewer_4)

[原文链接](https://blog.pixelingene.com/2016/10/effective-mobx-patterns-part-3/)
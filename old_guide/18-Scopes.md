@ngdoc overview
@name Developer Guide: Scopes
@description

# 何为Scopes

{@link api/ng.$rootScope.Scope scope} 
- 是一个存储应用数据模型的对象
- 为 {@link expression expressions} 提供了一个执行上下文
- Scopes 的层级结构对应于 DOM 树结构
- Scopes 可以监听 {@link guide/expression expressions} 的变化并传播事件

## Scopes有什么

  - Scopes 提供了 ({@link api/ng.$rootScope.Scope#methods_$watch $watch}) 方法监听数据模型的变化

  - Scopes 提供了 ({@link api/ng.$rootScope.Scope#methods_$apply $apply}) 方法把不是由 ng 触发的数据模型的改变引入 ng 的控制范围内（如控制器，服务，及 ng 事件处理器等）
  
  - Scopes 提供了基于原型式继承其父 scope 属性的机制，就算是嵌套于独立的应用组件中的 scopes 也是可以访问共享的数据模型（这个涉及到指令间嵌套时 scope 的几种模式）

  - Scopes 提供了 {@link guide/expression expressions} 的执行环境，比如像 `{{username}}` 这个表达式，必须得是在一个拥有属性这个属性的 scope 中执行才会有意义，也就是说，scope 中可能会像这样 `scope.username` 或是 `$scope.username`，至于有没有 $ 符号，看你是在哪里调用 scope 了

## Scope作为数据模型使用

Scope 是应用（注：一般指web应用）的控制器和视图之间的粘结剂。在 ng 中，最直观的表现是：在自定义指令中，处在模版的 {@link compiler
linking} 阶段时， {@link api/ng.$compileProvider#methods_directive directives} （指令）会设置一个 {@link api/ng.$rootScope.Scope#methods_$watch `$watch`} 函数监听着 scope 中的各表达式（注：这个过程是隐性的）。这个 `$watch` 允许指令在 scope 中的属性变化时收到通知，进而指令能够根据这个改变来对DOM进行重新渲染，以便更新已改变的属性值（注：属性值就是scope对象中的属性，也就是数据模型）。

其实，不止上面所说的指令拥有指向 scope 的引用，控制器中也有（注：可以理解为控制器与指令均能引用到与它们相对应的DOM结构所处的 scope）。但是控制器与指令是相互分离的，而且它们与视图之间也是分离的，这样的分离，或者说耦合度低，可以大大提高对应用进行测试的这一块工作的效率。

注：其实可以很简单地理解为有以下两个链条关系：

- 控制器 --> scope --> 视图（DOM）
- 指令 --> scope --> 视图（DOM）

让我们来看下面一个例子，可以说明 scope 作为视图与控制器的黏合剂：

<example>
  <file name="script.js">
    function MyController($scope) {
      $scope.username = 'World';

      $scope.sayHello = function() {
        $scope.greeting = 'Hello ' + $scope.username + '!';
      };
    }
  </file>
  <file name="index.html">
    <div ng-controller="MyController">
      Your name:
        <input type="text" ng-model="username">
        <button ng-click='sayHello()'>greet</button>
      <hr>
      {{greeting}}
    </div>
  </file>
</example>

在上面这个例子中，我们有：

- 控制器：`MyController`，它引用了 `$scope` 并在其上注册了两个属性和一个方法
- $scope 对象：持有上面例子所需的数据模型，包括 `username` 属性、`greeting`属性（注：这是在`sayHello()`方法被调用时注册的）和 `sayHello()` 方法
- 视图：拥有一个输入框、一个按钮以及一个利用双向绑定来显示数据的内容块

那么具体整个示例有这样两个流程，**从控制器发起的角度**来看就是：

1. 控制器往 scope 中写属性：
    - 给 scope 中的 `username` 赋值，然后 scope 通知视图中的 `input` 数据变化了，`input` 因为通过 `ng-model` 实现了双向绑定可以知道 `username` 的变化，进而在视图中渲染出改变的值，这里是 `World`
2. 控制器往 scope 中写方法
    - 给 scope 中的 `sayHello()` 方法赋值，该方法接受视图中的 `button` 调用，因为 `button` 通过 `ng-click` 绑定了该方法，当用户点击按钮时，`sayHello()` 被调用，这个方法读取 scope 中的 `username` 属性，加上前缀字符串 `Hello`，然后赋值给在 scope 中新创建的 `greeting` 属性

整个示例的过程如果**从视图的角度看**，那主要是以下三个部分：

1. `input` 中的渲染逻辑：展示了通过 `ng-model` 进行的 scope 和 视图中某表单元素的双向绑定
    - 根据 `ng-model` 中的 `username` 去 scope 中取，如果已经有赋值，那么用这个默认值填充当前的输入框
    - 接受用户输入，并且将用户输入的字符串传给 `username`，这时候 scope 中的该属性值实时对应用户输入的值

2. `button` 中的逻辑
    - 接受用户单击，调用 scope 中的 `sayHello()` 方法

3. `{{greeting}}` 的渲染逻辑
    - 在用户未单击按钮时，不显示内容
    - *取值阶段*：在用户单击后，这个表达式会去scope中取 `greeting` 属性，而这个 scope 和控制器是同一个的（这个例子中），这时候，该 scope 下 `greeting` 属性已经有了，这时候这个属性就被取回来了
    - *计算阶段*：在当前 scope 下去计算 `greeting` {@link guide/expression expression} ，然后渲染视图，显示 `Hello World`

经过以上的两种角度分析示例过程，我们可以知道：**scope 对象以及其属性是视图渲染的唯一数据来源。

从测试的角度来看，视图与控制器分离的需求在于它允许测试人员可以单独对应用的操作逻辑进行测试，而不必考虑页面的渲染细节。

<pre>
  it('should say hello', function() {
    var scopeMock = {};
    var cntl = new MyController(scopeMock);

    // Assert that username is pre-filled
    expect(scopeMock.username).toEqual('World');

    // Assert that we read new username and greet
    scopeMock.username = 'angular';
    scopeMock.sayHello();
    expect(scopeMock.greeting).toEqual('Hello angular!');
  });
</pre>


## Scope分层结构

Each Angular application has exactly one {@link api/ng.$rootScope root scope}, but
may have several child scopes.

The application can have multiple scopes, because some {@link guide/directive directives} create
new child scopes (refer to directive documentation to see which directives create new scopes).
When new scopes are created, they are added as children of their parent scope. This creates a tree
structure which parallels the DOM where they're attached

When Angular evaluates `{{name}}`, it first looks at the scope associated with the given
element for the `name` property. If no such property is found, it searches the parent scope
and so on until the root scope is reached. In JavaScript this behavior is known as prototypical
inheritance, and child scopes prototypically inherit from their parents.

This example illustrates scopes in application, and prototypical inheritance of properties. The example is followed by
a diagram depicting the scope boundaries.

<example>
  <file name="index.html">
  <div class="show-scope-demo">
    <div ng-controller="GreetCtrl">
      Hello {{name}}!
    </div>
    <div ng-controller="ListCtrl">
      <ol>
        <li ng-repeat="name in names">{{name}} from {{department}}</li>
      </ol>
    </div>
  </div>
  </file>
  <file name="script.js">
    function GreetCtrl($scope, $rootScope) {
      $scope.name = 'World';
      $rootScope.department = 'Angular';
    }

    function ListCtrl($scope) {
      $scope.names = ['Igor', 'Misko', 'Vojta'];
    }
  </file>
  <file name="style.css">
    .show-scope-demo.ng-scope,
    .show-scope-demo .ng-scope  {
      border: 1px solid red;
      margin: 3px;
    }
  </file>
</example>

<img class="center" src="img/guide/concepts-scope.png">

Notice that Angular automatically places `ng-scope` class on elements where scopes are
attached. The `<style>` definition in this example highlights in red the new scope locations. The
child scopes are necessary because the repeater evaluates `{{name}}` expression, but
depending on which scope the expression is evaluated it produces different result. Similarly the
evaluation of `{{department}}` prototypically inherits from root scope, as it is the only place
where the `department` property is defined.


## 从DOM中抓取Scopes

Scopes are attached to the DOM as `$scope` data property, and can be retrieved for debugging
purposes. (It is unlikely that one would need to retrieve scopes in this way inside the
application.) The location where the root scope is attached to the DOM is defined by the location
of {@link api/ng.directive:ngApp `ng-app`} directive. Typically
`ng-app` is placed on the `<html>` element, but it can be placed on other elements as well, if,
for example, only a portion of the view needs to be controlled by Angular.

To examine the scope in the debugger:

  1. right click on the element of interest in your browser and select 'inspect element'. You
  should see the browser debugger with the element you clicked on highlighted.

  2. The debugger allows you to access the currently selected element in the console as `$0`
    variable.

  3. To retrieve the associated scope in console execute: `angular.element($0).scope()` or just type $scope


## 基于Scope的事件传播

Scopes can propagate events in similar fashion to DOM events. The event can be {@link
api/ng.$rootScope.Scope#methods_$broadcast broadcasted} to the scope children or {@link
api/ng.$rootScope.Scope#methods_$emit emitted} to scope parents.

<example>
  <file name="script.js">
    function EventController($scope) {
      $scope.count = 0;
      $scope.$on('MyEvent', function() {
        $scope.count++;
      });
    }
  </file>
  <file name="index.html">
    <div ng-controller="EventController">
      Root scope <tt>MyEvent</tt> count: {{count}}
      <ul>
        <li ng-repeat="i in [1]" ng-controller="EventController">
          <button ng-click="$emit('MyEvent')">$emit('MyEvent')</button>
          <button ng-click="$broadcast('MyEvent')">$broadcast('MyEvent')</button>
          <br>
          Middle scope <tt>MyEvent</tt> count: {{count}}
          <ul>
            <li ng-repeat="item in [1, 2]" ng-controller="EventController">
              Leaf scope <tt>MyEvent</tt> count: {{count}}
            </li>
          </ul>
        </li>
      </ul>
    </div>
  </file>
</example>



## Scope轮循（生命周期）

The normal flow of a browser receiving an event is that it executes a corresponding JavaScript
callback. Once the callback completes the browser re-renders the DOM and returns to waiting for
more events.

When the browser calls into JavaScript the code executes outside the Angular execution context,
which means that Angular is unaware of model modifications. To properly process model
modifications the execution has to enter the Angular execution context using the {@link
api/ng.$rootScope.Scope#methods_$apply `$apply`} method. Only model modifications which
execute inside the `$apply` method will be properly accounted for by Angular. For example if a
directive listens on DOM events, such as {@link
api/ng.directive:ngClick `ng-click`} it must evaluate the
expression inside the `$apply` method.

After evaluating the expression, the `$apply` method performs a {@link
api/ng.$rootScope.Scope#methods_$digest `$digest`}. In the $digest phase the scope examines all
of the `$watch` expressions and compares them with the previous value. This dirty checking is done
asynchronously. This means that assignment such as `$scope.username="angular"` will not
immediately cause a `$watch` to be notified, instead the `$watch` notification is delayed until
the `$digest` phase. This delay is desirable, since it coalesces multiple model updates into one
`$watch` notification as well as it guarantees that during the `$watch` notification no other
`$watch`es are running. If a `$watch` changes the value of the model, it will force additional
`$digest` cycle.

  1. **Creation**

     The {@link api/ng.$rootScope root scope} is created during the application
     bootstrap by the {@link api/AUTO.$injector $injector}. During template
     linking, some directives create new child scopes.

  2. **Watcher registration**

     During template linking directives register {@link
     api/ng.$rootScope.Scope#methods_$watch watches} on the scope. These watches will be
     used to propagate model values to the DOM.

  3. **Model mutation**

     For mutations to be properly observed, you should make them only within the {@link
     api/ng.$rootScope.Scope#methods_$apply scope.$apply()}. (Angular APIs do this
     implicitly, so no extra `$apply` call is needed when doing synchronous work in controllers,
     or asynchronous work with {@link api/ng.$http $http} or {@link
     api/ng.$timeout $timeout} services.

  4. **Mutation observation**

     At the end `$apply`, Angular performs a {@link api/ng.$rootScope.Scope#methods_$digest
     $digest} cycle on the root scope, which then propagates throughout all child scopes. During
     the `$digest` cycle, all `$watch`ed expressions or functions are checked for model mutation
     and if a mutation is detected, the `$watch` listener is called.

  5. **Scope destruction**

     When child scopes are no longer needed, it is the responsibility of the child scope creator
     to destroy them via {@link api/ng.$rootScope.Scope#methods_$destroy scope.$destroy()}
     API. This will stop propagation of `$digest` calls into the child scope and allow for memory
     used by the child scope models to be reclaimed by the garbage collector.


### Scopes和指令

During the compilation phase, the {@link compiler compiler} matches {@link
api/ng.$compileProvider#methods_directive directives} against the DOM template. The directives
usually fall into one of two categories:

  - Observing {@link api/ng.$compileProvider#methods_directive directives}, such as
    double-curly expressions `{{expression}}`, register listeners using the {@link
    api/ng.$rootScope.Scope#methods_$watch $watch()} method. This type of directive needs
    to be notified whenever the expression changes so that it can update the view.

  - Listener directives, such as {@link api/ng.directive:ngClick
    ng-click}, register a listener with the DOM. When the DOM listener fires, the directive
    executes the associated expression and updates the view using the {@link
    api/ng.$rootScope.Scope#methods_$apply $apply()} method.

When an external event (such as a user action, timer or XHR) is received, the associated {@link
expression expression} must be applied to the scope through the {@link
api/ng.$rootScope.Scope#methods_$apply $apply()} method so that all listeners are updated
correctly.

### 可以创建Scopes的指令

In most cases, {@link api/ng.$compileProvider#methods_directive directives} and scopes interact
but do not create new instances of scope. However, some directives, such as {@link
api/ng.directive:ngController ng-controller} and {@link
api/ng.directive:ngRepeat ng-repeat}, create new child scopes
and attach the child scope to the corresponding DOM element. You can retrieve a scope for any DOM
element by using an `angular.element(aDomElement).scope()` method call.

### Scopes与控制器

Scopes and controllers interact with each other in the following situations:

   - Controllers use scopes to expose controller methods to templates (see {@link
     api/ng.directive:ngController ng-controller}).

   - Controllers define methods (behavior) that can mutate the model (properties on the scope).

   - Controllers may register {@link api/ng.$rootScope.Scope#methods_$watch watches} on
     the model. These watches execute immediately after the controller behavior executes.

See the {@link api/ng.directive:ngController ng-controller} for more
information.


### Scope `$watch` 性能

Dirty checking the scope for property changes is a common operation in Angular and for this reason
the dirty checking function must be efficient. Care should be taken that the dirty checking
function does not do any DOM access, as DOM access is orders of magnitude slower then property
access on JavaScript object.

## 与浏览器事件轮循整合
<img class="pull-right" style="padding-left: 3em; padding-bottom: 1em;" src="img/guide/concepts-runtime.png">

The diagram and the example below describe how Angular interacts with the browser's event loop.

  1. The browser's event-loop waits for an event to arrive. An event is a user interaction, timer event,
     or network event (response from a server).
  2. The event's callback gets executed. This enters the JavaScript context. The callback can
      modify the DOM structure.
  3. Once the callback executes, the browser leaves the JavaScript context and
     re-renders the view based on DOM changes.

Angular modifies the normal JavaScript flow by providing its own event processing loop. This
splits the JavaScript into classical and Angular execution context. Only operations which are
applied in Angular execution context will benefit from Angular data-binding, exception handling,
property watching, etc... You can also use $apply() to enter Angular execution context from JavaScript. Keep in
mind that in most places (controllers, services) $apply has already been called for you by the
directive which is handling the event. An explicit call to $apply is needed only when
implementing custom event callbacks, or when working with third-party library callbacks.

  1. Enter Angular execution context by calling {@link guide/scope scope}`.`{@link
     api/ng.$rootScope.Scope#methods_$apply $apply}`(stimulusFn)`. Where `stimulusFn` is
     the work you wish to do in Angular execution context.
  2. Angular executes the `stimulusFn()`, which typically modifies application state.
  3. Angular enters the {@link api/ng.$rootScope.Scope#methods_$digest $digest} loop. The
     loop is made up of two smaller loops which process {@link
     api/ng.$rootScope.Scope#methods_$evalAsync $evalAsync} queue and the {@link
     api/ng.$rootScope.Scope#methods_$watch $watch} list. The {@link
     api/ng.$rootScope.Scope#methods_$digest $digest} loop keeps iterating until the model
     stabilizes, which means that the {@link api/ng.$rootScope.Scope#methods_$evalAsync
     $evalAsync} queue is empty and the {@link api/ng.$rootScope.Scope#methods_$watch
     $watch} list does not detect any changes.
  4. The {@link api/ng.$rootScope.Scope#methods_$evalAsync $evalAsync} queue is used to
     schedule work which needs to occur outside of current stack frame, but before the browser's
     view render. This is usually done with `setTimeout(0)`, but the `setTimeout(0)` approach
     suffers from slowness and may cause view flickering since the browser renders the view after
     each event.
  5. The {@link api/ng.$rootScope.Scope#methods_$watch $watch} list is a set of expressions
     which may have changed since last iteration. If a change is detected then the `$watch`
     function is called which typically updates the DOM with the new value.
  6. Once the Angular {@link api/ng.$rootScope.Scope#methods_$digest $digest} loop finishes
     the execution leaves the Angular and JavaScript context. This is followed by the browser
     re-rendering the DOM to reflect any changes.


Here is the explanation of how the `Hello world` example achieves the data-binding effect when the
user enters text into the text field.

  1. During the compilation phase:
     1. the {@link api/ng.directive:ngModel ng-model} and {@link
        api/ng.directive:input input} {@link guide/directive
        directive} set up a `keydown` listener on the `<input>` control.
     2. the {@link api/ng.$interpolate &#123;&#123;name&#125;&#125; } interpolation
        sets up a {@link api/ng.$rootScope.Scope#methods_$watch $watch} to be notified of
        `name` changes.
  2. During the runtime phase:
     1. Pressing an '`X`' key causes the browser to emit a `keydown` event on the input control.
     2. The {@link api/ng.directive:input input} directive
        captures the change to the input's value and calls {@link
        api/ng.$rootScope.Scope#methods_$apply $apply}`("name = 'X';")` to update the
        application model inside the Angular execution context.
     3. Angular applies the `name = 'X';` to the model.
     4. The {@link api/ng.$rootScope.Scope#methods_$digest $digest} loop begins
     5. The {@link api/ng.$rootScope.Scope#methods_$watch $watch} list detects a change
        on the `name` property and notifies the {@link api/ng.$interpolate
        &#123;&#123;name&#125;&#125; } interpolation, which in turn updates the DOM.
     6. Angular exits the execution context, which in turn exits the `keydown` event and with it
        the JavaScript execution context.
     7. The browser re-renders the view with update text.
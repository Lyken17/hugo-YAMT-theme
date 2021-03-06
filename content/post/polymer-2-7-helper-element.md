+++
date = "2017-07-02T22:56:11+08:00"
title = "Polymer 2.0 文档笔记(7) Helper Element"
tags = ["Polymer"]
categories = ["Web"]
+++


Polymer提供一系列的自定义元素来简化一些共有的数据绑定逻辑：

- `dom-repeat` 遍历显示数组
- `array-selector` 数组选择器
- `dom-if` 条件显示
- `dom-bind` 自动绑定


>**2.0 tip.** The data binding helper elements are bundled in to the backward-compatible,
`polymer.html` import. If you aren't using the legacy import, you'll need to import the
helper elements you're using.

为了向前兼容，`polymer.html`引入了所有的helper元素，而2.0的`polymer.Element`则要按照需要一个个手动引入。



## Template repeater (dom-repeat)

`dom-repeat`需要绑定一个数组，遍历显示里面元素，并为每个数组元素创建一个新的data scope，包括下面两个属性：

*   `item` 数组元素
*   `index` 元素下标

有两种用法:

-   在Polymer element template内部，可以直接使用简写
    ```html
      <template is="dom-repeat" items="{{items}}">
        ...
      </template>
    ```
*   在Polymer element template外部，使用`<dom-repeat>`标签
    ```html
        <dom-repeat>
          <template>
            ...
          </template>
        </dom-repeat>
    ```
    在这种情况下，还需要手动使用js给`<dom-repeat>`标签设置数据：
    ```js
        var repeater = document.querySelector('dom-repeat');
        repeater.items = someArray;
    ```

```html
<link rel="import" href="components/polymer/polymer-element.html">
<! -- import template repeater -->
<link rel="import" href="components/polymer/lib/elements/dom-repeat.html">

<dom-module id="x-custom">
  <template>
    <div> Employee list: </div>
    <template is="dom-repeat" items="{{employees}}">
        <div># [[index]]</div>
        <div>First name: <span>[[item.first]]</span></div>
        <div>Last name: <span>[[item.last]]</span></div>
    </template>
  </template>

  <script>
    class XCustom extends Polymer.Element {

      static get is() { return 'x-custom'; }

      static get properties() {
        return {
          employees: {
            type: Array,
            value() {
              return [
                {first: 'Bob', last: 'Smith'},
                {first: 'Sally', last: 'Johnson'},
              ];
            }
          }
        }
      }

    }

    customElements.define(XCustom.is, XCustom);
  </script>

</dom-module>
```
需要使用可被监听的手段去更改dom-repeat绑定的数组

```js
// Use Polymer array mutation methods:
this.push('employees', {first: 'Diana', last: 'Villiers'});

// Use Polymer set method:
this.set('employees.2.last', 'Maturin');

// Use native methods followed by notifyPath
this.employees.push({first: 'Barret', last: 'Bonden'});
this.notifyPath('employees');
```


### Handling events in `dom-repeat` templates {#handling-events}

>When handling events generated by a `dom-repeat` template instance, you
frequently want to map the element firing the event to the model data that
generated that item.

>When you add a declarative event handler **inside** the `<dom-repeat>` template,
the repeater adds a `model` property to each event sent to the listener. The `model`
object contains the scope data used to generate the template instance, so the item
data is `model.item`:

当你帮定义一个事件到dom-repeat的内部元素之后，事件参数e会有一个`model`项，代表着当前元素的data scope.


```html
<link rel="import" href="polymer/polymer-element.html">
<link rel="import" href="polymer/lib/elements/dom-repeat.html">

<dom-module id="x-custom">

  <template>
    <template is="dom-repeat" id="menu" items="{{menuItems}}">
        <div>
          <span>{{item.name}}</span>
          <span>{{item.ordered}}</span>
          <button on-click="order">Order</button>
        </div>
    </template>
  </template>

  <script>
    class XCustom extends Polymer.Element {

      static get is() { return 'x-custom'; }

      static get properties() {
        return {
          menuItems: {
            type: Array,
            value() {
              return [
                {name: 'Pizza', ordered: 0},
                {name: 'Pasta', ordered: 0},
                {name: 'Toast', ordered: 0}
              ];
            }
          }
        }
      }

      order(e) {
        e.model.set('item.ordered', e.model.item.ordered+1);
      }
    }

    customElements.define(XCustom.is, XCustom);
  </script>

</dom-module>
```

>The `model` is an instance of `TemplateInstance`, which provides the Polymer
data APIs: `get`, `set`, `setProperties`, `notifyPath` and the array manipulation methods.
You can use these to manipulate the model, using paths _relative to template instance._

`model`也是一个`TemplateInstance`的子类，提供了`get`,`set`,`setProperties`,`notifyPath`等data API

只有在dom-repeat里面绑定过的属性才会赋给`model`，如下面例子，将productId绑定到不可见的自定义属性上，以便将productId添加到`model`对象中。


```html
  <template is="dom-repeat" items="{{products}}" as="product">
    <div product-id="[[product.productId]]">[[product.name]]</div>
  </template>
```

#### Handling events outside the `dom-repeat` template.

>The `model` property is **not** added for event listeners registered imperatively (using `addEventListener`), or listeners added to one of the `dom-repeat` template's parent nodes. In these cases, you can use the `dom-repeat` `modelForElement` method to retrieve the model data that generated a given element. (There are also corresponding `itemForElement` and `indexForElement` methods.)
外部如果使用标准的DOM API`addEventListener`来监听子元素的事件时，则事件参数里面没有`e.model`属性，可以使用下面几个函数手动获得：

- `dom-repeat.modelForElement`
- `dom-repeat.itemForElement`
- `dom-repeat.indexForElement`


### Filtering and sorting lists

>To filter or sort the _displayed_ items in your list, specify a `filter` or
`sort` property on the `dom-repeat` (or both):
可以在dom-repeat上指定`filter`或`sort`的方法。


默认，`filter`和`sort`方法只在两种情况下被调用：
1. 数组发生可被监控的变化(observable change)
2. 两者方法被动态重现、改变

使用`render`方法强制`filter`和`sort`方法重新执行。（见Forcing synchronous renders）


>To re-run the `filter` or `sort` functions when certain sub-fields of `items` change, set the `observe` property to a space-separated list of `item` sub-fields that should cause the list to be re-filtered or re-sorted.

如果`filter/sort`是依据数组元素的某一个子属性来排序的，需要在dom-repeat标签上设置一个observe属性，将过滤或排序依据的子属性按照空格连接起来的字符设为它的值。
比如，下面这个例子，设置一个叫`isEngineer`的`filter`：

```js
isEngineer: function(item) {
    return item.type == 'engineer' || item.manager.type == 'engineer';
}
```

在dom-repeat标签上设置过滤器所使用过的元素子属性`type manager.type`

```html
<template is="dom-repeat" items="{{employees}}"
    filter="isEngineer" observe="type manager.type">
```


修改第0个元素中的`manager.type`将会导致整个列表重新过滤
```js
this.set('employees.0.manager.type', 'engineer');
```


#### Dynamic sort and filter changes
`observe`属性并不能完全解决所有需求，也许`filter/sort`函数需要用到其他地方的变量，因此我们可以实现一个computed binding来动态返回一个`filter/sort`函数

```html
<dom-module id="x-custom">

  <template>
    <input value="{{searchString::input}}">

    <!-- computeFilter returns a new filter function whenever searchString changes -->
    <template is="dom-repeat" items="{{employees}}" as="employee"
        filter="{{computeFilter(searchString)}}">
        <div>{{employee.lastname}}, {{employee.firstname}}</div>
    </template>
  </template>

  <script>
    class XCustom extends Polymer.Element {

      static get is() { return 'x-custom'; }

      static get properties() {
        return {
          employees: {
            type: Array,
            value() {
              return [
                { firstname: "Jack", lastname: "Aubrey" },
                { firstname: "Anne", lastname: "Elliot" },
                { firstname: "Stephen", lastname: "Maturin" },
                { firstname: "Emma", lastname: "Woodhouse" }
              ]
            }
          }
        }
      }

      computeFilter(string) {
        if (!string) {
          // set filter to null to disable filtering
          return null;
        } else {
          // return a filter function for the current search string
          string = string.toLowerCase();
          return function(employee) {
            var first = employee.firstname.toLowerCase();
            var last = employee.lastname.toLowerCase();
            return (first.indexOf(string) != -1 ||
                last.indexOf(string) != -1);
          };
        }
      }
    }

    customElements.define(XCustom.is, XCustom);
  </script>
</dom-module>
```


#### Filtering on array index 

>Because of the way Polymer tracks arrays internally, the array index isn't passed to the filter function. Looking up the array index for an item is an O(n) operation. Doing so  in a filter function could have **significant performance impact**.

不能获得数组索引，只能通过`this.items`获得原来数组，再通过`indexOf`方法获得索引，效率低下。
注意，`filter/sort`方法中，不是数组元素所创建的子data scope


```js
filter: function(item) {
  var index = this.items.indexOf(item);
  ...
}
```

### Nesting dom-repeat templates {#nesting-templates}

>When nesting multiple `dom-repeat` templates, you may want to access data
from a parent scope. Inside a `dom-repeat`, you can access any properties available
to the parent scope unless they're hidden by a property in the current scope.
比如类继承一样，子scope中可以取父scope的所有没有被覆盖的属性。

>To access properties from nested `dom-repeat` templates, use the `as` attribute to
assign a different name for the item property. Use the `index-as` attribute to assign a
different name for the index property.
可以使用`as`标记来对默认的`item`来重命名
可以使用`index-as`标记来对默认的`index`来重命名

```html
<div> Employee list: </div>
<template is="dom-repeat" items="{{employees}}" as="employee">
    <div>First name: <span>{{employee.first}}</span></div>
    <div>Last name: <span>{{employee.last}}</span></div>

    <div>Direct reports:</div>

    <template is="dom-repeat" items="{{employee.reports}}" as="report" index-as="report_no">
      <div><span>{{report_no}}</span>.
           <span>{{report.first}}</span> <span>{{report.last}}</span>
      </div>
    </template>
</template>
```

### Forcing synchronous renders 

默认的`dom-repeat`是异步执行的，但是可以调用`render`方法使之立即同步渲染。
此方法性能代价比较大，适用于下面几种情况：

- 单元测试的时候保证检查之前所有的选项都被渲染完毕
- 在要滚动到某一特定元素之前保证其已经渲染完毕
- 当外部数据变化导致的`sort/filter`方法需要重新运行

注意： `render`方法只会更新模型数据的Observable change，如果要全局更新见下一节。

### Forcing the template to update
强制刷新列表元素，像之前所说过的三种解决方案：

- `notifySplices`，如果你知道数组的具体变更方式
- 克隆数组，如果必要，可以深度克隆，性能不佳
- 使用`mutableData`标签再`this.notifyPath(items)`


### Improve performance for large lists

当列表数据量很大的时候，`dom-repeat`支持延迟加载。设置`initialCount`属性，可以启动`chunked mode`，`dom-repeat`首先会渲染`initialCount`个元素，然后按照一个`animation frame`一`chunk`的形式渲染其他元素，这样能够让UI在渲染的过程中处理用户的输入。可以查看`renderedItemCount`属性（只读）来获得当前已被渲染的元素总数。

>`dom-repeat` adjusts the number of items rendered in each chunk to try and maintain a target framerate. You can further tune rendering by setting targetFramerate.

`dom-repeat`尝试去维护一个targetFramerate函数来调整每一个渲染的`chunk`里面的元素个数，具体：[targetFramerate](https://www.polymer-project.org/2.0/docs/api/elements/Polymer.DomRepeat#property-targetFramerate)




## Data bind an array selection (array-selector) 

>Keeping structured data in sync requires that Polymer understand the path associations of data being bound.  The `array-selector` element ensures path linkage when selecting specific items from an array.

`array-selector` 可以选择数组里面的元素，并自动把选择出来的元素的路径跟这些元素的原来路径进行连接。(linkPaths)

```html
<link rel="import" href="components/polymer/polymer-element.html">
<! -- import template repeater -->
<link rel="import" href="components/polymer/lib/elements/dom-repeat.html">
<!-- import array selector -->
<link rel="import" href="components/polymer/lib/elements/array-selector.html">

<dom-module id="x-custom">

  <template>

    <div> Employee list: </div>
    <template is="dom-repeat" id="employeeList" items="{{employees}}">
        <div>First name: <span>{{item.first}}</span></div>
        <div>Last name: <span>{{item.last}}</span></div>
        <button on-click="toggleSelection">Select</button>
    </template>

    <array-selector id="selector" items="{{employees}}" selected="{{selected}}" multi toggle></array-selector>

    <div> Selected employees: </div>
    <template is="dom-repeat" items="{{selected}}">
        <div>First name: <span>{{item.first}}</span></div>
        <div>Last name: <span>{{item.last}}</span></div>
    </template>

  </template>

  <script>
    class XCustom extends Polymer.Element {

      static get is() { return 'x-custom'; }

      static get properties() {
        return {
          employees: {
            type: Array,
            value() {
              return [
                {first: 'Bob', last: 'Smith'},
                {first: 'Sally', last: 'Johnson'},
                // ...
              ];
            }
          }
        }
      }

      toggleSelection(e) {
        var item = this.$.employeeList.itemForElement(e.target);
        this.$.selector.select(item);
      }
    }

    customElements.define(XCustom.is, XCustom);
  </script>

</dom-module>
```

- `items`属性接收一个数组。
- 两个常见API: `select(item)\deselect(item)`
- 对于数组元素子属性的任何变化都会同步到items数组中（path已经link了）
- `multi`属性可以开关多选


## Conditional templates (dom-if)

`dom-if` 可以按条件来显示其中的内容。最开始的时候`dom-if`中没有元素，当把`if`属性设置为`true`时里面就会出现template中的的元素，当`if`属性再次变为`false`时，其内部元素默认不会被删除，而是直接隐藏。可以设置`restamp`为`true`来禁止这种隐藏行为。

跟`dom-if`一样，有两种方式来定义：

*  在Polymer element template内部，使用简写方式

   ```html
    <template is="dom-if" if="{{condition}}">
      ...
    </template>
    ```
*   在Polymer element template外部，使用`<dom-if>`标签
    ```html
    <dom-if>
      <template>
        ...
      </template>
    </dom-if> 
    ```
    这种情况需要手动设置`dom-if`的属性：
    ```js
    var conditional = document.querySelector('dom-if');
    conditional.if = true;
    ```

代码实例：

  ```html
  <link rel="import" href="components/polymer/polymer-element.html">
  <! -- import conditional template -->
  <link rel="import" href="components/polymer/lib/elements/dom-if.html">

  <dom-module id="x-custom">

    <template>

      <!-- All users will see this -->
      <my-user-profile user="{{user}}"></my-user-profile>


      <template is="dom-if" if="{{user.isAdmin}}">
        <!-- Only admins will see this. -->
        <my-admin-panel user="{{user}}"></my-admin-panel>
      </template>

    </template>

    <script>
      class XCustom extends Polymer.Element {

        static get is() { return 'x-custom'; }

        static get properties() {
          return {
            user: Object
          }
        }

      }

      customElements.define(XCustom.is, XCustom);
    </script>

  </dom-module>
  ```
>Conditional templates introduce some overhead, so they shouldn't be used for small UI elements that could be easily shown and hidden using CSS.

条件模板会引入一些开销，因此不适合一些可以直接设置CSS来控制显示和隐藏的小元素。
但是条件模板也适用于下面几种情况：

-   懒惰加载
-   节省大型复杂网站的内存消耗（`restam=true`会带来性能上的损失）


## Auto-binding templates (dom-bind）
>Polymer data binding is only available in templates that are managed by Polymer. So data binding works inside an element's DOM template (or inside a `dom-repeat` or `dom-if` template), but not for elements placed in the main document.

>To use Polymer bindings **without** defining a new custom element, use the `<dom-bind>` element.  This template immediately stamps the contents of its child templateinto the main document. Data bindings in an auto-binding template use the `<dom-bind>` element itself as the binding scope.

Polymer的数据绑定只能在template中，为了简化流程，使用自动绑定(`<dom-bind>`)能够在不定义新的自定义元素的前提下进行数据绑定。

```html
<!doctype html>
<html>
<head>
  <meta charset="utf-8">
  <script src="components/webcomponentsjs/webcomponents-lite.js"></script>
  <link rel="import" href="polymer/lib/elements/dom-bind.html">
  <link rel="import" href="polymer/lib/elements/dom-repeat.html">

</head>
<body>
  <!-- Wrap elements with auto-binding template to -->
  <!-- allow use of Polymer bindings in main document -->
  <dom-bind>
    <template>

      <template is="dom-repeat" items="{{data}}">
        <div>{{item.name}}: {{item.price}}</div>
      </template>

    </template>
  </dom-bind>
  <script>
    var autobind = document.querySelector('dom-bind');

    // The dom-change event signifies when the template has stamped its DOM.
    autobind.addEventListener('dom-change', function() {
      console.log('template is ready.')
    });

    // set data property on dom-bind
    autobind.data = [
      { name: 'book', price: '$5.00'},
      { name: 'pencil', price: '$1.00'},
      { name: 'flux capacitor', price: '$8,000,000.00'}
    ];
  </script>
</body>
</html>
```

注意：

1. 自动绑定只能在Polymer element之外定义(因此只有一种定义方式)
2. `dom-bind`也提供了一个`render`方法来进行强制同步刷新。
3. `dom-bind`同样也用一个`mutableData`属性来开关脏检测机制。



## dom-change event

>When one of the template helper elements updates the DOM tree, it fires a `dom-change` event.

当任意一个template helper elements更新了DOM树时，它们都会触发一个`dom-change`事件。

>In most cases, you should interact with the created DOM by changing the _model data_, not by interacting directly with the created nodes. For those cases where you need to access the nodes directly, you can use the `dom-change` event.

原则上，我们不应该直接与DOM进行交互，而应该修改对应的模型数据。如果的确需要这样做的话，可以监听这个的`dom-change`事件来完成。

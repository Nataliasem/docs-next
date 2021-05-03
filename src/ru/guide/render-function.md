# Render-функции

Для большинства случаев рекомендуется использовать шаблоны для создания приложений. Но бывают ситуации, когда необходима вся программная мощь JavaScript. В таких случаях можно использовать **render-функции**.

Рассмотрим небольшой пример, где функция `render()` оказалась бы практичнее обычного подхода. Например, требуется сгенерировать заголовки с якорными ссылками:

```html
<h1>
  <a name="hello-world" href="#hello-world">
    Hello world!
  </a>
</h1>
```

Такие заголовки будут использоваться часто, поэтому сразу стоит создать компонент:

```vue
<anchored-heading :level="1">Hello world!</anchored-heading>
```

Компонент должен генерировать заголовок, в зависимости от входного параметра `level`, что скорее всего приведёт к такому решению:

```js
const { createApp } = Vue

const app = createApp({})

app.component('anchored-heading', {
  template: `
    <h1 v-if="level === 1">
      <slot></slot>
    </h1>
    <h2 v-else-if="level === 2">
      <slot></slot>
    </h2>
    <h3 v-else-if="level === 3">
      <slot></slot>
    </h3>
    <h4 v-else-if="level === 4">
      <slot></slot>
    </h4>
    <h5 v-else-if="level === 5">
      <slot></slot>
    </h5>
    <h6 v-else-if="level === 6">
      <slot></slot>
    </h6>
  `,
  props: {
    level: {
      type: Number,
      required: true
    }
  }
})
```

Шаблон такого компонента выглядит не очень. Он не только многословен, но и дублирует `<slot></slot>` для каждого уровня заголовка. А при добавлении нового элемента якоря, потребуется снова дублировать его в каждой ветке `v-if/v-else-if`.

Хотя шаблоны отлично работают для большинства компонентов, очевидно, что данный случай не один из них. Давайте перепишем компонент с помощью функции `render()`:

```js
const { createApp, h } = Vue

const app = createApp({})

app.component('anchored-heading', {
  render() {
    return h(
      'h' + this.level, // имя тега
      {}, // входные параметры/атрибуты
      this.$slots.default() // массив дочерних элементов
    )
  },
  props: {
    level: {
      type: Number,
      required: true
    }
  }
})
```

Реализация с функцией `render()` получилась гораздо проще, но требует больше знаний о свойствах экземпляра компонента. Для этого примера потребуется знать, что при передаче дочерних элементов в компонент без директивы `v-slot`, например `Hello world!` внутрь `anchored-heading`, они будут доступны в экземпляре компонента через `$slots.default()`. Если это ещё непонятно, **рекомендуем сначала изучить раздел [API свойств экземпляра](../api/instance-properties.md) перед углублением в render-функции.**

## DOM-дерево

Прежде чем погрузиться в изучение render-функций, важно немного подробнее разобраться в том, как работают браузеры. Возьмём, к примеру, этот HTML:

```html
<div>
  <h1>My title</h1>
  Some text content
  <!-- TODO: Add tagline -->
</div>
```

Когда браузер читает этот код, он строит [дерево «DOM узлов»](https://javascript.info/dom-nodes), чтобы помочь себе отслеживать всё.

Для HTML из примера выше дерево DOM-узлов получится таким:

![Визуализация дерева DOM](/images/dom-tree.png)

Каждый элемент является узлом. Каждый текст является узлом. Каждый комментарий является узлом! Каждый узел может иметь дочерние элементы (т.е. каждый узел может содержать другие узлы).

Эффективно обновлять все эти узлы — непростая задача, но, к счастью, это не потребуется делать вручную. Требуется лишь сообщать Vue какой HTML нужен на странице в шаблоне:

```html
<h1>{{ blogTitle }}</h1>
```

Или в render-функции:

```js
render() {
  return h('h1', {}, this.blogTitle)
}
```

В обоих случаях Vue будет автоматически поддерживать страницу в обновлённом состоянии, даже при изменениях значения `blogTitle`.

## Виртуальное DOM-дерево

Vue поддерживает страницу в обновлённом состоянии с помощью **виртуального DOM**. Он помогает определить изменения, которые необходимо внести в реальный DOM. Взглянем внимательнее на эту строку:

```js
return h('h1', {}, this.blogTitle)
```

Что вернёт функция `h()`? Это _не совсем_ настоящий DOM-элемент. Возвращается обычный объект с информацией для Vue, какой узел должен отобразиться на странице, в том числе описание любых дочерних элементов. Это описание называют «виртуальным узлом» или «виртуальной нодой», обычно сокращая до **VNode**. «Виртуальным DOM» можно назвать всё дерево из VNode, созданных по дереву компонентов Vue.

## Аргументы `h()`

Функция `h()` — утилита для создания VNode. Возможно её стоило назвать `createVNode()` для точности, но она называется `h()` из-за частого использования и для краткости. Функция принимает три аргумента:

```js
// @returns {VNode}
h(
  // {String | Object | Function} тег
  // Имя HTML-тега, компонента, асинхронного или функционального компонента.
  // Использование функции, возвращающей null, будет отрисовывать комментарий.
  //
  // Обязательный параметр
  'div',

  // {Object} входные параметры
  // Объект, соответствующий атрибутам, входным параметрам
  // и событиям, которые использовались бы в шаблоне.
  //
  // Опциональный
  {},

  // {String | Array | Object} дочерние элементы
  // Дочерние VNode, созданные с помощью `h()`,
  // или строки для получения 'текстовых VNode' или
  // объект со слотами.
  //
  // Опциональный
  [
    'Какой-то текст в начале.',
    h('h1', 'Заголовок'),
    h(MyComponent, {
      someProp: 'foobar'
    })
  ]
)
```

Если входных параметров нет, то дочерние элементы можно передать вторым аргументом. В случаях, если это может добавить путаницы, можно указывать `null` вторым аргументом, чтобы третьим явно передавать дочерние элементы.

## Полный пример

С полученными знаниями можно теперь завершить начатый компонент:

```js
const { createApp, h } = Vue

const app = createApp({})

/** Рекурсивно получаем текст из дочерних узлов */
function getChildrenTextContent(children) {
  return children
    .map(node => {
      return typeof node.children === 'string'
        ? node.children
        : Array.isArray(node.children)
        ? getChildrenTextContent(node.children)
        : ''
    })
    .join('')
}

app.component('anchored-heading', {
  render() {
    // создаём ID в kebab-case из текстового содержимого дочерних узлов
    const headingId = getChildrenTextContent(this.$slots.default())
      .toLowerCase()
      .replace(/\W+/g, '-') // заменяем не-буквенные символы на тире
      .replace(/(^-|-$)/g, '') // удаляем в начале и конце висящие тире

    return h('h' + this.level, [
      h(
        'a',
        {
          name: headingId,
          href: '#' + headingId
        },
        this.$slots.default()
      )
    ])
  },
  props: {
    level: {
      type: Number,
      required: true
    }
  }
})
```

## Ограничения

### VNode должны быть уникальными

В дереве компонентов все VNode должны быть уникальными. Это означает, что следующая render-функция некорректна:

```js
render() {
  const myParagraphVNode = h('p', 'hi')
  return h('div', [
    // НЕПРАВИЛЬНО - одинаковые VNode!
    myParagraphVNode, myParagraphVNode
  ])
}
```

Если требуется многократно дублировать один и тот же элемент/компонент, то реализовать это можно с помощью функции-фабрики. Например, следующая render-функция будет абсолютно корректным способом для отрисовки 20 одинаковых параграфов:

```js
render() {
  return h('div',
    Array.from({ length: 20 }).map(() => {
      return h('p', 'hi')
    })
  )
}
```

## Создание VNode компонентов

При создании VNode для компонента первым аргументом `h` должен быть сам компонент:

```js
render() {
  return h(ButtonCounter)
}
```

Если необходимо разрешить компонент по имени, можно использовать `resolveComponent`:

```js{1,6}
const { h, resolveComponent } = Vue

// ...

render() {
  const ButtonCounter = resolveComponent('ButtonCounter')
  return h(ButtonCounter)
}
```

Функцию `resolveComponent` используют шаблоны под капотом для разрешения компонентов по имени.

В функции `render` обычно приходится использовать `resolveComponent` для компонентов [зарегистрированных глобально](component-registration.md#глобальная-регистрация). При [локальной регистрации компонентов](component-registration.md#локальная-регистрация) обычно можно обойтись без неё. Рассмотрим следующий пример:

```js
// Такой код можно упростить
components: {
  ButtonCounter
},
render() {
  return h(resolveComponent('ButtonCounter'))
}
```

Вместо регистрации компонента по имени, а затем поиска, можно сразу его использовать:

```js
render() {
  return h(ButtonCounter)
}
```

## Замена возможностей шаблона обычным JavaScript

### `v-if` и `v-for`

Всё что используется можно легко реализовать на простом JavaScript, для render-функции Vue не создаёт никакой проприетарной альтернативы. Например, шаблон с `v-if` и `v-for`:

```html
<ul v-if="items.length">
  <li v-for="item in items">{{ item.name }}</li>
</ul>
<p v-else>Элементов не найдено.</p>
```

Можно переписать с использованием JavaScript `if`/`else` и `map()` в render-функции:

```js
props: ['items'],
render() {
  if (this.items.length) {
    return h('ul', this.items.map((item) => {
      return h('li', item.name)
    }))
  } else {
    return h('p', 'Элементов не найдено.')
  }
}
```

В шаблоне иногда удобно использовать тег `<template>`, чтобы указать `v-if` или `v-for`. При миграции на использование `render`-функции тег `<template>` можно просто опустить.

### `v-model`

На этапе компиляции шаблона директива `v-model` раскладывается на входные параметры `modelValue` и `onUpdate:modelValue` — их потребуется указать самостоятельно:

```js
props: ['modelValue'],
emits: ['update:modelValue'],
render() {
  return h(SomeComponent, {
    modelValue: this.modelValue,
    'onUpdate:modelValue': value => this.$emit('update:modelValue', value)
  })
}
```

### `v-on`

Необходимо предоставить правильное имя входного параметра для обработчика события, например, для обработки событий `click` имя входного параметра должно быть `onClick`.

```js
render() {
  return h('div', {
    onClick: $event => console.log('кликнули!', $event.target)
  })
}
```

#### Модификаторы событий

Модификаторы событий `.passive`, `.capture` и `.once` необходимо указывать после имени события в camelCase.

Например:

```js
render() {
  return h('input', {
    onClickCapture: this.doThisInCapturingMode,
    onKeyupOnce: this.doThisOnce,
    onMouseoverOnceCapture: this.doThisOnceInCapturingMode
  })
}
```

Для любых других событий и модификаторов клавиш специального API не требуется, потому что в обработчике события можно использовать нативные свойства и методы:

| Модификатор(ы)                                             | Эквивалент в обработчике                                                                                   |
|------------------------------------------------------------|------------------------------------------------------------------------------------------------------------|
| `.stop`                                                    | `event.stopPropagation()`                                                                                  |
| `.prevent`                                                 | `event.preventDefault()`                                                                                   |
| `.self`                                                    | `if (event.target !== event.currentTarget) return`                                                         |
| Клавиши:<br>например, `.enter`                             | `if (event.key !== 'Enter') return`<br><br>Замените `Enter` на [соответствующий key](http://keycode.info/) |
| Модификаторы клавиш:<br>`.ctrl`, `.alt`, `.shift`, `.meta` | `if (!event.ctrlKey) return`<br><br>Замените `ctrlKey` на `altKey`, `shiftKey` или `metaKey`               |

Пример обработчика со всеми этими модификаторами, используемыми вместе:

```js
render() {
  return h('input', {
    onKeyUp: event => {
      // Отменяем обработку, если элемент вызвавший событие
      // не является элементом, к которому событие было привязано
      if (event.target !== event.currentTarget) return

      // Отменяем обработку, если код клавиши не соответствовал
      // enter и клавиша shift не была нажата в то же время
      if (!event.shiftKey || event.key !== 'Enter') return

      // Останавливаем всплытие события
      event.stopPropagation()

      // Останавливаем поведение по умолчанию для этого элемента
      event.preventDefault()
      // ...
    }
  })
}
```

### Слоты

Доступ к содержимому слотов в виде массива VNode можно получить через [`this.$slots`](../api/instance-properties.md#slots):

```js
render() {
  // `<div><slot></slot></div>`
  return h('div', this.$slots.default())
}
```

```js
props: ['message'],
render() {
  // `<div><slot :text="message"></slot></div>`
  return h('div', this.$slots.default({
    text: this.message
  }))
}
```

Для VNode компонента необходимо передать дочерние элементы в `h` в виде объекта, а не массива. Каждое свойство будет использовано для заполнения одноимённого слота:

```js
render() {
  // `<div><child v-slot="props"><span>{{ props.text }}</span></child></div>`
  return h('div', [
    h(
      resolveComponent('child'),
      null,
      // передаём `slots` как дочерний объект в формате
      // { slotName: props => VNode | Array<VNode> }
      {
        default: (props) => h('span', props.text)
      }
    )
  ])
}
```

Слоты передаются как функции, что позволяет дочернему компоненту управлять созданием содержимого каждого слота. Любые реактивные данные должны быть доступны внутри функции слота, чтобы гарантировать, что они зарегистрированы как зависимость дочернего компонента, а не родительского. И наоборот, обращения к `resolveComponent` нужно делать вне функции слота, иначе они будут разрешаться относительно неправильного компонента:

```js
// `<MyButton><MyIcon :name="icon" />{{ text }}</MyButton>`
render() {
  // Вызовы resolveComponent должны находиться вне функции слота
  const Button = resolveComponent('MyButton')
  const Icon = resolveComponent('MyIcon')

  return h(
    Button,
    null,
    {
      // Используем стрелочную функцию для сохранения значения `this`
      default: (props) => {
        // Реактивные свойства должны считываться внутри функции слота,
        // чтобы они стали зависимостями для отрисовки дочернего компонента
        return [
          h(Icon, { name: this.icon }),
          this.text
        ]
      }
    }
  )
}
```

Если компонент получает слоты из родителя — их можно передать напрямую дочернему:

```js
render() {
  return h(Panel, null, this.$slots)
}
```

Но можно также передавать их по-отдельности или оборачивать по необходимости:

```js
render() {
  return h(
    Panel,
    null,
    {
      // Если хотим просто передать функцию слота
      header: this.$slots.header,
      
      // Если требуется как-то управлять слотом,
      // тогда нужно обернуть его в новую функцию
      default: (props) => {
        const children = this.$slots.default ? this.$slots.default(props) : []
        
        return children.concat(h('div', 'Extra child'))
      }
    } 
  )
}
```

### `<component>` и `is`

Шаблоны для реализации атрибута `is` используют под капотом `resolveDynamicComponent`. Можно воспользоваться этой же функцией, если в создаваемой `render`-функции требуется вся гибкость, предоставляемая `is`:

```js{1,7}
const { h, resolveDynamicComponent } = Vue

// ...

// `<component :is="name"></component>`
render() {
  const Component = resolveDynamicComponent(this.name)
  return h(Component)
}
```

Аналогично `is`, `resolveDynamicComponent` поддерживает передачу имени компонента, имени HTML-элемента или объекта с опциями компонента.

Но обычно такой уровень гибкости не требуется. Поэтому `resolveDynamicComponent` часто можно заменить на более конкретную альтернативу.

К примеру, если нужно поддерживать только имена компонентов — можно использовать `resolveComponent`.

Если VNode всегда будет HTML-элементом — можно передавать имя напрямую в `h`:

```js
// `<component :is="bold ? 'strong' : 'em'"></component>`
render() {
  return h(this.bold ? 'strong' : 'em')
}
```

Аналогично, если передаваемое в `is` значение будет объектом опций компонента, то не нужно ничего разрешать и можно сразу передать его первым аргументом в `h`.

Подобно тегу `<template>`, тег `<component>` нужен в шаблонах только в качестве синтаксического сахара и его следует опустить при миграции на `render`-функции.

### Пользовательские директивы

Пользовательские директивы можно применить к VNode с помощью [`withDirectives`](../api/global-api.md#withdirectives):

```js
const { h, resolveDirective, withDirectives } = Vue
// ...
// <div v-pin:top.animate="200"></div>
render () {
  const pin = resolveDirective('pin')
  return withDirectives(h('div'), [
    [pin, 200, 'top', { animate: true }]
  ])
}
```

Функция [`resolveDirective`](../api/global-api.md#resolvedirective) используется в шаблонах под капотом, чтобы разрешить директиву по имени. Это нужно лишь в случаях, когда нет прямого доступа к объекту с объявлением директивы.

### Встроенные компоненты

[Встроенные компоненты](../api/built-in-components.md), такие как `<keep-alive>`, `<transition>`, `<transition-group>` и `<teleport>` по умолчанию не регистрируются глобально. Это позволяет системе сборки выполнять tree-shaking, чтобы добавлять эти компоненты в сборку только в случае, если они используются. Но это также означает, что не выйдет получить к ним доступ с помощью `resolveComponent` или `resolveDynamicComponent`.

Шаблоны имеют специальную обработку для этих компонентов, автоматически импортируя их при использовании. Но при создании собственных `render` функций импортировать потребуется их самостоятельно:

```js
const { h, KeepAlive, Teleport, Transition, TransitionGroup } = Vue

// ...

render () {
  return h(Transition, { mode: 'out-in' }, /* ... */)
}
```

## Возвращаемые значения render-функций

Во всех примерах, рассматривавшихся ранее, функция `render` возвращала один корневой узел VNode. Но могут быть и другие варианты.

Если вернуть строку, то будет создана текстовая VNode без какого-либо элемента-обёртки:

```js
render() {
  return 'Привет мир!'
}
```

Также можно вернуть массив дочерних узлов, не оборачивая их в корневой узел. В таком случае будет создан фрагмент:

```js
// Аналогично шаблону `Пример<br>мир!`
render() {
  return [
    'Привет',
    h('br'),
    'мир!'
  ]
}
```

Если компоненту не нужно ничего отображать (например, потому что ещё загружаются данные), то можно просто вернуть `null`. Тогда в DOM будет создан узел комментария.

## JSX

При создании множества `render`-функций может быть мучительно писать подобное:

```js
h(
  resolveComponent('anchored-heading'),
  {
    level: 1
  },
  {
    default: () => [h('span', 'Привет'), ' мир!']
  }
)
```

Особенно, когда эквивалент в шаблоне выглядит очень лаконично:

```vue
<anchored-heading :level="1"> <span>Привет</span> мир! </anchored-heading>
```

Поэтому есть [плагин для Babel](https://github.com/vuejs/jsx-next), чтобы использовать JSX во Vue и получить синтаксис, близкий к шаблонам:

```jsx
import AnchoredHeading from './AnchoredHeading.vue'

const app = createApp({
  render() {
    return (
      <AnchoredHeading level={1}>
        <span>Привет</span> мир!
      </AnchoredHeading>
    )
  }
})

app.mount('#demo')
```

Подробнее о том, как JSX преобразуется в JavaScript смотрите в [документации плагина](https://github.com/vuejs/jsx-next#installation).

## Функциональные компоненты

Functional components are an alternative form of component that don't have any state of their own. They are rendered without creating a component instance, bypassing the usual component lifecycle.

To create a functional component we use a plain function, rather than an options object. The function is effectively the `render` function for the component. As there is no `this` reference for a functional component, Vue will pass in the `props` as the first argument:

```js
const FunctionalComponent = (props, context) => {
  // ...
}
```

The second argument, `context`, contains three properties: `attrs`, `emit`, and `slots`. These are equivalent to the instance properties [`$attrs`](../api/instance-properties.md#attrs), [`$emit`](../api/instance-methods.md#emit), and [`$slots`](../api/instance-properties.md#slots) respectively.

Most of the usual configuration options for components are not available for functional components. However, it is possible to define [`props`](../api/options-data.md#props) and [`emits`](../api/options-data.md#emits) by adding them as properties:

```js
FunctionalComponent.props = ['value']
FunctionalComponent.emits = ['click']
```

If the `props` option is not specified, then the `props` object passed to the function will contain all attributes, the same as `attrs`. The prop names will not be normalized to camelCase unless the `props` option is specified.

Functional components can be registered and consumed just like normal components. If you pass a function as the first argument to `h`, it will be treated as a functional component.

## Компиляция шаблона

Интересный факт, все шаблоны во Vue компилируются в render-функции. Обычно нет нужды знать такие детали реализации, но может быть любопытно увидеть как же компилируются те или иные возможности шаблона. Небольшая демонстрация ниже показывает работу метода `Vue.compile` при компиляции строковых шаблонов на лету, попробуйте сами:

<iframe src="https://vue-next-template-explorer.netlify.app/" width="100%" height="420"></iframe>

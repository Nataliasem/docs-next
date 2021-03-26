# Конфигурация приложения

Каждое приложение Vue предоставляет доступ к объекту `config`, который содержит настройки конфигурации для этого приложения:

```js
const app = createApp({})

console.log(app.config)
```

Перед монтированием приложения можно изменять свойства, перечисленные ниже.

## errorHandler

- **Тип:** `Function`

- **По умолчанию:** `undefined`

- **Использование:**

```js
app.config.errorHandler = (err, vm, info) => {
  // Обработка ошибки
  // `info` — это специфичная для Vue информация об ошибке.
  // Например, в каком хуке жизненного цикла была найдена ошибка
}
```

Добавление обработчика неперехваченных ошибок во время отрисовки компонентов и наблюдателей. В качестве аргументов обработчик получает ошибку и экземпляр приложения.

> Сервисы отслеживания ошибок [Sentry](https://sentry.io/for/vue/) и [Bugsnag](https://docs.bugsnag.com/platforms/browsers/vue/) предоставляют официальную интеграцию с использованием этого свойства.

## warnHandler

- **Тип:** `Function`

- **По умолчанию:** `undefined`

- **Использование:**

```js
app.config.warnHandler = function(msg, vm, trace) {
  // `trace` — это трассировка иерархии компонентов
}
```

Определение пользовательского обработчика для предупреждений Vue во время выполнения. Работает только в режиме разработки и игнорируется в production.

## globalProperties

- **Тип:** `[key: string]: any`

- **По умолчанию:** `undefined`

- **Использование:**

```js
app.config.globalProperties.foo = 'bar'

app.component('child-component', {
  mounted() {
    console.log(this.foo) // 'bar'
  }
})
```

Добавление глобального свойства, к которому можно обращаться из любого компонента приложения. В случае конфликта имён свойства компонента имеют приоритет.

Данный подход заменяет расширение `Vue.prototype` во Vue 2.x:

```js
// Раньше
Vue.prototype.$http = () => {}

// Сейчас
const app = createApp({})
app.config.globalProperties.$http = () => {}
```

## isCustomElement

- **Тип:** `(tag: string) => boolean`

- **По умолчанию:** `undefined`

- **Использование:**

```js
// любой элемент, начинающийся с 'ion-', будет считаться пользовательским
app.config.isCustomElement = tag => tag.startsWith('ion-')
```

Определяет метод распознавания пользовательских элементов, определённых вне Vue (например, использование Web Components API). Если компонент совпадает с условием, то его не нужно регистрировать и Vue не выдаст предупреждения `Unknown custom element`.

> Обратите внимание, что нет необходимости указывать HTML и SVG теги, — парсер Vue определяет их автоматически.

:::tip Важно
Эта опция конфигурации работает только при использовании компилятора шаблонов. Для только-runtime сборки `isCustomElement` нужно передавать в `@vue/compiler-dom` в настройках сборки — например, через [опцию vue-loader `compilerOptions`](https://vue-loader.vuejs.org/ru/options.html#compileroptions).
:::

## optionMergeStrategies

- **Тип:** `{ [key: string]: Function }`

- **По умолчанию:** `{}`

- **Использование:**

```js
const app = createApp({
  mounted() {
    console.log(this.$options.hello)
  }
})

app.config.optionMergeStrategies.hello = (parent, child, vm) => {
  return `Hello, ${child}`
}

app.mixin({
  hello: 'Vue'
})

// 'Hello, Vue'
```

Определение пользовательской функции для слияния опций.

Первый аргумент функции слияния — значения опций родительского элемента. Второй аргумент — опции дочернего элемента.
Третий — контекст действующего экземпляра приложения.

- **См. также:** [Пользовательские функции слияния](../guide/mixins.md#custom-option-merge-strategies)

## performance

- **Тип:** `boolean`

- **По умолчанию:** `false`

- **Использование**:

Установка в значение `true` включит отслеживание производительности во время инициализации, компиляции, отрисовки и обновления компонентов в инструментах разработчика браузера. Работает только в режиме разработки в браузерах, поддерживающих API [performance.mark](https://developer.mozilla.org/en-US/docs/Web/API/Performance/mark).

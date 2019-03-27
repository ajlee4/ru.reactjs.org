---
id: hooks-reference
title: Справочник API хуков
permalink: docs/hooks-reference.html
prev: hooks-custom.html
next: hooks-faq.html
---

*Хуки* — нововведение в React 16.8, которое позволяет использовать состояние и другие возможности React без написания классов.

На этой странице описан API, относящийся к встроенным хукам React.

Если вы новичок в хуках, вы можете сначала ознакомиться с [общим обзором](/docs/hooks-overview.html). Вы также можете найти полезную информацию в главе [«Хуки: ответы на вопросы»](/docs/hooks-faq.html).

- [Основные хуки](#basic-hooks)
  - [`useState`](#usestate)
  - [`useEffect`](#useeffect)
  - [`useContext`](#usecontext)
- [Дополнительные хуки](#additional-hooks)
  - [`useReducer`](#usereducer)
  - [`useCallback`](#usecallback)
  - [`useMemo`](#usememo)
  - [`useRef`](#useref)
  - [`useImperativeHandle`](#useimperativehandle)
  - [`useLayoutEffect`](#uselayouteffect)
  - [`useDebugValue`](#usedebugvalue)

## Основные хуки {#basic-hooks}

### `useState` {#usestate}

```js
const [state, setState] = useState(initialState);
```

Возвращает значение с состоянием и функцию для его обновления.

Во время первоначального рендеринга возвращаемое состояние (`state`) совпадает со значением, переданным в качестве первого аргумента (`initialState`).

Функция `setState` используется для обновления состояния. Она принимает новое значение состояния и ставит в очередь повторный рендер компонента.

```js
setState(newState);
```

Во время последующих повторных рендеров первое значение, возвращаемое `useState`, всегда будет самым последним состоянием после применения обновлений.

>Примечание
>
>React гарантирует, что идентичность функции `setState` стабильна и не изменяется при повторных рендерах. Поэтому её можно безопасно не включать в списки зависимостей хуков `useEffect` и `useCallback`.

#### Функциональные обновления {#functional-updates}

Если новое состояние вычисляется с использованием предыдущего состояния, вы можете передать функцию `setState`. Функция получит предыдущее значение и вернёт обновленное значение. Вот пример компонента счётчик, который использует обе формы `setState`:

```js
function Counter({initialCount}) {
  const [count, setCount] = useState(initialCount);
  return (
    <>
      Счёт: {count}
      <button onClick={() => setCount(initialCount)}>Сбросить</button>
      <button onClick={() => setCount(prevCount => prevCount + 1)}>+</button>
      <button onClick={() => setCount(prevCount => prevCount - 1)}>-</button>
    </>
  );
}
```
Кнопки «+» и «-» используют функциональную форму, потому что обновленное значение основано на предыдущем значении. Но кнопка «Сбросить» использует обычную форму, потому что она всегда устанавливает счетчик обратно в 0.

> Примечание
>
> В отличие от метода `setState`, который вы можете найти в классовых компонентах, `setState` не объединяет объекты обновления автоматически. Вы можете повторить это поведение, комбинируя форму функции обновления с синтаксисом расширения объекта:
>
> ```js
> setState(prevState => {
>   // Object.assign также будет работать
>   return {...prevState, ...updatedValues};
> });
> ```
>
> Другой вариант – `useReducer`, который больше подходит для управления объектами состояния, содержащими несколько значений.

#### Ленивая инициализация состояния {#lazy-initial-state}

Аргумент `initialState` – это состояние, используемое во время начального рендеринга. В последующих рендерах это не учитывается. Если начальное состояние является результатом дорогостоящих вычислений, вы можете вместо этого предоставить функцию, которая будет выполняться только при начальном рендеринге:

```js
const [state, setState] = useState(() => {
  const initialState = someExpensiveComputation(props);
  return initialState;
});
```

#### Досрочное прекращение обновления состояния {#bailing-out-of-a-state-update}

Если вы обновите состояние хука тем же значением, что и текущее состояние, React досрочно выйдет из хука без повторного рендера дочерних элементов и запуска эффектов. (React использует [алгоритм сравнения `Object.is`](https://developer.mozilla.org/ru/docs/Web/JavaScript/Reference/Global_Objects/Object/is#Description).)

Note that React may still need to render that specific component again before bailing out. That shouldn't be a concern because React won't unnecessarily go "deeper" into the tree. If you're doing expensive calculations while rendering, you can optimize them with `useMemo`.

### `useEffect` {#useeffect}

```js
useEffect(didUpdate);
```

Принимает функцию, которая содержит императивный код, возможно, с эффектами.

Мутации, подписки, таймеры, логирование и другие побочные эффекты не допускаются внутри основного тела функционального компонента (называемого _этапом рендеринга_ React). Это приведёт к запутанным ошибкам и несоответствиям в пользовательском интерфейсе.

Вместо этого используйте `useEffect`. Функция, переданная в `useEffect`, будет запущена после того, как рендер будет зафиксирован на экране. Думайте об эффектах как о лазейке из чисто функционального мира React в мир императивов.

По умолчанию эффекты запускаются после каждого завершённого рендеринга, но вы можете решить запускать их [только при изменении определённых значений](#conditionally-firing-an-effect).

#### Очистка эффекта {#cleaning-up-an-effect}

Часто эффекты создают ресурсы, которые необходимо очистить (или сбросить) перед тем, как компонент покидает экран, например подписку или идентификатор таймера. Чтобы сделать это, функция переданная в `useEffect`, может вернуть функцию очистки. Например, чтобы создать подписку:

```js
useEffect(() => {
  const subscription = props.source.subscribe();
  return () => {
    // Очистить подписку
    subscription.unsubscribe();
  };
});
```

Функция очистки запускается до удаления компонента из пользовательского интерфейса, чтобы предотвратить утечки памяти. Кроме того, если компонент рендерится несколько раз (как обычно происходит), **предыдущий эффект очищается перед выполнением следующего эффекта**. В нашем примере это означает, что новая подписка создаётся при каждом обновлении. Чтобы избежать воздействия на каждое обновление, обратитесь к следующему разделу.

#### Порядок срабатывания эффектов {#timing-of-effects}

В отличие от `componentDidMount` и `componentDidUpdate`, функция, переданная в `useEffect`, запускается во время отложенного события **после** разметки и отрисовки. Это делает хук подходящим для многих распространённых побочных эффектов, таких как настройка подписок и обработчиков событий, потому что большинство типов работы не должны блокировать обновление экрана браузером.

Однако не все эффекты могут быть отложены. Например, изменение DOM, которое видно пользователю, должно запускаться синхронно до следующей отрисовки, чтобы пользователь не замечал визуального несоответствие. (Различие концептуально схоже с пассивным и активным слушателями событий.) Для этих типов эффектов React предоставляет один дополнительный хук, называемый [`useLayoutEffect`](#uselayouteffect). Он имеет ту же сигнатуру, что и `useEffect`, и отличается только в его запуске.

Хотя `useEffect` откладывается до тех пор, пока браузер не выполнит отрисовку, он гарантированно срабатывает перед любыми новыми рендерами. React всегда полностью применяет эффекты предыдущего рендера перед началом нового обновления.

#### Условное срабатывание эффекта {#conditionally-firing-an-effect}

По умолчанию эффекты запускаются после каждого завершённого рендера. Таким образом, эффект всегда пересоздаётся, если значение какой-то из зависимости изменилось.

Однако в некоторых случаях это может быть излишним, например, в примере подписки из предыдущего раздела. Нам не нужно создавать новую подписку на каждое обновление, а только если изменился проп `source`.

Чтобы реализовать это, передайте второй аргумент в `useEffect`, который является массивом значений, от которых зависит эффект. Наш обновленный пример теперь выглядит так:

```js
useEffect(
  () => {
    const subscription = props.source.subscribe();
    return () => {
      subscription.unsubscribe();
    };
  },
  [props.source],
);
```

Теперь подписка будет создана повторно только при изменении `props.source`.

>Примечание
>
>Если вы хотите использовать эту оптимизацию, обратите внимание на то, чтобы массив включал в себя **все значения из области видимости компонента (такие как пропсы и состояние), которые могут изменяться с течением времени, и которые будут использоваться эффектом**. В противном случае, ваш код будет ссылаться на устаревшее значение из предыдущих рендеров. Отдельные страницы документации рассказывают о том, [как поступить с функциями](/docs/hooks-faq.html#is-it-safe-to-omit-functions-from-the-list-of-dependencies) и [что делать с часто изменяющимися массивами](/docs/hooks-faq.html#what-can-i-do-if-my-effect-dependencies-change-too-often).
>
>Если вы хотите запустить эффект и сбросить его только один раз (при монтировании и размонтировании), вы можете передать пустой массив (`[]`) вторым аргументом. React посчитает, что ваш эффект не зависит от *каких-либо* значений из пропсов или состояния и поэтому не будет выполнять повторных рендеров. Это не обрабатывается как особый случай -- он напрямую следует из логики работы входных массивов.
>
>Если вы передадите пустой массив (`[]`), пропсы и состояние внутри эффекта всегда будут иметь значения, присвоенные им изначально. Хотя передача `[]` ближе по модели мышления к знакомым `componentDidMount` и `componentWillUnmount`, обычно есть [более хорошие](/docs/hooks-faq.html#is-it-safe-to-omit-functions-from-the-list-of-dependencies) [способы](/docs/hooks-faq.html#what-can-i-do-if-my-effect-dependencies-change-too-often) избежать частых повторных рендеров. Не забывайте, что React откладывает выполнение `useEffect`, пока браузер не отрисует все изменения, поэтому выполнение дополнительной работы не является существенной проблемой.
>
>Мы рекомендуем использовать правило [`exhaustive-deps`](https://github.com/facebook/react/issues/14920), входящее в наш пакет правил линтера [`eslint-plugin-react-hooks`](https://www.npmjs.com/package/eslint-plugin-react-hooks#installation). Оно предупреждает, когда зависимости указаны неправильно и предлагает исправление.

Массив зависимостей не передаётся в качестве аргументов функции эффекта. Концептуально, однако, это то, что они представляют: каждое значение, на которое ссылается функция эффекта, должно также появиться в массиве зависимостей. В будущем достаточно продвинутый компилятор сможет создать этот массив автоматически.

### `useContext` {#usecontext}

```js
const value = useContext(MyContext);
```

<<<<<<< HEAD
Принимает объект контекста (значение, возвращённое из `React.createContext`) и возвращает текущее значение контекста, как указано ближайшим поставщиком контекста для данного контекста.

Когда провайдер обновляется, этот хук инициирует повторный рендер с последним значением контекста.
=======
Accepts a context object (the value returned from `React.createContext`) and returns the current context value for that context. The current context value is determined by the `value` prop of the nearest `<MyContext.Provider>` above the calling component in the tree.

When the nearest `<MyContext.Provider>` above the component updates, this Hook will trigger a rerender with the latest context `value` passed to that `MyContext` provider.

Don't forget that the argument to `useContext` must be the *context object itself*:

 * **Correct:** `useContext(MyContext)`
 * **Incorrect:** `useContext(MyContext.Consumer)`
 * **Incorrect:** `useContext(MyContext.Provider)`

A component calling `useContext` will always re-render when the context value changes. If re-rendering the component is expensive, you can [optimize it by using memoization](https://github.com/facebook/react/issues/15156#issuecomment-474590693).

>Tip
>
>If you're familiar with the context API before Hooks, `useContext(MyContext)` is equivalent to `static contextType = MyContext` in a class, or to `<MyContext.Consumer>`.
>
>`useContext(MyContext)` only lets you *read* the context and subscribe to its changes. You still need a `<MyContext.Provider>` above in the tree to *provide* the value for this context.
>>>>>>> 2304fa1a7c34b719c10cca1023003e22bf0fd137

## Дополнительные хуки {#additional-hooks}

Следующие хуки являются вариантами базовых из предыдущего раздела или необходимы только для конкретных крайних случаев. Их не требуется основательно изучать заранее.

### `useReducer` {#usereducer}

```js
const [state, dispatch] = useReducer(reducer, initialArg, init);
```

Альтернатива для [`useState`](#usestate). Принимает редюсер типа `(state, action) => newState` и возвращает текущее состояние в паре с методом `dispatch`. (Если вы знакомы с Redux, вы уже знаете, как это работает.)

Хук `useReducer` обычно предпочтительнее `useState`, когда у вас сложная логика состояния, которая включает в себя несколько значений, или когда следующее состояние зависит от предыдущего. `useReducer` также позволяет оптимизировать производительность компонентов, которые запускают глубокие обновления, [поскольку вы можете передавать `dispatch` вместо колбэков](/docs/hooks-faq.html#how-to-avoid-passing-callbacks-down).

Вот пример счётчика из раздела [`useState`](#usestate), переписанный для использования редюсера:

```js
const initialState = {count: 0};

function reducer(state, action) {
  switch (action.type) {
    case 'increment':
      return {count: state.count + 1};
    case 'decrement':
      return {count: state.count - 1};
    default:
      throw new Error();
  }
}

function Counter({initialState}) {
  const [state, dispatch] = useReducer(reducer, initialState);
  return (
    <>
      Count: {state.count}
      <button onClick={() => dispatch({type: 'increment'})}>+</button>
      <button onClick={() => dispatch({type: 'decrement'})}>-</button>
    </>
  );
}
```

>Примечание
>
>React гарантирует, что идентичность функции `setState` стабильна и не изменяется при повторных рендерах. Поэтому её можно безопасно не включать в списки зависимостей хуков `useEffect` и `useCallback`.

#### Указание начального состояния {#specifying-the-initial-state}

Существует два разных способа инициализации состояния `useReducer`. Вы можете выбрать любой из них в зависимости от ситуации. Самый простой способ -- передать начальное состояние в качестве второго аргумента:

```js{3}
  const [state, dispatch] = useReducer(
    reducer,
    {count: initialCount}
  );
```

>Примечание
>
>React не использует соглашение об аргументах `state = initialState`, популярное в Redux. Начальное значение иногда должно зависеть от пропсов и поэтому указывается вместо вызова хука. Если вы сильно в этом уверены, вы можете вызвать `useReducer(reducer, undefined, reducer)`, чтобы эмулировать поведение Redux, но это не рекомендуется.

#### Ленивая инициализация {#lazy-initialization}

Вы также можете создать начальное состояние лениво. Для этого вы можете передать функцию `init` в качестве третьего аргумента. Начальное состояние будет установлено равным результату вызова `init(initialArg)`.

Это позволяет извлечь логику для расчёта начального состояния за пределы редюсера. Это также удобно для сброса состояния позже в ответ на действие:

```js{1-3,11-12,19,24}
function init(initialCount) {
  return {count: initialCount};
}

function reducer(state, action) {
  switch (action.type) {
    case 'increment':
      return {count: state.count + 1};
    case 'decrement':
      return {count: state.count - 1};
    case 'reset':
      return init(action.payload);
    default:
      throw new Error();
  }
}

function Counter({initialCount}) {
  const [state, dispatch] = useReducer(reducer, initialCount, init);
  return (
    <>
      Count: {state.count}
      <button
        onClick={() => dispatch({type: 'reset', payload: initialCount})}>
        Reset
      </button>
      <button onClick={() => dispatch({type: 'increment'})}>+</button>
      <button onClick={() => dispatch({type: 'decrement'})}>-</button>
    </>
  );
}
```

#### Досрочное прекращение `dispatch` {#bailing-out-of-a-dispatch}

Если вы вернёте то же значение из редюсера хука, что и текущее состояние, React выйдет без перерисовки дочерних элементов или запуска эффектов. (React использует [алгоритм сравнения Object.is](https://developer.mozilla.org/ru/docs/Web/JavaScript/Reference/Global_Objects/Object/is#Description).)

Note that React may still need to render that specific component again before bailing out. That shouldn't be a concern because React won't unnecessarily go "deeper" into the tree. If you're doing expensive calculations while rendering, you can optimize them with `useMemo`.

### `useCallback` {#usecallback}

```js
const memoizedCallback = useCallback(
  () => {
    doSomething(a, b);
  },
  [a, b],
);
```

Возвращает [запомненный](https://ru.wikipedia.org/wiki/%D0%9C%D0%B5%D0%BC%D0%BE%D0%B8%D0%B7%D0%B0%D1%86%D0%B8%D1%8F) колбэк.

Передайте встроенный колбэк и массив зависимостей. Хук `useCallback` вернёт запомненную версию колбэка, который изменяется только в случае изменения значения одной из зависимостей. Это полезно при передаче колбэков оптимизированным дочерним компонентам, которые полагаются на равенство ссылок для предотвращения ненужных рендеров (например, `shouldComponentUpdate`).

`useCallback(fn, deps)` – это эквивалент `useMemo(() => fn, deps)`.

> Примечание
>
> Массив зависимостей не передаётся в качестве аргументов для обратного вызова. Концептуально, однако, это то, что они представляют: каждое значение, указанное в обратном вызове, должно также появиться в массиве зависимостей. В будущем достаточно продвинутый компилятор может создать этот массив автоматически.
>
> Мы рекомендуем использовать правило [`exhaustive-deps`](https://github.com/facebook/react/issues/14920), входящее в наш пакет правил линтера [`eslint-plugin-react-hooks`](https://www.npmjs.com/package/eslint-plugin-react-hooks#installation). Оно предупреждает, когда зависимости указаны неправильно и предлагает исправление.

### `useMemo` {#usememo}

```js
const memoizedValue = useMemo(() => computeExpensiveValue(a, b), [a, b]);
```

Возвращает [запомненное](https://ru.wikipedia.org/wiki/%D0%9C%D0%B5%D0%BC%D0%BE%D0%B8%D0%B7%D0%B0%D1%86%D0%B8%D1%8F) значение.

Передайте «создающую» функцию и массив зависимостей. `useMemo` будет повторно вычислять запомненное значение только тогда, когда значение какой-либо из зависимостей изменилось. Эта оптимизация помогает избежать дорогостоящих вычислений при каждом рендере.

Помните, что функция, переданная `useMemo`, запускается во время рендеринга. Не делайте там ничего, что вы обычно не делаете во время рендеринга. Например, побочные эффекты принадлежат `useEffect`, а не `useMemo`.

Если массив не был передан, новое значение будет вычисляться при каждом рендере.

**Вы можете использовать `useMemo` как оптимизацию производительности, а не как семантическую гарантию.** В будущем React может решить «забыть» некоторые ранее запомненные значения и пересчитать их при следующем рендере, например, чтобы освободить память для компонентов вне области видимости экрана. Напишите свой код, чтобы он по-прежнему работал без `useMemo`, а затем добавьте его для оптимизации производительности.

> Примечание
>
> Массив зависимостей не передаётся в качестве аргументов функции. Концептуально, однако, это то, что они представляют: каждое значение, на которое ссылается функция, должно также появиться в массиве зависимостей. В будущем достаточно продвинутый компилятор может создать этот массив автоматически.
>
> Мы рекомендуем использовать правило [`exhaustive-deps`](https://github.com/facebook/react/issues/14920), входящее в наш пакет правил линтера [`eslint-plugin-react-hooks`](https://www.npmjs.com/package/eslint-plugin-react-hooks#installation). Оно предупреждает, когда зависимости указаны неправильно и предлагает исправление.

### `useRef` {#useref}

```js
const refContainer = useRef(initialValue);
```

`useRef` возвращает изменяемый ref-объект, свойство `.current` которого инициализируется переданным аргументом (`initialValue`). Возвращенный объект будет сохраняться в течение всего времени жизни компонента.

Обычный случай использования – это доступ к потомку в императивном стиле:

```js
function TextInputWithFocusButton() {
  const inputEl = useRef(null);
  const onButtonClick = () => {
    // `current` указывает на смонтированный элемент `input`
    inputEl.current.focus();
  };
  return (
    <>
      <input ref={inputEl} type="text" />
      <button onClick={onButtonClick}>Установить фокус на поле ввода</button>
    </>
  );
}
```

<<<<<<< HEAD
Обратите внимание, что `useRef()` полезен не только для атрибута `ref`. Он также [удобен для хранения любого изменяемого значения](/docs/hooks-faq.html#is-there-something-like-instance-variables) примерно так же, как вы используете поля экземпляров в классах.
=======
Essentially, `useRef` is like a "box" that can hold a mutable value in its `.current` property.

You might be familiar with refs primarily as a way to [access the DOM](/docs/refs-and-the-dom.html). If you pass a ref object to React with `<div ref={myRef} />`, React will set its `.current` property to the corresponding DOM node whenever that node changes.

However, `useRef()` is useful for more than the `ref` attribute. It's [handy for keeping any mutable value around](/docs/hooks-faq.html#is-there-something-like-instance-variables) similar to how you'd use instance fields in classes.

This works because `useRef()` creates a plain JavaScript object. The only difference between `useRef()` and creating a `{current: ...}` object yourself is that `useRef` will give you the same ref object on every render.

Keep in mind that `useRef` *doesn't* notify you when its content changes. Mutating the `.current` property doesn't cause a re-render. If you want to run some code when React attaches or detaches a ref to a DOM node, you may want to use a [callback ref](/docs/hooks-faq.html#how-can-i-measure-a-dom-node) instead.

>>>>>>> 2304fa1a7c34b719c10cca1023003e22bf0fd137

### `useImperativeHandle` {#useimperativehandle}

```js
useImperativeHandle(ref, createHandle, [deps])
```

`useImperativeHandle` настраивает значение экземпляра, которое предоставляется родительским компонентам при использовании `ref`. Как всегда, в большинстве случаев следует избегать императивного кода, использующего ссылки. `useImperativeHandle` должен использоваться с `forwardRef`:

```js
function FancyInput(props, ref) {
  const inputRef = useRef();
  useImperativeHandle(ref, () => ({
    focus: () => {
      inputRef.current.focus();
    }
  }));
  return <input ref={inputRef} ... />;
}
FancyInput = forwardRef(FancyInput);
```

В этом примере родительский компонент, который отображает `<FancyInput ref={fancyInputRef} />`, сможет вызывать `fancyInputRef.current.focus()`.

### `useLayoutEffect` {#uselayouteffect}

Сигнатура идентична `useEffect`, но этот хук запускается синхронно после всех изменений DOM. Используйте его для чтения макета из DOM и синхронного повторного рендеринга. Обновления, запланированные внутри `useLayoutEffect`, будут полностью применены синхронно перед тем, как браузер получит шанс осуществить отрисовку.

Предпочитайте стандартный `useEffect`, когда это возможно, чтобы избежать блокировки визуальных обновлений.

> Совет
>
<<<<<<< HEAD
> Если вы переносите код из классового компонента, `useLayoutEffect` запускается в той же фазе, что и `componentDidMount` и `componentDidUpdate`, поэтому, если вы не уверены, какой хук эффект использовать, это, вероятно, наименее рискованно.
=======
> If you're migrating code from a class component, note `useLayoutEffect` fires in the same phase as `componentDidMount` and `componentDidUpdate`. However, **we recommend starting with `useEffect` first** and only trying `useLayoutEffect` if that causes a problem.
>
>If you use server rendering, keep in mind that *neither* `useLayoutEffect` nor `useEffect` can run until the JavaScript is downloaded. This is why React warns when a server-rendered component contains `useLayoutEffect`. To fix this, either move that logic to `useEffect` (if it isn't necessary for the first render), or delay showing that component until after the client renders (if the HTML looks broken until `useLayoutEffect` runs).
>
>To exclude a component that needs layout effects from the server-rendered HTML, render it conditionally with `showChild && <Child />` and defer showing it with `useEffect(() => { setShowChild(true); }, [])`. This way, the UI doesn't appear broken before hydration.
>>>>>>> 2304fa1a7c34b719c10cca1023003e22bf0fd137

### `useDebugValue` {#usedebugvalue}

```js
useDebugValue(value)
```

`useDebugValue` может использоваться для отображения метки для пользовательских хуков в React DevTools.

Например, рассмотрим пользовательский хук `useFriendStatus`, описанный в разделе [«Создание собственных хуков»](/docs/hooks-custom.html):

```js{6-8}
function useFriendStatus(friendID) {
  const [isOnline, setIsOnline] = useState(null);

  // ...

  // Показывать ярлык в DevTools рядом с этим хуком
  // например, «FriendStatus: Online»
  useDebugValue(isOnline ? 'Online' : 'Offline');

  return isOnline;
}
```

> Совет
>
> Мы не рекомендуем добавлять значения отладки в каждый пользовательский хук. Это наиболее ценно для пользовательских хуков, которые являются частью общих библиотек.

#### Отложите форматирование значений отладки {#defer-formatting-debug-values}

В некоторых случаях форматирование значения для отображения может быть дорогой операцией. Это также не нужно, если хук не проверен.

По этой причине `useDebugValue` принимает функцию форматирования в качестве необязательного второго параметра. Эта функция вызывается только при проверке хуков. Она получает значение отладки в качестве параметра и должна возвращать форматированное отображаемое значение.

Например, пользовательский хук, который возвратил значение `Date`, может избежать ненужного вызова функции `toDateString`, передав следующую функцию форматирования:

```js
useDebugValue(date, date => date.toDateString());
```

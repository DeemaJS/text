# Руководство Плагинов Babel

В этом документе описано как создавать [плагины](https://babeljs.io/docs/advanced/plugins/) для [Babel](https://babeljs.io).
And you will learn markdown

# Содержание

  * [Введение](#toc-introduction)
  * [Базовые концепции](#toc-basics) 
      * [Абстрактные синтаксические деревья (ASTs)](#toc-asts)
      * [Этапы работы Babel](#toc-stages-of-babel)
      * [Парсинг](#toc-parse) 
          * [Лексический анализ](#toc-lexical-analysis)
          * [Синтаксический анализ](#toc-syntactic-analysis)
      * [Трансформация](#toc-transform)
      * [Генерация](#toc-generate)
      * [Обход](#toc-traversal)
      * [Посетители](#toc-visitors)
      * [Пути](#toc-paths) 
          * [Пути в Посетителях](#toc-paths-in-visitors)
      * [Состояние](#toc-state)
      * [Области видимости](#toc-scopes) 
          * [Привязка контекста](#toc-bindings)
  * [API](#toc-api) 
      * [babylon](#toc-babylon)
      * [babel-traverse](#toc-babel-traverse)
      * [babel-types](#toc-babel-types)
      * [Определения](#toc-definitions)
      * [Строители](#toc-builders)
      * [Валидаторы](#toc-validators)
      * [Преобразователи](#toc-converters)
      * [babel-generator](#toc-babel-generator)
      * [babel-template](#toc-babel-template)
  * [Создание вашего первого плагина Babel](#toc-writing-your-first-babel-plugin)
  * [Операции преобразования](#toc-transformation-operations) 
      * [Посещение](#toc-visiting)
      * [Проверка типа узла](#toc-check-if-a-node-is-a-certain-type)
      * [Проверка, есть ли ссылка на идентификатор](#toc-check-if-an-identifier-is-referenced)
      * [Манипуляция](#toc-manipulation)
      * [Замена узла](#toc-replacing-a-node)
      * [Замена узла несколькими узлами](#toc-replacing-a-node-with-multiple-nodes)
      * [Замена узла исходной строкой](#toc-replacing-a-node-with-a-source-string)
      * [Добавление узла-потомка](#toc-inserting-a-sibling-node)
      * [Удаление узла](#toc-removing-a-node)
      * [Замена родителя](#toc-replacing-a-parent)
      * [Удаление родителя](#toc-removing-a-parent)
      * [Область видимости](#toc-scope)
      * [Проверка, привязана ли локальная переменная](#toc-checking-if-a-local-variable-is-bound)
      * [Создание UID](#toc-generating-a-uid)
      * [Отправка объявления переменной в родительскую область видимости](#toc-pushing-a-variable-declaration-to-a-parent-scope)
      * [Переименование привязки и ссылок на нее](#toc-rename-a-binding-and-its-references)
  * [Параметры плагина](#toc-plugin-options)
  * [Построение узлов](#toc-building-nodes)
  * [Лучшие практики](#toc-best-practices) 
      * [Избегайте обхода AST насколько это возможно](#toc-avoid-traversing-the-ast-as-much-as-possible)
      * [Слияние посетителей, когда это возможно](#toc-merge-visitors-whenever-possible)
      * [Do not traverse when manual lookup will do](#toc-do-not-traverse-when-manual-lookup-will-do)
      * [Оптимизации вложенных посетителей](#toc-optimizing-nested-visitors)
      * [Избегайте вложенных структур](#toc-being-aware-of-nested-structures)

# <a id="toc-introduction"></a>Введение

Babel - это многоцелевой компилятор общего назначения для JavaScript. Более того, это коллекция модулей, которая может быть использована для множества различных форм синтаксического анализа.

> Статический анализ - это процесс анализа кода без запуска этого кода. (Анализ кода во время выполнения известен как динамический анализ). Цели синтаксического анализа очень разнообразны. Это может быть использовано для контроля качества кода (linting), компиляции, подсветки синтаксиса, трансформации, оптимизации, минификации и много другого.

Вы можете использовать Babel для создания множества различных инструментов, которые помогут Вам стать более продуктивным и писать лучшие программы.

> ***Оставайтесь в курсе последних обовлений - подписывайтесь в Твиттере на [@thejameskyle](https://twitter.com/thejameskyle).***

* * *

# <a id="toc-basics"></a>Базовые концепции

Babel - это JavaScript компилятор, точнее компилятор, преобразующий программу на одном языке в программу на другом языке (source-to-source compiler), часто называемый трянслятор. Это означает, что вы даёте Babel некоторый JavaScript код, а Babel модифицирует его, генерирует новый код и возвращает его.

## <a id="toc-asts"></a>Абстрактные синтаксические деревья (ASTs)

Каждый из этих шагов требует создания или работы с [Абстрактным синтаксическим деревом](https://en.wikipedia.org/wiki/Abstract_syntax_tree), или AST.

> Babel использует в качестве AST модифицированный [ESTree](https://github.com/estree/estree) и его спецификация находится [здесь](https://github.com/babel/babel/blob/master/doc/ast/spec.md).

```js
function square(n) {
  return n * n;
}
```

> Взгляните на [AST Explorer](http://astexplorer.net/) чтобы получить более полное представление об AST-нодах. [Здесь](http://astexplorer.net/#/Z1exs6BWMq) находится ссылка на него с уже скопированным примером выше.

Эта же программа может быть представлена в виде подобного списка:

```md
- FunctionDeclaration:
  - id:
    - Identifier:
      - name: square
  - params [1]
    - Identifier
      - name: n
  - body:
    - BlockStatement
      - body [1]
        - ReturnStatement
          - argument
            - BinaryExpression
              - operator: *
              - left
                - Identifier
                  - name: n
              - right
                - Identifier
                  - name: n
```

Или в виде JavaScript-объекта, вроде этого:

```js
{
  type: "FunctionDeclaration",
  id: {
    type: "Identifier",
    name: "square"
  },
  params: [{
    type: "Identifier",
    name: "n"
  }],
  body: {
    type: "BlockStatement",
    body: [{
      type: "ReturnStatement",
      argument: {
        type: "BinaryExpression",
        operator: "*",
        left: {
          type: "Identifier",
          name: "n"
        },
        right: {
          type: "Identifier",
          name: "n"
        }
      }
    }]
  }
}
```

Вы могли заметить, что каждый уровень AST имеет одинаковую структуру:

```js
{
  type: "FunctionDeclaration",
  id: {...},
  params: [...],
  body: {...}
}
```

```js
{
  type: "Identifier",
  name: ...
}
```

```js
{
  type: "BinaryExpression",
  operator: ...,
  left: {...},
  right: {...}
}
```

> Некоторые свойства были убраны для упрощения.

Каждый из этих уровней называется **Нода (Node)**. Отдельный AST может состоять как из одной ноды, так и из сотен, если не тысяч нод. Все вместе они способны описать синтаксис программы, который может быть использован для статического анализа.

Каждая нода имеет следующий интерфейс:

```typescript
interface Node {
  type: string;
}
```

Поле `type` - это строка, описывающая, чем является объект, представляемый данной нодой (т.е. `"FunctionDeclaration"`, `"Identifier"`, или `"BinaryExpression"`). Каждый тип ноды определяет некоторый дополнительный набор полей, описывающий этот конкретный тип.

Пример. Каждая нода, сгенерированная Babel, имеет дополнительные свойства, которые описывают позицию этой ноды в оригинальном исходном коде.

```js
{
  type: ...,
  start: 0,
  end: 38,
  loc: {
    start: {
      line: 1,
      column: 0
    },
    end: {
      line: 3,
      column: 1
    }
  },
  ...
}
```

Эти свойства - `start`, `end`, `loc` - присутствуют в каждой отдельной ноде.

## <a id="toc-stages-of-babel"></a>Этапы работы Babel

Три основных этапа работы Babel это **парсинг**, **трансформация**, **генерация**.

### <a id="toc-parse"></a>Парсинг

Стадия **разбора** принимает код и выводит AST. Существуют два этапа разбора в Babel: [**Лексический анализ**](https://en.wikipedia.org/wiki/Lexical_analysis) и [**Синтаксический анализ**](https://en.wikipedia.org/wiki/Parsing).

#### <a id="toc-lexical-analysis"></a>Лексический анализ

Лексический анализ будет принимать строку кода и превращать его в поток **токенов**.

Вы можете думать о токенах как о плоском массиве элементов синтаксиса языка.

```js
n * n;
```

```js
[
  { type: { ... }, value: "n", start: 0, end: 1, loc: { ... } },
  { type: { ... }, value: "*", start: 2, end: 3, loc: { ... } },
  { type: { ... }, value: "n", start: 4, end: 5, loc: { ... } },
  ...
]
```

Каждый `тип` здесь имеет набор свойств, описывающих токен:

```js
{
  type: {
    label: 'name',
    keyword: undefined,
    beforeExpr: false,
    startsExpr: true,
    rightAssociative: false,
    isLoop: false,
    isAssign: false,
    prefix: false,
    postfix: false,
    binop: null,
    updateContext: null
  },
  ...
}
```

Как узлы AST, они также имеют `start`, `end` и `loc`.

#### <a id="toc-syntactic-analysis"></a>Синтаксический анализ

Синтаксический анализ примет поток токенов и преобразует их в AST представление. Используя информацию в токенах, этот этап переформатирует их как AST, который отображает структуру кода таким образом, что облегчает работу с ним.

### <a id="toc-transform"></a>Трансформация

Этап преобразования принимает AST и проходит через него, доб

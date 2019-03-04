---
title: "Автоматизируем рутину с помощью AST"
date: 2019-02-19T17:45:38+03:00
draft: false
---

Звучит интригующе не правда ли? Под рутиной я имею ввиду такие манипуляции как смена внешнего API у вашей любимой библиотеки (помните скрипт который помог вам переехать на отдельный модуль prop-types в реакт приложениях?), преобразование типов JSDoc в Typescript или flow и остальные подобные манипуляции где вы обычно исправляли всё в куче мест руками.

## Что же такое AST?

Для начала давайте разберемся с тем, что же такое ast. AST(abstract syntax tree) это абстрактное синтаксическое дерево - размеченное древовидное
представление кода, готовое для анализа и преобразования. На основе ast написаны различные инструменты анализа кода таки как eslint, csslint, autoprefixer prettier и т.д.

## Инструменты для работы с AST

[@babel/parser](https://babeljs.io/docs/en/babel-Parser) - парсер кода, на вход берет строку кода, на выходе отдает AST

[@babel/traverse](https://babeljs.io/docs/en/babel-Traverse) - утилита используемая для прохода узлов AST

[@babel/types](https://babeljs.io/docs/en/babel-Types) - набор вспомогательных функций, можно использовать, например,
для создания узлов AST или для их валидации

[@babel/generator](https://babeljs.io/docs/en/babel-generator) - используется для преобразования ast обратно в кода

## Разберем работу с AST на примере
В момент перезда на Typescript появилась необходимость переехать с redux-actions, т.к. он не
имеет собственного тайпинга, а тот, который я нашел оказался не самым лучшим, поэтому логичным
решением было переехать на более приспособленный инструмент для работы с redux actions на TS

## Начнём
Для начала рассмотрим reducer, написанный с помощью redux-actions. Я решил не заморачиваться сильно и использовать давно проверенную схему: брать за основу TODO приложение

```javascript
import { handleActions } from 'redux-actions';

import * as actions from './actions';

const initialState = {
    todos: [],
};

export default handleActions({
    [actions.addTodo](state, { payload }) {
        return {
            ...state,
            todos: [...state.todos, payload],
        };
    },
    [actions.deleteTodo](state, { payload }) {
        return {
            ...state,
            todos: [...state.todos, payload],
        };
    },
}, initialState);
```

Чтобы получить аналогичный код для typesafe-actions нам нужно:

1. Заменить импорт
2. Заменить вызов handleActions на обявление функции с логикой по обработке действий в switch/case

### Заменим импорт

Для совершения манипуляций над узлами ast мы будем пользоваться, упомянутым выше, модулем @babel/traverse, он принимает на вход ast и объект с методами называемыми "визиторы", визитор это метод, название которого должно соответствовать типу ноды которую мы хотим посетить, например в случае с заменой импорта мы должны назвать метод-визитор `ImportDeclaration`. Давайте перейдем непосредственно к коду нашего визитора.

```javascript
traverse(ast, {
    ImportDeclaration(path) {
        // определяем что это именно тот импорт, который нам нужно заменить
        if (path.node.source.value === "redux-actions") {
            path.replaceWith(
                types.importDeclaration(
                    [
                        types.importSpecifier(
                            types.identifier('getType'),
                            types.identifier('getType')
                        ),
                        types.importSpecifier(
                            types.identifier('ActionType'),
                            types.identifier('ActionType')
                        ),
                    ],
                    types.stringLiteral('typesafe-actions')
                )
            );
        }
    }
}
```

Визитор принимает один аргумент - объект со всей информацией о текущей ноде, в контексте @babel/traverse он называется path, а также различные методы для работы с ней, в данном примере мы использем метод `replaceWith`, который позволяет нам заменить текущую ноду на другую. Сама же нода лежит в поле под именем node, остальные поля вы можете увидеть в режиме дебаггера в своей IDE или в документации [здесь](https://github.com/jamiebuilds/babel-handbook/blob/master/translations/en/plugin-handbook.md#paths), подробно расписывать их здесь я не буду.

Итак, вернемся к нашему примеру. Как вы могли заметить, для создания новой ноды мы используем библиотеку @babel/types, в частности такие функции как `importDeclaration` для создания ноды импорта, `importSpecifier` для указания того, что мы будем импортировать и `stringLiteral` для указания имени библиотеки импорта.

Если вы не знаете, как создать нужную вам ноду, вы можете воспользоваться сервисом astexplorer.net написав там нужный вам код и посмотрев результат превращения его в AST.

Есть более короткий способ записать наш пример:

```javascript
traverse(ast, {
    ImportDeclaration(path) {
        // определяем что это именно тот импорт, который нам нужно заменить
        if (path.node.source.value === "redux-actions") {
            path.replaceWithSourceString('import { getType, ActionType } from "typesafe-actions";);
        }
    }
}
```

Выглядит более просто и лаконично, правда? Мы лишь заменили функцию `replaceWith` на `replaceWithSourceString`, которая вместо ноды принимает в себя строку кода. Но в коде библиотеки `@babel/traverse` я нашёл предостережение, которое гласит, что такой способ является антипаттерном и не должен использоваться. Не смотря на это, я решил всё таки показать вам, что такой способ существует.

### Заменим вызов handleActions

Итак, импорт мы заменили, пришло время для второго шага. Напишем визитор `CallExpression`:

```javascript
traverse(ast, {
    // type ноды вызова функции
    CallExpression(path) {
        const node = path.node;

        // Убедимся, что это имеено тот вызов, который нам нужен
        if (node.callee.name === "handleActions") {
            const actionsMap = node.arguments[0].properties;

            const actionHandlers = actionsMap.map(item => ({
                body: item.body,
                action: item.key
            }));

            path.replaceWith(
                types.arrowFunctionExpression(
                    [
                        types.identifier('state'),
                        types.identifier('action')
                    ],
                    types.blockStatement(
                        [
                            types.switchStatement(
                                types.identifier('action'),
                                actionHandlers.map(item => types.switchCase(
                                    types.callExpression(
                                        types.identifier('getType'),
                                        [item.action]
                                    ),
                                    [item.body]
                                ))
                            )
                        ]
                    )
                )
            );
        }
    }
});
```

Здесь мы сперва проходимся по всем узлам вызова функции handleActions, собираем информацию об обработчиках redux-действий, такую как тело обработчика и имя действия для того, чтобы переиспользовать готовые ноды в построении switch/case оператора.

## Получим код из AST

Для этого воспользуемся библиотекой `@babel/generator`, она принимает на вход принимает ast, объект с опциями и исходный код:

```javascript
const result = generator(ast, { retainLines: true }, codeExample);

console.log(result.code); // выведем получившийся код
```

На выходе получим объект с полем code, в котором будет лежать строка с получившимся кодом.
Форматирование кода не на самом высоком уровне, но это можно легко исправить с помощью встроенных средств своей IDE или с помощью prettier.

На этом всё, ниже я прикрепляю репозиторий с исходным кодом, где вы сможете не только посмотреть мой код, но и попрактиковаться, внутри я оставил несколько задачек для практики, have fun!

---
title: Настройка в React/TypeScript приложении ESLint, Prettier и Husky
date: 2022-04-22 19:59:24
tags: [React, CreateReactApp, ESLint, Prettier, Husky]
---

Попросили меня друзья помочь с настройкой ```CreateReactApp``` (*CRA*) приложения, чтобы синтаксис проверялся, красота кода в одном стиле автоматически наводилась, и чтобы не давало закоммитить, если всё вышесказанное не выполняется.

Начал я с настройки *ESLint*. Он нужен для проверки синтаксиса, стиля кодирования и поиска ошибок в коде. По умолчанию *ESLint* уже есть в *CRA*, его нужно только инициализировать через консоль в папке проекта:

``` bash
yarn eslint --init
```

И затем настроить. Установите через консоль следующие плагины в папку проекта:

``` bash
yarn add -D @typescript-eslint/eslint-plugin @typescript-eslint/parser eslint-config-airbnb-typescript eslint-plugin-jest eslint-config-airbnb
```

Заполните файл .eslintrc.json следующими настройками:

``` json
{
    "env": {
        "browser": true,
        "es2022": true,
        "jest": true
    },
    "extends": [
        "airbnb-typescript",
        "airbnb/hooks",
        "eslint:recommended",
        "plugin:react/recommended",
        "plugin:react/jsx-runtime",
        "plugin:@typescript-eslint/recommended",
        "plugin:jest/recommended",
        "plugin:prettier/recommended",
        "plugin:import/recommended",
        "prettier"      
    ],
    "parser": "@typescript-eslint/parser",
    "parserOptions": {
        "ecmaFeatures": {
            "jsx": true
        },
        "ecmaVersion": "latest",
        "sourceType": "module",
        "project": "./tsconfig.json"
    },
    "plugins": [ "react", "@typescript-eslint", "jest" ],
    "rules": {
        "no-alert": "warn",
        "no-console": "warn",
        "linebreak-style": "off",
        "prettier/prettier": [
            "error",
            {
                "endOfLine": "auto"
            },
            { "usePrettierrc": true }
        ],
        "react/prop-types": "off",
        "@typescript-eslint/no-unused-vars": ["warn"]
    },
    "settings": {
        "react": {
          "createClass": "createReactClass",
          "pragma": "React",
          "fragment": "Fragment",
          "version": "detect",
          "flowVersion": "0.53"
        },
        "propWrapperFunctions": [
            "forbidExtraProps",
            {"property": "freeze", "object": "Object"},
            {"property": "myFavoriteWrapper"},
            {"property": "forbidExtraProps", "exact": true}
        ],
        "componentWrapperFunctions": [
            "observer",
            {"property": "styled"},
            {"property": "observer", "object": "Mobx"},
            {"property": "observer", "object": "<pragma>"}
        ],
        "formComponents": [
          "CustomForm",
          {"name": "Form", "formAttribute": "endpoint"}
        ],
        "linkComponents": [
          "Hyperlink",
          {"name": "Link", "linkAttribute": "to"}
        ]
      }
}
```

Из конфига *ESLint* видно, что мы используем *AirBnb*, а также настраиваем сразу и *Prettier*. *Prettier* - это инструмент автоматического форматирования кода. Для его установки запустите в консоли:

``` bash
yarn add -D prettier eslint-config-prettier eslint-plugin-prettier
```

Cоветую создать файл ```.eslintignore``` и поместить туда следующее содержимое:

``` bash
build/*
public/*
docs/*
templates/*
src/react-app-env.d.ts
src/serviceWorker.ts
```

Создайте файл ```.prettierrc``` и поместите туда следующее содержимое:

``` json
{
    "trailingComma": "es5",
    "printWidth": 100,
    "semi": false,
    "singleQuote": true,
    "tabWidth": 2
}
```

Осталось только поправить ```package.json```, внеся туда следующие секции:

``` json
"scripts": {
    .... // предыдущий полезный код скриптов
    "format": "prettier --write src/**/*.{ts,tsx,scss,css,json}",
    "lint": "tsc --noEmit && eslint src/**/*.ts{,x} --color",
    "lint-staged": "lint-staged",
    "prepare": "husky install"
}
"eslintConfig": {
    "extends": [
        "react-app",
        "react-app/jest"
    ]
},
```

Теперь можно проверить что всё впорядке, запустив в консоли следующие команды:

``` bash
yarn lint
yarn format
```

Если вы настроили всё правильно команды отработают без ошибок.

Добавим теперь заключительную вещь: чтобы при попытке коммита в ```git``` запускались наши вышеуказанные скрипты, и можно было закоммитить только код, который эти скрипты успешно проходит.

Для этого установим *Husky*, который будет реагировать на событие попытки коммита, следующей командой из консоли:

``` bash
yarn add -D lint-staged husky
```

Для инициализации *Husky* выполните в консоли команду:

``` bash
yarn husky install
```

Так как у нас версия *Husky* 7, то в соответствующей скрытой папке ```.husky``` необходимо внести следующие изменения в файл ```pre-commit```:

``` bash
#!/bin/sh
. "$(dirname "$0")/_/husky.sh"

yarn lint
yarn format
```

Теперь вам удастся закоммитить ваш код только в случае успешного выполнения скриптов из файла ```pre-commit```.

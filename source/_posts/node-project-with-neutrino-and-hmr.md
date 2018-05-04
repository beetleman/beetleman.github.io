---
title: Node.js project with neutrino and hmr
tags:
  - node
  - neutrino
  - HMR
date: 2018-05-04 22:47:33
---

# Why?

For frontend apps based on [React]/new-foo-framework-of-month, we are used to [HMR] its make development easier. On another hand, we still use nodaemon or manual restart for backend app. This solution work but destroy current state of the application during restart which can make debugging harder and startup of some apps is not a quick process.

# Solution

Luckily [neutrino] project provides the easy way to start [Node.js] project with enabled [HMR], [Jest], and [Babel] enabled by default. In this quick tutorial, I'll show how to setup project with [docker-compose] based development environment.

## create project

```shell
yarn create @neutrinojs/project nodeutrino
```

Answer few questions in order to setup project

- select application

```shell
                          _          _
      _ __    ___  _   _ | |_  _ __ (_) _ __    ___
     | '_ \  / _ \| | | || __|| '__|| || '_ \  / _ \
     | | | ||  __/| |_| || |_ | |   | || | | || (_) |
     |_| |_| \___| \__,_| \__||_|   |_||_| |_| \___/

Welcome to Neutrino! 👋
To help you create your new project, I am going to ask you a few questions.

? 🤔  First up, what would you like to create? (Use arrow keys)
❯ A web or Node.js application
  A library
  Components
```

- select [Node.js]

```
                          _          _
      _ __    ___  _   _ | |_  _ __ (_) _ __    ___
     | '_ \  / _ \| | | || __|| '__|| || '_ \  / _ \
     | | | ||  __/| |_| || |_ | |   | || | | || (_) |
     |_| |_| \___| \__,_| \__||_|   |_||_| |_| \___/

Welcome to Neutrino! 👋
To help you create your new project, I am going to ask you a few questions.

? 🤔  First up, what would you like to create? A web or Node.js application
? 🤔  Next, what kind of application would you like to create?
  React
  Preact
  Vue
❯ Node.js
  Some other web app, e.g. Angular, jQuery, or plain JS
```

- select [Jest]

```
                          _          _
      _ __    ___  _   _ | |_  _ __ (_) _ __    ___
     | '_ \  / _ \| | | || __|| '__|| || '_ \  / _ \
     | | | ||  __/| |_| || |_ | |   | || | | || (_) |
     |_| |_| \___| \__,_| \__||_|   |_||_| |_| \___/

Welcome to Neutrino! 👋
To help you create your new project, I am going to ask you a few questions.

? 🤔  First up, what would you like to create? A web or Node.js application
? 🤔  Next, what kind of application would you like to create? Node.js
? 🤔  Would you like to add a test runner to your project? (Use arrow keys)
❯ Jest
  Mocha
  None
```

- select airbnb [eslint] preset

```
                          _          _
      _ __    ___  _   _ | |_  _ __ (_) _ __    ___
     | '_ \  / _ \| | | || __|| '__|| || '_ \  / _ \
     | | | ||  __/| |_| || |_ | |   | || | | || (_) |
     |_| |_| \___| \__,_| \__||_|   |_||_| |_| \___/

Welcome to Neutrino! 👋
To help you create your new project, I am going to ask you a few questions.

? 🤔  First up, what would you like to create? A web or Node.js application
? 🤔  Next, what kind of application would you like to create? Node.js
? 🤔  Would you like to add a test runner to your project? Jest
? 🤔  Would you like to add linting to your project? (Use arrow keys)
❯ Airbnb style rules
  StandardJS rules
  None
```

## setup docker

As startup project from host machine is acceptable as development environment should we predictable and as close as possible to production. Both of this requirements are satisfied by docker. Save this file as `docker-compose.yml`:

```yml
version: "2.2"

services:
  app:
    image: node:8
    volumes:
      - ./:$PWD
    working_dir: $PWD
    ports:
      - "3000:3000"
    command: yarn start

  tests:
    extends: app
    command: yarn test
```

## run tests

```shell
docker-compose rm --rm tests
```

## run app

```shell
docker-compose up app
```

# The problem

HMR work out of box but if you make some syntaxt error you will need restart aplication

```shell
echo "the bug!" >> src/app.js
```

and you get something like:

```shell
app_1    | [HMR] Cannot apply update.
app_1    | [HMR] Error: Module build failed: SyntaxError: Unexpected token, expected ; (6:4)
app_1    |
app_1    |   4 |
app_1    |   5 | export default app;
app_1    | > 6 | the bug
app_1    |     |     ^
app_1    |   7 |
app_1    |
app_1    |     at Object../src/app.js (/home/beetleman/Projects/nodeutrino/build/0.c485d22b6ea60c2a147d.hot-update.js:8:7)
app_1    |     at __webpack_require__ (/home/beetleman/Projects/nodeutrino/build/index.js:639:30)
app_1    |     at fn (/home/beetleman/Projects/nodeutrino/build/index.js:49:20)
app_1    |     at /home/beetleman/Projects/nodeutrino/build/index.js:855:108
app_1    |     at hotApply (/home/beetleman/Projects/nodeutrino/build/index.js:542:17)
app_1    |     at /home/beetleman/Projects/nodeutrino/build/index.js:250:21
app_1    |     at <anonymous>
app_1    | [HMR] You need to restart the application!
```

It is rly anoying during development but lucky solution for this is quite simple. We need wrap our
`import` statment in `try catch` block. To achieve this goal we create `HMRApp.js`:

```javascript
/* eslint-disable */
let app;
try {
  app = require('./app').default;
} catch (err) {
  console.error(err);
  app = () => err.message;
}
export default app;
```

and then replace `./app` with `./HMRApp` in `index.js`:

```javascript
 import { createServer } from 'http';
import app from './HMRApp';

const port = process.env.PORT || 3000;

createServer((request, response) => response.end(app()))
  .listen(port, () => process.stdout.write(`Running on :${port}\n`));

if (module.hot) {
  module.hot.accept('./HMRApp');
}
```

 after restarting app you should get:

 ```shell
app_1    | yarn run v1.5.1
app_1    | $ neutrino start
app_1    | Error: Module build failed: SyntaxError: Unexpected token, expected ; (6:4)
app_1    |
app_1    |   4 |
app_1    |   5 | export default app;
app_1    | > 6 | the bug
app_1    |     |     ^
app_1    |   7 |
app_1    |
app_1    |     at Object../src/app.js (/home/beetleman/Projects/nodeutrino/build/index.js:848:7)
app_1    |     at __webpack_require__ (/home/beetleman/Projects/nodeutrino/build/index.js:639:30)
app_1    |     at fn (/home/beetleman/Projects/nodeutrino/build/index.js:49:20)
app_1    |     at Object../src/HMRApp.js (/home/beetleman/Projects/nodeutrino/build/index.js:836:9)
app_1    |     at __webpack_require__ (/home/beetleman/Projects/nodeutrino/build/index.js:639:30)
app_1    |     at fn (/home/beetleman/Projects/nodeutrino/build/index.js:49:20)
app_1    |     at Object../src/index.js (/home/beetleman/Projects/nodeutrino/build/index.js:859:66)
app_1    |     at __webpack_require__ (/home/beetleman/Projects/nodeutrino/build/index.js:639:30)
app_1    |     at fn (/home/beetleman/Projects/nodeutrino/build/index.js:49:20)
app_1    |     at Object.0 (/home/beetleman/Projects/nodeutrino/build/index.js:876:1)
app_1    | Running on :3000
 ```

 and now you can remove `the bug` and enjoi HMR!

 ```shell
app_1    |     at __webpack_require__ (/home/beetleman/Projects/nodeutrino/build/index.js:639:30)
app_1    |     at fn (/home/beetleman/Projects/nodeutrino/build/index.js:49:20)
app_1    |     at Object.0 (/home/beetleman/Projects/nodeutrino/build/index.js:876:1)
app_1    | Running on :3000
app_1    | [HMR] Updated modules:
app_1    | [HMR]  - ./src/app.js
app_1    | [HMR]  - ./src/HMRApp.js
app_1    | [HMR] Update applied.
 ```

Of course implementation of `HMRApp.js` depends on how `app.js` is implemented bu basic idea is that if import throw exception replace if with mock of `app` and print error on `strerr`.

Implementation of example app is avialable in github [repo].

Happy hacking ❤️!


[Node.js]: https://nodejs.org/en/
[Jest]: https://facebook.github.io/jest/
[neutrino]: https://neutrino.js.org/
[HMR]: https://webpack.js.org/concepts/hot-module-replacement/
[Babel]: https://babeljs.io/
[React]: https://reactjs.org/
[docker-compose]: https://docs.docker.com/compose/
[eslint]: https://eslint.org/
[repo]: https://github.com/beetleman/nodeutrino

# npm-based front-end development using browserify and browser loader library

On browser side front end development, we can also use npm as our source of modules.
In this article i will introduce a way of using a browser loader library(modulex) to load npm module in browser on development and using browserify to package modules on production.
You may need to see this article first: https://github.com/yiminghe/learning-react/blob/master/tutorial/env/setup-your-playground.md

The demo is at https://github.com/yiminghe/learning-react/tree/master/example/react-router

## dependent tools

* [nodejs](http://nodejs.org/) - server side javascript runtime
* [modulex](https://github.com/kissyteam/modulex) - browser side loader library
* [koa](https://github.com/koajs/koa) - a nodejs web framework
* [koa-jsx](https://www.npmjs.org/package/koa-jsx) - koa middleware for transforming jsx of react
* [koa-modularize](https://www.npmjs.org/package/koa-modularize) - koa middleware for transforming commonjs file into browser module format
* [node-browserify](https://github.com/substack/node-browserify) - browser-side require() the node.js way
* [bower](https://github.com/bower/bower) - A package manager for the web
* [gulp](https://github.com/gulpjs/gulp) - The streaming build system
* [react](https://www.npmjs.org/package/react) - ui library from facebook
* [react-tools](https://www.npmjs.org/package/react-tools) - provide api to transform JSX into vanilla JS
* [modulex-npm](https://github.com/yiminghe/modulex-npm) - generate config file to use npm modules with modulex

## using modulex

modulex is a browser loader library like requirejs.

### create package.json

For example:

```javascript
{
  "name": "learning-react",
  "version": "1.0.0",
  "author": "yiminghe <yiminghe@gmail.com>",
  "engines": {
    "node": ">=0.11"
  },
  "license": "MIT",
  "repository": {
    "type": "git",
    "url": "http://github.com/yiminghe/learning-react.git"
  },
  "keywords": [
    "react",
    "jsx",
    "koa"
  ],
  "devDependencies": {
    "gulp": "^3.8.10",
    "koa": "^0.13.0",
    "koa-jsx": "^1.0.0",
    "koa-modularize": "^1.0.0",
    "koa-mount": "^1.3.0",
    "koa-serve-index": "^1.0.1",
    "koa-static": "^1.4.7",
    "modulex-npm": "^1.0.4",
    "react": "^0.12.0",
    "react-router": "^0.10.2",
    "react-tools": "^0.12.0",
    "reactify": "~0.15.2"
  },
  "scripts": {
    "start": "node --harmony server"
  }
}

```

### create server.js

For example:

```javascript
var koa = require('koa');
var serve = require('koa-static');
var app = koa();
var path = require('path');
var cwd = __dirname;
var serveIndex = require('koa-serve-index');
app.use(serveIndex(cwd, {
    hidden: true,
    view: 'details'
}));
var jsx = require('koa-jsx');
app.use(jsx(cwd, {
    reactTools: require('react-tools'),
    next:function(){
        return 1;
    }
}));
var modularize = require('koa-modularize');
var mount=require('koa-mount');
app.use(mount('/example',modularize(path.resolve(cwd,'example'))));
app.use(mount('/node_modules',modularize(path.resolve(cwd,'node_modules'))));
app.use(serve(cwd, {
    hidden: true
}));
app.listen(8000);
console.log('server start at ' + 8000);
```

pay attention to ```app.use(mount('/node_modules', modularize(path.resolve(cwd,'node_modules'))));``,
we need this statement to transform npm modules for modulex to load in browser.


### create gulpfile.js

We will use gulp as our build system.

```javascript
var gulp = require('gulp');
var fs = require('fs');
var path = require('path');
var cwd = process.cwd();
gulp.task('config', function () {
    // https://github.com/hapijs/qs/pull/58
    var qsIndex = path.resolve(cwd, 'node_modules/react-router/node_modules/qs/index.js');
    if (fs.existsSync(qsIndex)) {
        fs.writeFileSync(qsIndex, "/*modified*/module.exports = require('./lib/');");
    }
    var modulexNpm = require('modulex-npm');
    var config = modulexNpm.generateConfig(['react', 'react-router']);
    fs.writeFileSync(path.join(cwd, 'config.js'), 'require.config(' + JSON.stringify(config) + ');');
});
```

pay attention to ``var config = modulexNpm.generateConfig(['react', 'react-router']);``,
we will generate config file for modulex to load npm packages in browser.

### create application demo

For example example/demo.html:

```html
<!DOCTYPE html>
<html>
<head>
    <script src="/bower_components/modulex/build/modulex-debug.js"></script>
</head>
<body>
<script src="/config.js"></script>
<script>
    window.process = {
        env: {

        }
    };
    require.config('packages', {
        test: {
            base: './'
        }
    });
    require(['test/init']);
</script>
</body>
</html>
```

pay attention to ``<script src="/config.js"></script>``,
we need this statement to load npm modules into browser by modulex.

And example/init.js

```javascript
/** @jsx React.DOM */

var React = require('react');
var Router = require('react-router');
var Route = Router.Route;
var Routes = Router.Routes;
var Redirect = Router.Redirect;
var Link = Router.Link;

var App = React.createClass({
    render: function() {
        return (
            <div>
                <ul>
                    <li><Link to="user" params={{userId: "123"}}>Bob</Link></li>
                    <li><Link to="user" params={{userId: "abc"}}>Sally</Link></li>
                </ul>
                <this.props.activeRouteHandler />
            </div>
            );
    }
});

var User = React.createClass({
    render: function() {
        return (
            <div className="User">
                <h1>User id: {this.props.params.userId}</h1>
                <ul>
                    <li><Link to="task" params={{userId: this.props.params.userId, taskId: "foo"}}>foo task</Link></li>
                    <li><Link to="task" params={{userId: this.props.params.userId, taskId: "bar"}}>bar task</Link></li>
                </ul>
                <this.props.activeRouteHandler />
            </div>
            );
    }
});

var Task = React.createClass({
    render: function() {
        return (
            <div className="Task">
                <h2>User id: {this.props.params.userId}</h2>
                <h3>Task id: {this.props.params.taskId}</h3>
            </div>
            );
    }
});

var routes = (
    <Route handler={App}>
        <Route name="user" path="/user/:userId" handler={User}>
            <Route name="task" path="tasks/:taskId" handler={Task}/>
            <Redirect from="todos/:taskId" to="task"/>
        </Route>
    </Route>
    );

React.renderComponent(
    <Routes children={routes}/>,
    document.body
);
```

## run

Run the following command first
```
bower install modulex  # install a browser loader library
npm install # install npm modules
gulp config
npm start # start local server
```

then open [http://localhost:8000/example/demo.html](http://localhost:8000/example/demo.html) to see the result.


## using browserify

Add bundle task to gulpfile first

```javascript
gulp.task('bundle', function () {
    require('child_process').exec('browserify -t reactify example/init.js > example-bundle/init.js');
});
```

Then create demo-bundle.html to use bundled javascript

```html
<!DOCTYPE html>
<html>
<head>
</head>
<body>
<script src="/example-bundle/init.js"></script>
</body>
</html>
```

## run

Run ``gulp bundle`` first and then open [http://localhost:8000/example/demo-bundle.html](http://localhost:8000/example/demo-bundle.html) to see the result.

### The End

This is it! Congratulation, you can use browser loader library for quick development and use browserify on production.
The last and most important thing: you can also share code between server and client easily.
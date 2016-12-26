# KlarkJS

A package management system based on Plugins and dependency injection for NodeJS applications.

KlarkJS is a novel system for NodeJS module managment that handles the creation of the modules, resolve their internal and external dependencies, and provide them to other modules.

It works with dependency injection based on function parameters in order to define the components dependencies. This architecture decreases dramatically the boiler plate code of an ordinary node js application.

We inspired from 2 main tools:

- Angular 1 dependency injection
- Architect js package management

## Table of Contents

<!-- MarkdownTOC depth=4 autolink=true bracket=round -->

- [Main Idea](#main-idea)
- [Benefits](#benefits)
    - [Pure NodeJS implementation](#pure-nodejs-implementation)
    - [KlarkJS implementation](#klarkjs-implementation)
    - [Comparison](#comparison)
        - [Boilerplate Code](#boilerplate-code)
        - [Reative Path Avoidance](#reative-path-avoidance)
        - [Code Guidance](#code-guidance)
- [API](#api)
    - [run\(\[config\]\)](#runconfig)
        - [config](#config)
    - [KlarkAPI](#klarkapi)
        - [getModule\(name\)](#getmodulename)
        - [getInternalModule\(name\)](#getinternalmodulename)
        - [getExternalModule\(name\)](#getexternalmodulename)
        - [injectInternalModuleFromMetadata\(moduleName, controller\)](#injectinternalmodulefrommetadatamodulename-controller)
        - [injectInternalModuleFromFilepath\(filepath\)](#injectinternalmodulefromfilepathfilepath)
        - [injectExternalModule\(name\)](#injectexternalmodulename)
        - [getApplicationDependenciesGraph\(\)](#getapplicationdependenciesgraph)
- [KlarkJS Development](#klarkjs-development)
- [References](#references)

<!-- /MarkdownTOC -->

## Main Idea

Potentially, all the NodeJS modules are isolated plugins. Each plugin depends on other plugin. The other plugins could either be external (from `node_modules` folder) or internal (another plugin in our source code tree). Thus, we can simply define a plugin providing the plugin name and the dependencies. The following is a typical KlarkJS module declaration:

```javascript
KlarkModule(module, 'myModuleName1', function($nodeModule1, myModuleName2) {
    return {
        log: function() { console.log('Hello from module myModuleName1') }
    };
});
```

`KlarkModule` is a global function provided by KlarkJS in order to register a module. The module registration requires 3 parameters.

1. The NodeJS native module
2. The name of the module
3. The module's controller

The dependencies of the module are defined by the controller parameter names. The names of the modules are always camel case and the external modules (`node_modules`) are always prefixed by `$`.

## Benefits

On the following excerpts we depict a simple NodeJS application in pure NodeJS modules and a NodeJS application in KlarkJS. We will observe the simplicity and the minimalism that KlarkJS provides.

### Pure NodeJS implementation


```javascript
/////////////////////
// ./app.js
var express = require('express');
var config = require('./config');
var mongooseConnector = require('./db/mongoose-connector.js');

var app = express();
app.get('/', function(req, res){
    mongooseConnector.connect(function(db) {
        db.collection('myCollection').find().toArray(function(err, items) {
            res.send(items);
        });
    });
});
app.listen(config.PORT);

/////////////////////
// /db/mongoose-connector.js
var mongoose = require('mongoose');
var logger = require('./logger');
var config = require('./config');

module.exports = {
    connect: connect
};

var connected = false;
function connect(cb) {
    if (connected) {
        cb($mongoose.connection);
        return;
    }
    mongoose.connect(config.MONGODB_URL);]
    db.once('open', function() {
      logger.log('connected');
      connected = true;
      cb(mongoose.connection);
    });
}

/////////////////////
// ./logger.js
module.exports = {
    log: function() { console.log.apply(console, arguments); },
    error: function() { console.error.apply(console, arguments); }
};

/////////////////////
// ./config.js
module.exports = {
    MONGODB_URL: 'mongodb://localhost:27017/my-db',
    POST: 3000
};

/////////////////////
// ./index.js
require('./app');
```

### KlarkJS implementation

```javascript
/////////////////////
// /plugins/app/index.js
KlarkModule(module, 'app', function($express, config, dbMongooseConnector) {
    var app = $express();
    app.get('/', function(req, res){
        dbMongooseConnector.connect(function(db) {
            db.collection('myCollection').find().toArray(function(err, items) {
                res.send(items);
            });
        });
    });
    app.listen(config.PORT);
});

/////////////////////
// /plugins/db/mongoose-connector/index.js
KlarkModule(module, 'dbMongooseConnector', function($mongoose, logger, config) {
    var connected = false;
    return {
        connect: connect
    };

    function connect(cb) {
        if (connected) {
            cb($mongoose.connection);
            return;
        }
        $mongoose.connect(config.MONGODB_URL);]
        db.once('open', function() {
          logger.log('connected');
          connected = true;
          cb($mongoose.connection);
        });
    }
});

/////////////////////
// /plugins/logger/index.js
KlarkModule(module, 'logger', function() {
    return {
        log: function() { console.log.apply(console, arguments); },
        error: function() { console.error.apply(console, arguments); }
    };
});

/////////////////////
// /plugins/config/index.js
KlarkModule(module, 'config', function() {
    return {
        MONGODB_URL: 'mongodb://localhost:27017/my-db',
        POST: 3000
    };
});

/////////////////////
// ./index.js
var klark = require('klark-js');
klark.run();
```

### Comparison

#### Boilerplate Code

```javascript
// pure NodeJS
var express = require('express');
var app = express(); ...

// Kark JS
KlarkModule(module, 'app', function($express) {
```

In pure NodeJS version, when we want to use an external dependency, we have to repeat the dependency name 3 times.

1. `require('express');`
2. `var express`
3. `express();`

In KlarkJS version, we define the dependency only once, as the parameter of the function.

#### Reative Path Avoidance

```javascript
// pure NodeJS
var mongooseConnector = require('./db/mongoose-connector.js');

// Kark JS
KlarkModule(module, 'app', function(dbMongooseConnector) { ...
```

In pure NodeJS version, we define the internal dependencies using the relative path from the current file location to the dependency's file location. This pattern generates an organization issue. When our source files increases, and we want to change the location of a file, we have to track all the files that depends on the this file, and change the relative path.
In KlarkJS version, we refer on the file with a unique name. The location of the file inside the source tree does not effect the inclusion process.

#### Code Guidance

The module registration function `KlarkModule` forces the programmer to follow a standard pattern for the module definition. For instance, we always know that the dependencies of the module are defined on the controller's parameters. In pure NodeJS the dependencies of the module can be written in many different ways.

## API

```javascript
var klark = require('klark-js');
klark.run();
```

### run([config])

Starts the Klark engine. Briefly, it loads the files, creates the modules dependencies, instantiates the modules.

* `config` `{Object}`, (optionally)
* return `Promise<KlarkAPI>`

#### config

* `predicateFilePicker`, `Function -> Array<string>`. A function that returns the file path patterns that the modules are located. Default:
```javascript
predicateFilePicker: function() {
  return [
    'plugins/**/index.js',
    'plugins/**/*.module.js'
  ];
}
```
* `globalRegistrationModuleName`, `String`. The name of the global function that registers the KlarkJS modules. Default: `KlarkModule`.
* `globalConfigurationName`, `String`. The name of the global object that holds the KlarkJS configuration. Default: `KlarkConfig`.
* `base`, 'String'. The root location of the application. The `predicateFilePicker` search for files under the `base` folder. Default: `process.cwd()`
* `logLevel`, `String`. The verbose level of KlarkJS logging. When we want to debug the module loading process, we can yield the logger on `high` and observe the sequence of loading. It is enumerated by:
    - high
    - middle
    - low
    - off (Default)
* `moduleAlias`, `Object<String, String>`. Alias name for the modules. If you provide an alias name for a module, you can either refer to it with the original name, or the alias name. For instance, we will could take a look on the Default object:
```javascript
moduleAlias: {
  '_': 'lodash'
}
```
When we define the alias name we can either refer on external library lodash either using the name `'lodash'`
```
KlarkModule(module, '..', function(lodash) {
```
or `'_'`
```
KlarkModule(module, '..', function(_) {
```

### KlarkAPI

#### getModule(name)

Searches on the internal and the external modules.

* `name` `String`. The name of the module.
* return: the instance of the module.

#### getInternalModule(name)

Searches on the internal modules.

* `name` `String`. The name of the module.
* return: the instance of the module.

#### getExternalModule(name)

Searches on the external modules.

* `name` `String`. The name of the module.
* return: the instance of the module.

#### injectInternalModuleFromMetadata(moduleName, controller)

Creates and inserts in the internal dependency chain a new module with the name `moduleName` and the controller `controller`.

* `moduleName` `String`. The name of the module.
* `controller` `Function`. The controller of the module.
* return: `Promise<ModuleInstance>`

#### injectInternalModuleFromFilepath(filepath)

Creates and inserts in the internal dependency chain a new module from the file on `filepath`. The content of the file file should follow the KlarkJS module registration pattern.

* `filepath` `String`. The filepath of the file.
* return: `Promise<ModuleInstance>`

#### injectExternalModule(name)

Creates and inserts in the external dependency chain a new module with the name `moduleName`. This module should already exists in the external dependencies (*nome_modules*)

* `name` `String`. The name of the module.
* return: the instance of the module.

#### getApplicationDependenciesGraph()

Returns the internal and external dependencies of the application's modules in a graph form.

* return:
```javascript
{
    innerModule1: [
        {
            isExternal: true,
            name: 'lodash'
        }
    ],
    innerModule2: [
        {
            isExternal: false,
            name: 'innerModule1'
        }
    ]
}
```

## KlarkJS Development

* `npm test`. Runs the tests located on `/test` folder.
* `npm run coverage`. Runs the test coverage tool and extraxts the results on `/coverage` folder.

## References

* [Architect JS](https://github.com/c9/architect)
* [Angular Dependency Injection](https://docs.angularjs.org/guide/di)
* [Wiki Dependency Injection](https://en.wikipedia.org/wiki/Dependency_injection)

# Marionette.Application.module (В процессе перевода)

Marionette.js позволяет определять модули внутри вашего приложения, 
которые также могут включать в себя подмодули. Это дает нам возможность
создавать модульные приложения, в которых каждый модуль вынесен в 
отдельный файл и инкапсулирует в себе какой-то законченный функционал.

Marionette's modules allow you to have unlimited sub-modules hanging off of
your application, and serve as an event aggregator in themselves.

## Содержание

* [Basic Usage](#basic-usage)
* [Starting And Stopping Modules](#starting-and-stopping-modules)
  * [Starting Modules](#starting-modules)
  * [Start Events](#start-events)
  * [Preventing Auto-Start Of Modules](#preventing-auto-start-of-modules)
  * [Starting Sub-Modules With Parent](#starting-sub-modules-with-parent)
  * [Stopping Modules](#stopping-modules)
  * [Stop Events](#stop-events)
* [Defining Sub-Modules With . Notation](#defining-sub-modules-with--notation)
* [Module Definitions](#module-definitions)
  * [Module Initializers](#module-initializers)
  * [Module Finalizers](#module-finalizers)
  * [Initialize function](#initialize-function)
* [The Module's `this` Argument](#the-modules-this-argument)
* [Custom Arguments](#custom-arguments)
* [Splitting A Module Definition Apart](#splitting-a-module-definition-apart)
* [Extending Modules](#extending-modules)

## Основное применение

Модуль определяется из главного объекта приложения (`Application`) с указанием своего имени:

```js
var MyApp = new Backbone.Marionette.Application();

var myModule = MyApp.module("MyModule");

MyApp.MyModule; // => a new Marionette.Application object

myModule === MyApp.MyModule; // => true
```

Модули должны иметь уникальные имена. Если при создании модуля вы укажете уже 
существующее имя, то новый модуль не создастся, а вам вернется ссылка на существующий модуль.

## Запуск и остановка модулей

Модули могут быть запущены и остановлены независимо от приложения и друг друга. Это дает нам возможность запускать их асинхронно и останавливать, если дальнейшая их работа нам не требуется.

Это также облегчает тестирование модулей, так как мы можем изолированно запустить только требуемый модуль.

### Запуск модулей

Запуск модулей осуществляется одним из двух способов:

1. Автоматически с вызовом метода `.start()` родительского модуля или всего приложения
2. Вручную вызовом метода `.start()` самого модуля

В примере ниже модуль будет запущен автоматически со стартом объекта приложения `MyApp.start()`:

```js
MyApp = new Backbone.Marionette.Application();

MyApp.module("Foo", function(){

  // module code goes here

});

MyApp.start();
```

Помните, что модули загруженные и определенне после `app.start()` тоже могут быть запущены автоматически.

### События запуска модулей

When starting a module, a "before:start" event will be triggered prior
to any of the initializers being run. A "start" event will then be 
triggered after they have been run.

```js
var mod = MyApp.module("MyMod");

mod.on("before:start", function(){
  // do stuff before the module is started
});

mod.on("start", function(){
  // do stuff after the module has been started
});
```

#### Передача данных в события запуска модуля

`.start` takes a single `options` parameter that will be passed to start events and their equivalent methods (`onStart` and `onBeforeStart`.) 

```js
var mod = MyApp.module("MyMod");

mod.on("before:start", function(options){
  // do stuff before the module is started
});

mod.on("start", function(options){
  // do stuff after the module has been started
});

var options = {
 // any data
};
mod.start(options);
```

### Отмена автоматического запуска модулей

If you wish to manually start a module instead of having the application
start it, you can tell the module definition not to start with the parent:

```js
var fooModule = MyApp.module("Foo", function(){

  // prevent starting with parent
  this.startWithParent = false;

  // ... module code goes here
});

// start the app without starting the module
MyApp.start();

// later, start the module
fooModule.start();
```

Note the use of a function instead of just an object literal to define
the module, and the presence of the `startWithParent` attribute, to tell it
not to start with the application. Then to start the module, the module's 
`start` method is manually called.

You can also grab a reference to the module at a later point in time, to
start it:

```js
MyApp.module("Foo", function(){
  this.startWithParent = false;
});

// start the module by getting a reference to it first
MyApp.module("Foo").start();
```

#### Specifying `startWithParent: false` setting as an object literal

There is a second way of specifying `startWithParent` in a `.module`
call, using an object literal:

```js
var fooModule = MyApp.module("Foo", { startWithParent: false });
```

This is most useful when defining a module across multiple files and
using a single definition to specify the `startWithParent` option.

If you wish to combine the `startWithParent` object literal
with a module definition, you can include a `define` attribute on
the object literal, set to the module function:

```js
var fooModule = MyApp.module("Foo", {
  startWithParent: false,

  define: function(){
    // module code goes here
  }
});
```

### Запуск подмодулей вместе с родительским модулем

Starting of sub-modules is done in a depth-first hierarchy traversal. 
That is, a hierarchy of `Foo.Bar.Baz` will start `Baz` first, then `Bar`,
and finally `Foo.

Подмодули по умолчанию запускаются с запуском их родительского модуля. 

```js
MyApp.module("Foo", function(){...});
MyApp.module("Foo.Bar", function(){...});

MyApp.start();
```

In this example, the "Foo.Bar" module will be started with the call to
`MyApp.start()` because the parent module, "Foo" is set to start
with the app.

A sub-module can override this behavior by setting its `startWithParent`
to false. This prevents it from being started by the parent's `start` call.

```js
MyApp.module("Foo", function(){...});

MyApp.module("Foo.Bar", function(){
  this.startWithParent = false;
})

MyApp.start(); 
```

Now the module "Foo" will be started, but the sub-module "Foo.Bar" will
not be started.

A sub-module can still be started manually, with this configuration:

```js
MyApp.module("Foo.Bar").start();
```

### Остановка модулей

A module can be stopped, or shut down, to clear memory and resources when
the module is no longer needed. Like starting of modules, stopping is done
in a depth-first hierarchy traversal. That is, a hierarchy of modules like
`Foo.Bar.Baz` will stop `Baz` first, then `Bar`, and finally `Foo`.

To stop a module and its children, call the `stop()` method of a module.

```js
MyApp.module("Foo").stop();
```

Modules are not automatically stopped by the application. If you wish to 
stop one, you must call the `stop` method on it. The exception to this is
that stopping a parent module will stop all of its sub-modules.

```js
MyApp.module("Foo.Bar.Baz");

MyApp.module("Foo").stop();
```

This call to `stop` causes the `Bar` and `Baz` modules to both be stopped
as they are sub-modules of `Foo`. For more information on defining
sub-modules, see the section "Defining Sub-Modules With . Notation".

### События остановки

When stopping a module, a "before:stop" event will be triggered prior
to any of the finalizers being run. A "stop" event will then be triggered
after they have been run.

```js
var mod = MyApp.module("MyMod");

mod.on("before:stop", function(){
  // do stuff before the module is stopped
});

mod.on("stop", function(){
  // do stuff after the module has been stopped
});
```

## Defining Sub-Modules With . Notation

Sub-modules or child modules can be defined as a hierarchy of modules and 
sub-modules all at once:

```js
MyApp.module("Parent.Child.GrandChild");

MyApp.Parent; // => a valid module object
MyApp.Parent.Child; // => a valid module object
MyApp.Parent.Child.GrandChild; // => a valid module object
```

When defining sub-modules using the dot-notation, the 
parent modules do not need to exist. They will be created
for you if they don't exist. If they do exist, though, the
existing module will be used instead of creating a new one.

## Module Definitions

You can specify a callback function to provide a definition
for the module. Module definitions are invoked immediately
on calling `module` method. 

The module definition callback will receive 6 parameters:

* The module itself
* The Parent module, or Application object that `.module` was called from
* Backbone
* Backbone.Marionette
* jQuery
* Underscore
* Any custom arguments

You can add functions and data directly to your module to make
them publicly accessible. You can also add private functions
and data by using locally scoped variables.

```js
MyApp.module("MyModule", function(MyModule, MyApp, Backbone, Marionette, $, _){

  // Private Data And Functions
  // --------------------------

  var myData = "this is private data";
 
  var myFunction = function(){
    console.log(myData);
  }


  // Public Data And Functions
  // -------------------------

  MyModule.someData = "public data";

  MyModule.someFunction = function(){
    console.log(MyModule.someData);
  }
});

console.log(MyApp.MyModule.someData); //=> public data
MyApp.MyModule.someFunction(); //=> public data
```

### Module Initializers

Modules have initializers, similarly to `Application` objects. A module's
initializers are run when the module is started.

```js
MyApp.module("Foo", function(Foo){

  Foo.addInitializer(function(){
    // initialize and start the module's running code, here.
  });

});
```

Any way of starting this module will cause its initializers to run. You
can have as many initializers for a module as you wish.

### Module Finalizers

Modules also have finalizers that are run when a module is stopped.

```js
MyApp.module("Foo", function(Foo){

  Foo.addFinalizer(function(){
    // tear down, shut down and clean up the module, here
  });

});
```

Calling the `stop` method on the module will run all that module's 
finalizers. A module can have as many finalizers as you wish.


### Initialize Function

Modules have an `initialize` function which is distinct from its collection of initializers. `initialize` is immediately called when the Module is invoked, unlike initializers, which are only called once the module is started. You can think of the `initialize` function as an extension of the constructor.

```js
MyApp.module("Foo", {
  startWithParent: false,
  initialize: function( options ) {
    // This code is immediately executed
    this.someProperty = 'someValue';
  },
  define: function() {
    // This code is not executed until the Module has started
    console.log( this.someProperty ); // Logs 'someValue' once the module is started
  }
});
```

The `initialize` function is passed a single argument, which is the object literal definition of the module itself. This allows you to pass arbitrary values to it.

```js
MyApp.module("Foo", {
  initialize: function( options ) {
    console.log( options.someVar ); // Logs 'someString'
  },
  someVar: 'someString'
});
```

## The Module's `this` Argument

The module's `this` argument is set to the module itself.

```js
MyApp.module("Foo", function(Foo){
  this === Foo; //=> true
});
```

## Custom Arguments

You can provide any number of custom arguments to your module, after the
module definition function. This will allow you to import 3rd party
libraries, and other resources that you want to have locally scoped to
your module.

```js
MyApp.module("MyModule", function(MyModule, MyApp, Backbone, Marionette, $, _, Lib1, Lib2, LibEtc){

  // Lib1 === LibraryNumber1;
  // Lib2 === LibraryNumber2;
  // LibEtc === LibraryNumberEtc;

}, LibraryNumber1, LibraryNumber2, LibraryNumberEtc);
```

## Splitting A Module Definition Apart

Sometimes a module gets to be too long for a single file. In
this case, you can split a module definition across multiple
files:

```js
MyApp.module("MyModule", function(MyModule){
  MyModule.definition1 = true;
});

MyApp.module("MyModule", function(MyModule){
  MyModule.definition2 = true;
});

MyApp.MyModule.definition1; //=> true
MyApp.MyModule.definition2; //=> true
```

## Extending Modules

Modules can be extended. This allows you to make custom Modules to include in your Application.

```
var CustomModule = Marionette.Module.extend({
  constructor: function() {
    // Configure your module
  }
});
```

When attaching a Module to your Application you can specify what class to use with the parameter `moduleClass`.

```
MyApp.module("Foo", {
  moduleClass: CustomModule,
  define: function() {} // You can still use the definition function on custom modules
});
```

When `moduleClass` is omitted, Marionette will default to instantiating a new `Marionette.Module`.
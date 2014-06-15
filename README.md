duck-angular
============

Guides to use this are at:

* [Part One](http://avishek.net/blog/?p=1202)
* [Part Two](http://avishek.net/blog/?p=1188)
* [Part Three](http://avishek.net/blog/?p=1225)
* [Part Four](http://avishek.net/blog/?p=1239)
* [Part Five: Testing Directives](http://avishek.net/blog/?p=1489)
* [A Quick Recap](http://avishek.net/blog/?p=1472)

An example AngularJS app using RequireJS and Duck-Angular is at https://github.com/asengupta/AngularJS-RequireJS-Seed in two combinations:

* [Mocha + RequireJS](https://github.com/asengupta/AngularJS-RequireJS-Seed/tree/master)
* [Jasmine + RequireJS](https://github.com/asengupta/AngularJS-RequireJS-Seed/tree/karma-jasmine)

An example which does not use RequireJS as part of the app is available at: [Angular-Toy](https://github.com/kylehodgson/angular-toy).

duck-angular is a container for bootstrapping and testing AngularJS views and controllers in memory: no browser or external process needed.

Setup
------

duck-angular is available as a Bower package. Install it using 'bower install duck-angular'.

If you intend to set up Duck manually in an environment where RequireJS is not available, you'll need to make sure that the following libraries are available to Duck.

* q.js
* require.js
* text.js
* underscore.js

If you are using RequireJS in your app, Duck will detect it and attempt to load "angular", "underscore", and "Q".

Your controller/service/object initialisation scripts need to have run before you use Duck-Angular. Put them in script tags, or load them using a script loader like RequireJS or Inject.
If you're not using RequireJS in your app, see the example at: [Angular-Toy](https://github.com/kylehodgson/angular-toy). 
Here is an example taken from the [AngularJS-RequireJS Seed app](https://github.com/asengupta/AngularJS-RequireJS-Seed)

    // Using Mocha-as-Promised in this example
    it("can reflect data that is refreshed asynchronously", function () {
      return mother.createMvc("route2Controller", "../templates/route2.html", {}).then(function (mvc) {
        var dom = new DuckDOM(mvc.view, mvc.scope);
        var interaction = new UIInteraction(dom);
        expect(dom.element("#data")[0].innerText).to.eql("Some Data");
        dom.interactWith("#changeLink");
        expect(dom.element("#data")[0].innerText).to.eql("Some New Data");
        return interaction.with("#refreshLink").waitFor(mvc.scope, "refreshData").then(function() {
          expect(dom.element("#data")[0].innerText).to.eql("Some Data");
        });
      });
    });


If using RequireJS, including duck-angular as a dependency will expose duckCtor as a parameter you use in your tests.
If including duck-angular using script tags, window.duckCtor will be available to you.

Initialise the application container, like so:

    var duckFactory = duckCtor(_, angular, Q, $);
    var builder = duckFactory.ContainerBuilder;
    var container = builder.withDependencies(appLevelDependencies).build("MyModuleName", myModule, { baseUrl: "baseUrl/for/Duck/dependencies", textPluginPath: "path/to/text.js"});

The withDependencies(...) call is optional, unless you want to inject some dependency which the controller does not use directly.

##ContainerBuilder API

###withDependencies()

This method allows you to specify module-level dependencies, i.e., dependencies which will be overridden for the entire module. The dependencies are specified as simple key-value pairs, with the key reflecting the actual name of the Angular dependency. If the value is an object, it will be specified configured in Angular's DI via a provider. If the value is a function, it will be executed with two parameters, $provide and the module. This lets the developer override the dependency in whatever fashion is most appropriate. The function returns the **builder** object, so it can be chained, until **build()** or **buildRaw()** is called.

###cacheTemplate()

This is specifically to prevent template load errors when we specify templateUrl values for directives. This preloads the templateUrl into Angular's template cache. Note that this returns a promise. You will need to wait for the promise to be fulfilled, either right after the point of the call, or before the start of the test. Here's an example of how you could do this:

    var setup = function(appLevelDependencies) {
      return buildContainer(appLevelDependencies).then(function (container) {
        return container.domMvc("ControllerName", "path/to/view", controllerDependencies)
      });
    };
     
    var buildContainer = function (appLevelDependencies) {
      var builder = duckFactory.ContainerBuilder;
      return builder.withDependencies(appLevelDependencies).
          cacheTemplate(moduleUnderTest, "declared/path/to/directive/template", "actual/path/to/template").
          then(function () {
            return builder.build("Cinnamon", cinnamon,
                {baseUrl: "/base", textPluginPath: "src/javascript_tests/lib/text"});
          });
    };

Note that it is entirely possible for the declared **templateUrl** to be the same as the path to access it; however, it may be different if you're using a test runner like Karma, which could serve static assets from a different path. This also allows a cheap form of URL rewriting if the path to your template does not match the path it actually is served from, like in Karma.


###build()

This method will construct and return the Container. It takes in 3 parameters:
* Module name: The module name will be the module under test.
* Module object: This is the actual module object that will be bootstrapped.
* Path options: This option is only required when the application is not using RequireJS. Because Duck-Angular uses the text plugin to load resources like views, it needs to know the path to the text plugin. This is where you specify both the baseUrl, and the path to the text plugin, like so:

    var container = builder.withDependencies(appLevelDependencies).build("MyModuleName", myModule, { baseUrl: "baseUrl/for/Duck/dependencies", textPluginPath: "path/to/text.js"});

This method returns a Container object, whose API is discussed below.
The dependencies are injected using an overriding module which is constructed dynamically. This preserves the original module's dependencies. If you wish to bootstrap the original module directly, please use **buildRaw()**.

###buildRaw()

This method is exactly like **build()**, except that it bootstraps the original module directly, instead of using a mock module to override dependencies. Please note that using this method instead of **build()** implies that any app-level dependencies can potentially override all dependencies for all tests, unless application initialisation scripts are run for every new scenario. Alternatively, you may restore the original dependencies manually after every scenario.

##Container API

###mvc()

This method sets up a controller and a view, with dependencies that you can inject. Any dependencies not overridden are fulfilled using the application's default dependencies. It returns an object which contains the controller, the view, and the scope.

    var options = { 
      preBindHook: function(scope) {...}, // optional
      preRenderHook: function(injector, scope) {...}, // optional
      dontWait: false, // optional
      async: false, // optional
      controllerLoadedPromise: function(controller) {...} // optional, required if async is true
    };

    return container.mvc(controllerName, viewUrl, dependencies, options).then(function(mvc) {
      var controller = mvc.controller;
      var view = mvc.view;
      var scope = mvc.scope;
      ...
    });

    // preBindHook and preRenderHook are optional.
    // dontWait is optional, and has a default value of false. Set it to true, if you do not want to wait for nested ng-include partial resolution.
    // async is optional, and has a default value of false. Set it to true, if your controller has to run asynchronous code to finish initialising. If asynchronous initialisation happens, Duck expects your controller to expose a promise whose fulfilment signals completion of controller setup.
    // controllerLoadedPromise is required if async is true. If not provided in this situation, it will assume the controller exposes promise called loaded.

###controller()


This method sets up only a controller without a view, with dependencies that you can inject. Any dependencies not overridden are fulfilled using the application's default dependencies. It returns the constructed controller.

    return controller(controllerName, dependencies, isAsync, controllerLoadedPromise).then(function(controller) {
      ...
    });

    // isAsync is optional, and has a default value of false. Set it to true, if your controller has to run asynchronous code to finish initialising. If asynchronous initialisation happens, Duck expects your controller to expose a promise whose fulfilment signals completion of controller setup.
    // controllerLoadedPromise is required if isAsync is true. If not provided in this situation, it will assume the controller exposes promise called loaded.

###domMvc()
This is a convenience wrapper over the mvc() method. It also constructs a new DuckDOM object (discussed in the DuckDOM Interaction API), and returns both the DuckDOM object, and the MVC object, in that order.

If you're using this method, remember to use spread() on the promise, instead of then() to spread the return value over the argument list, like so:

    return container.domMvc(controllerName, viewUrl, dependencies, options).spread(function(dom, mvc) {
      ...
    };


###get()

This method lets you retrieve any wired Angular dependency by name, like so:

    container.get("$http")

##Interaction API

The DuckDOM/DuckUIInteraction API lets you interact with elements in your constructed view. This only makes sense when you've set up your context using the Container.mvc() method.

###element()

This lets you access any element inside the view using standard jQuery selectors/semantics.

    var DuckDOM = duckFactory.DOM;

    return container.mvc(controllerName, viewUrl, dependencies, options).then(function(mvc) {
      var controller = mvc.controller;
      var view = mvc.view;
      var scope = mvc.scope;
      var dom = new DuckDOM(mvc.view, mvc.scope);
      expect(dom.element("#someElement").isHidden()).to.eq(true);
    });

###apply()

This lets you call Angular's $scope.$apply() method in a safe fashion.

###interactWith()

This lets you interact with elements whose controller behaviour is known to be synchronous. Note that $scope.$apply() is automatically invoked after each interaction, so there is no need to call it yourself.

    var DuckDOM = duckFactory.DOM;

    return container.mvc(controllerName, viewUrl, dependencies, options).then(function(mvc) {
      var controller = mvc.controller;
      var view = mvc.view;
      var scope = mvc.scope;
      var dom = new DuckDOM(mvc.view, mvc.scope);

      dom.interactWith("#emailAddress", "mojo@mojo.com");
      expect(dom.element("#emailAddress").val()).to.eq("mojo@mojo.com");
    });

The interactWith() method is 'overloaded' to understand what type of element you are interacting with, so you can simply pass the second parameter where appropriate. For example:

    dom.interactWith("#someButton");
    dom.interactWith("#someDropdown", 2);
    dom.interactWith("#textField", "Some Text");
    dom.interactWith("#someRadio", true);

###with().waitFor()

This call lets you interact with elements whose controller behaviour is known to be asynchronous. In such cases, you want to wait for the asynchronous behaviour to complete before proceeding with test assertions. This method assumes that the asynchronous logic returns a promise whose fulfilment indicates the completion of the user action.

    var DuckDOM = duckFactory.DOM;
    var UIInteraction = Duck.UIInteraction;

    return container.mvc(controllerName, viewUrl, dependencies, options).then(function(mvc) {
      var controller = mvc.controller;
      var view = mvc.view;
      var scope = mvc.scope;
      var dom = new DuckDOM(mvc.view, mvc.scope);
      var interaction = new UIInteraction(dom);
        return interaction.with("#refreshLink").waitFor(mvc.scope, "refreshData").then(function() {
          expect(dom.element("#data")[0].innerText).to.eql("Some Data");
        });
    });

The above example assumes that there is a method refreshData() present on the scope which returns a promise to indicate completion of the asynchronous code. The rest of the assertions will only continue after this promise as been fulfilled.


License
----------

The MIT License (MIT)

Copyright (c) 2013 Avishek Sen Gupta

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in
all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN
THE SOFTWARE.

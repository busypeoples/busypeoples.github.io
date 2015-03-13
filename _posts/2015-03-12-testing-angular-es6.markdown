---
layout: post
title:  "Implementing An AngularJS Test Workflow With ES6"
date:   2015-03-13 12:00:00
tags:   AngularJS, JavaScript, ES6, Karma, Jasmine
---

### Introduction

This is a follow up to the initial post [Implementing A Test Workflow With ES6](http://busypeoples.github.io/post/testing-workflow-with-es6).
The following should be a quick guide for setting up an **AngularJS** testing workflow with **ES6**.
Both, the actual application code as well as the tests are written in ES6.

###Installation
We will only focus on getting the tests up and running not on the complete development cycle, there are a large number of online resources on how to set up Gulp to build, minify and so on and so forth.
The minimal set up requires **gulp**, **karma**, **karma-jasmine** the **karma-phantomjs-launcher**, browserify as well as babelify for the transformation. All dependencies can be downloaded via npm. 

This is an example config file for the setup (which also includes dependencies used in the development process):


```javascript
{
  "dependencies": {
    "angular": "^1.3"
  },
  "devDependencies": {
    "angular-mocks": "^1.3.14",
    "babel": "^4.0.2",
    "babelify": "^5.0.3",
    "browserify": "~8.1.3",
    "gulp": "^3.8.11",
    "gulp-add-src": "^0.2.0",
    "gulp-concat": "^2.5.2",
    "gulp-sourcemaps": "~1.3.0",
    "gulp-uglify": "~1.1.0",
    "gulp-util": "^3.0.3",
    "jasmine-core": "~2.2.0",
    "karma-babel-preprocessor": "~4.0.0",
    "karma-browserify": "^4.0.0",
    "karma-jasmine": "~0.3.5",
    "karma-phantomjs-launcher": "^0.1.4",
    "stringify": "^3.1.0",
    "vinyl-buffer": "~1.0.0",
    "vinyl-source-stream": "~1.0.0"
  }
}
```

Running **npm install** will download the required resources.

###Configuration
The karma configuration has to be adapted. We define the files that should be preprocessed 
with browserify and configure browserify to transform via babelify. Another important aspect in our current setup is that 
we use **stringify** to require templates inside directives, which means we also have to include stringify in the browser transform
configuration.

```javascript

module.exports = function(config) {
   module.exports = function(config) {
       config.set({
   
           basePath: '',
           frameworks: ['browserify', 'jasmine'],
   
           files: [
               'node_modules/angular/angular.min.js',
               'node_modules/angular-mocks/angular-mocks.js',
               'Module.js',
               'views/*',
               'src/**/*.js',
               'test/**/*test.js'
           ],
   
           exclude: [
           ],
   
           preprocessors: {
               'Module.js': ['browserify'],
               'views/*' : ['browserify'],
               'src/**/*.js': ['browserify'],
               'test/**/*test.js': ['browserify']
           },
   
           browserify: {
               debug: true,
               transform: ['babelify', 'stringify']
           },

        // define reporters, port, logLevel, browsers etc.
    });
};
```


###Example

Everything should be set and ready after installing all required node modules and defining the Karma configuration.

To keep things simple we'll write a basic Angular directive with an external template that is imported at the beginning of the file.


```javascript

import helloWorldTemplate from './../views/hello-world.html';

class HelloWorldController {
    constructor() {
        this.greet = 'Hello';
    }
}

function HelloWorldDirective() {
    return {
        scope: {
            name: '@'
        },
        controller: HelloWorldController,
        controllerAs: 'ctrl',
        bindToController: true,
        replace: true,
        template: helloWorldTemplate
    }
}

export default HelloWorldDirective;

```

The template file will consist of a div that renders a greeting and a name.

```html
<div> {% raw %} {{ ctrl.greet }} {{ ctrl.name }} {% endraw %} </div>
```

The corresponding test will simply assert that a given string exists when calling the directive.

```javascript
describe('The HelloWorldDirective', () => {

    let element, scope;

    beforeEach(angular.mock.module('app'));

    beforeEach(inject(function(_$rootScope_,_$compile_) {
        let $rootScope = _$rootScope_,
            $compile = _$compile_;

        scope = $rootScope.$new();

        element = angular.element('<hello-world data-name="{{ name }}"></hello-world>');

        $compile(element)(scope);

    }));

    it('should display the defined name', () => {
        let name = 'Some rendered text';

        scope.name = name;
        scope.$digest();

        expect(element.text()).toContain(`Hello ${name}`);
    });
});
```

The **karma start** command will run the tests. 
Adding a gulp file to trigger the tests or enable Karma to generate code coverage should be straight forward from here on.

Also check out the [AngularJS-ES6-Test-Skeleton Gist](https://gist.github.com/busypeoples/e4ec7e7c1f1a753050dd).


###Links

[AngularJS-ES6-Test-Skeleton Gist](https://gist.github.com/busypeoples/e4ec7e7c1f1a753050dd)

[Implementing A Test Workflow With ES6](http://busypeoples.github.io/post/testing-workflow-with-es6)

[Karma](http://karma-runner.github.io/0.12/index.html)

[Jasmine](http://jasmine.github.io/)

[Browserify](http://browserify.org/)

[karma-browserify](https://github.com/Nikku/karma-browserify)

[Babel](http://babeljs.io/)


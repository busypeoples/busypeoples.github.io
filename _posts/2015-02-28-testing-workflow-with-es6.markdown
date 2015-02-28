---
layout: post
title:  "Implementing A Test Workflow With ES6"
date:   2015-02-28 12:00:00
tags:   JavaScript, ES6, Karma, Jasmine
---

### Introduction

The introduction of tools and libraries like **traceur**, **browserify** and a large number of competing and/or complemeting tools made writing ES6 code the de facto standard within the last 12 months.

The following write up should highlight a possible approach to tackle the problems that come with setting up a proper testing workflow, no matter if it is test driven or not.

It should be noted that this is just one of a large number of possible solutions. Also the setup is highly influenced by the current development setup, which includes **Gulp**, **Broweserify**, **6to5** a.k.a. Babel and the requirement to write the tests with the **Karma/Jasmine** combo.

So if your current setup includes traceur or **jspm** things will vary obviously, but there are large number of preprocessors if you are using karma for example. 
In case your current setup for writing ES6 code is based on browserify and 6to5/babel and you want to write your tests in ES6 than the following might work out of the box.

###Goal
What we needed in one of our latest projects was the ability to write the tests in ES6. The tests should be triggered via a simple gulp command or alternatively with karma start and the autowatch flag set to true.


###Installation
The following will only focus on getting the tests up and running not on the complete development cycle, there are a large number of online resources on how to set up Gulp to build, minify and so on and so forth.
    The minimal set up requires **gulp**, **karma**, **karma-jasmine** the **karma-phantomjs-launcher**, browserify as well as babelify for the transformation. All dependencies can be downloaded via npm.
    This is the config file for the minimal setup:

```javascript
{
    "devDependencies": {
        "gulp": "~3.8.11",
        "karma": "~0.12.31",
        "browserify": "^9.0.3",
        "babelify": "~5.0.3",
        "karma-browserify": "~3.0.2",
        "karma-jasmine": "~0.3.5",
        "karma-phantomjs-launcher": "~0.1.4"
    }
}
```

Running npm **install** will download the required resources.

###Configuration
The karma configuration has to be slightly adapted, including defining the files to be preprocessed with browserify and configuring browserify to transform via babelify.

```javascript

module.exports = function(config) {
    config.set({

        basePath: '',
        frameworks: ['browserify', 'jasmine'],

        files: [
            'src/**/*.js',
            'test/**/*.js'
        ],

        exclude: [
        ],

        preprocessors: {
            'src/**/*.js': ['browserify'],
            'test/**/*.js': ['browserify']
        },

        browserify: {
            debug: true,
            transform: [ 'babelify' ]
        },

        // define reporters, port, logLevel, browsers etc.
    });
};
```


###Example

Karma can already run against the ES6 test code, to verify we can quickly write a ES6 class:

```javascript

class Foo {

    doSomething() {
        return 'Do Something';
    }
};

export default Foo;
```

and the corresponding test:

```javascript

import Foo from '../src/foo.js';

describe('ES6 Foo', function () {

    let foo;

    beforeEach(()=>{
        foo = new Foo();
    });
    
    it('Foo should create new instance', ()=>{
        expect(foo.doSomething()).toEqual('Do Something');
    });
});
```

The **karma start** command will run the tests. 
Extending this set up from there on should be clear, including adding gulp tasks to trigger tests or adding test coverage.

###Links

[Karma](http://karma-runner.github.io/0.12/index.html)

[Jasmine](http://jasmine.github.io/)

[Browserify](http://browserify.org/)

[karma-browserify](https://github.com/Nikku/karma-browserify)

[Babel](http://babeljs.io/)


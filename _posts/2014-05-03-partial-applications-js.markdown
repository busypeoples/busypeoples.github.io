---
layout: post
title:  "Partial Application And Currying In Javascript"
date:   2014-05-03 10:00:00
tags:	javascript functional
published: true
---

## Introduction

The following article should highlight the difference between partial application and currying.
Sometimes there is a confusion between the two, as currying could be used to for partial application.

## Partial Application
In short a partial function application expects a function and returns the same function but with fewer arguments, as those have been bound to the function.
Take a look at the following example:


```javascript
function sumUpTheThree(a, b, c) {
	return a + b + c;
}
```

What if we only had variable a defined and wanted to bind the value to the function? This is how the most basic solution would look like (f.e. we know that a = 1):


```javascript
function sumUpTheThreeMinusA(b, c) {
    return sumUpTheThree(1, b, c);
}

sumUpTheThreeMinusA(2, 3);
```

We could even simply use [Function.prototype.bind()](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function/bind) method and would have the same result:


```javascript
var sumUpTheThreeMinusA = sumUpTheThree.bind(null, 1);
sumUpTheThreeMinusA(2, 3);
```

The second approach is of course more effective than the first, _bind()_ returns a new function with the provided arguments being set when calling the function.
This would also work:

```javascript
var sumUpTheThreeMinusA = sumUpTheThree.bind(null, 1, 2);
sumUpTheThreeMinusA(3);
```

Passing more arguments will have no effect on the result:

```javascript
var sumUpTheThreeMinusA = sumUpTheThree.bind(null, 1, 2);
sumUpTheThreeMinusA(3, 4, 5, 6);
```


### Custom Function

We should have a basic idea of partial application now. But we can take it a step further and write our own simple function that will take care of returning a function with predefined values:

```javascript
function partialApplication(fn) {
    var args = Array.prototype.slice.call(arguments, 1);

    return function() {
      return fn.apply(this, args.concat(Array.prototype.slice.call(arguments, 0)))
    };
}
```

We will use the previous example for verifying that this custom function will do what it should:

```javascript
var sumUpTheThreeMinusA = partialApplication(sumUpTheThree, 1);
sumUpTheThreeMinusA(2, 3);

var sumUpTheThreeMinusA = partialApplication(sumUpTheThree, 1, 2);
sumUpTheThreeMinusA(3);
```


### Partial Application With Wu.js

Of course we do not need to write our own partial application function as a couple of libraries already offer abstractions including [**wu.js**](http://fitzgen.github.io/wu.js) and lodash.
To round up the partial application part we will have a look at how simple it is to use wu.js to create a partial application.

```
var sumUpTheThreeMinusA = wu.partial(sumUpTheThree, 1);
sumUpTheThreeMinusA(2, 3);

var sumUpTheThreeMinusA = wu.partial(sumUpTheThree, 1, 2);
sumUpTheThreeMinusA(3);
```

We can even do this (we know a and c but we do not have b):

```javascript
var sumUpTheThreeMinusA = wu.partial(sumUpTheThree, 1, wu.___, 3);
sumUpTheThreeMinusA(2);
```


## Currying

Currying essentially solves the problem of dealing with functions that can only accept a single parameter and creating a multiple arguments functions via composition.

Take a look at the following:

```javascript
function sumUpTheThree(a) {
	return function(b) {
		return function(c) {
			return a + b + c;
		}
	}
}
```

and then:

```javascript
sumUpTheThree(1)(2)(3);
```

In javascript we simply wrote this:

```javascript
function sumUpTheThreeMinusA(b, c) {
    return sumUpTheThree(1, b, c);
}
```

But we can use currying to even a create partial application.
This might make sense in some situations.

### Custom Function

We will write a custom function that will return a function until all parameters have been defined.

```javascript
function currying(fn) {
    len = fn.length;

    function curryFn(prev) {
      return function(args) {
        args = prev.concat(args);
        return (args.length < len)? curryFn(args) : fn.apply(this, args);
      };
    }

    return curryFn([]);
}
```

Let's test the custom method:

```javascript
var curryTest = currying(sumUpTheThree);
curryTest(1)(2)(3);
```

Which could also be written like this:

```javascript
var curryTest = currying(sumUpTheThree);
var curryTestB = curryTest(1);
var curryTestC = curryTestB(2);
curryTestC(3);
```

### Currying With Wu.js

Finally we will have a look at how it is implemented in wu.js. Of course a couple of libraries have implemented currying.
Wu.js offers the _autoCurry()_ method that automatically adds currying to a given function, which is more or less what we want.


```javascript
var curryTest = wu.autoCurry(sumUpTheThree);
curryTest(1)(2)(3);

curryTest(1)()()(2)()(3);

curryTest(1)()()(2)()()(3);

curryTest(1)(2, 3);

curryTest(1, 2, 3);
```

All of the above will work.

## Round Up

This was a basic introduction into partial application and currying in javascript.
If you are looking for a library for lazy, functional programming in Javascript checkout [wu.js](http://fitzgen.github.io/wu.js).

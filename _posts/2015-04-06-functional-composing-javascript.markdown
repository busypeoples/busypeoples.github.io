---
layout: post
title:  "Why The Hipsters Compose Everything: Functional Composing In JavaScript"
date:   2015-04-06 12:00:00
tags:  Javascript
published: true
---

### Introduction
**Lodash** and **Underscore** are everywhere and still there is this one supper efficient method that actually only those hipsters use: **compose**.

We will look deeper into compose and a long the way learn the ins and outs of why this method 
will make your code more readable, more maintainable and a lot more elegant.

### The Basics

We will be using a lot of lodash functions, just be cause a.) we don't want to write our own low level implementations as they would distract from what we're trying to focus on and b.) lodash is widely used and could easily be replaced with underscore or other libraries or your own implementation if you need to.

Before we dive into some basic examples, let us recap what compose actually does and how we could implement our own compose function if needed to.

```javascript
var compose = function(f, g) {
    return function(x) {
        return f(g(x));
    };
};
```

This is the most basic implementation. Taking a closer look at the function above, you will notice that the passed in functions are actually applied **right to left**.
Meaning that the result of the right function is passed on to the function left of it. 

Now take a closer look at this code:

```javascript
function reverseAndUpper(str) {
  var reversed = reverse(str);
  return upperCase(reversed);
}
```

A function reverseAndUpper that first reverses the given string and then upper cases it. 
We can rewrite the following code with the help of our basic compose function:

```javascript
var reverseAndUpper = compose(upperCase, reverse);
```

Now we can use reverseAndUpper as follows:

```javascript
reverseAndUpper('test'); // TSET
```

This is equivalent to writing: 

```javascript
function reverseAndUpper(str) {
  return upperCase(reverse(str));
}
```

only more elegant, maintainable and reusable.


The ability to quickly compose functions and create a data pipeline can be leveraged in a number of ways enabling data transformation a long the pipeline. Imagine passing in a collection, mapping over the collection and then returning a max value at the end of the pipeline or turning a given string into a boolean. 
Compose enables us to easily connect a number of functions together to build more complex functions.

Let us implement a more flexible compose function which can handle any number of functions as well as any number of arguments. 
The previous compose only works with two functions and only accepts the first passed in argument. We can rewrite the compose function as follows:

```javascript
var compose = function() {
  var funcs = Array.protoype.slice.call(arguments);

  return funcs.reduce(function(f,g) {
    return function() {
      return f(g.apply(this, arguments));
    };
  });
};
```
 
This compose enables to write something like this:

```javascript
Var doSometing = compose(upperCase, reverse, doSomethingInitial);

doSomething('foo', 'bar');
```

A large number of libraries exist that implement compose for us. 
Our compose function should only help in understanding what's really happening inside underscores or scoreunders compose function 
and obviously the concrete implementations vary between the libraries. 
They still do the same thing mostly, making their respective compose functions interchangeable. 
Now that we have an idea about what compose is all about, let us use the lodash _.compose function for the next set of examples.

### Examples
Let's get started with a very basic example:

```javascript
function notEmpty(str) {
	return ! _.isEmpty(str);
}
```

The _notEmpty_ function simply negates the result of _.isEmpty.

We can achieve the same by using lodash's _.compose and writing a not function:

```javascript
function not(x) { return !x; }

var notEmpty = _.compose(not, _.isEmpty);
```

We can call notEmpty on any given argument now;

```javascript
notEmpty('foo'); // true
notEmpty(''); // false
notEmpty(); // false
notEmpty(null); // false
```

This first example is super simple, then next one is more advanced:

_findMaxForCollection_ returns the max id for a given collection of objects consisting of id and value properties.

```javascript
function findMaxForCollection(data) {
	var items = _.pluck(data, 'val');
	return Math.max.apply(null, items);
}

var data = [{id: 1, val: 5}, {id: 2, val: 6}, {id: 3, val: 2}];

findMaxForCollection(data);
```

The above example could be rewritten with the compose function:

```javascript
var findMaxForCollection = _.compose(function(xs) { return Math.max.apply(null, xs); }, _.pluck);

var data = [{id: 1, val: 5}, {id: 2, val: 6}, {id: 3, val: 2}];

findMaxForCollection(data, 'val'); // 6
```

There is a lot we can refactor here.

_.pluck expects the collection as the first and the callback function as the second argument. What if we could partially apply _.pluck?
We can use currying to flip the arguments in this case. 

```javascript
function pluck(key) {
	return function(collection) {
		return _.pluck(collection, key);
	}
}
```

Our _findMaxForCollection_ still needs more refinement, we could create our own max function:

```javascript
function max(xs) {
	return Math.max.apply(null, xs);
}
```

This enables us to rewrite the compose function to something more elegant:

```javascript
var findMaxForCollection = _.compose(max, pluck('val'));

findMaxForCollection(data);
```

By writing our own pluck function we are able to partially apply pluck with 'val'. Now you could 
obviously argue, why even write your own pluck method, when lodash already has that handy _.pluck function. 
The problem here is that _.pluck expects the collection as the first argument, which is what we don't want. 
By flipping the arguments around we were able to partially apply the key first, only needing to call the data on the returned function. 

We can still further rewrite the pluck method.
Lodash comes with a handy method *_.curry*, which enables us to write our pluck function as follows:

```javascript
function plucked(key, collection) {
	return _.pluck(collection, key);
}

var pluck = _.curry(plucked);
```

We simply want to wrap around the original pluck function just so we can flip the arguments.
Now pluck simply keeps returning a function, so long, until all the arguments have been supplied.
Let's take a look at the final code:

```javascript
function max(xs) {
	return Math.max.apply(null, xs);
}

function plucked(key, collection) {
	return _.pluck(collection, key);
}

var pluck = _.curry(plucked);

var findMaxForCollection = _.compose(max, pluck('val'));

var data = [{id: 1, val: 5}, {id: 2, val: 6}, {id: 3, val: 2}];

findMaxForCollection(data); // 6
```

The _findMaxForCollection_ can easily be read from right to left, meaning for each item in the collection return the val property and then get the max for all given values. 

```javascript
var findMaxForCollection = _.compose(max, pluck('val'));
```

This leads to more maintainable, reusable and especially elegant code. 
We will take a look at one last example just to highlight the elegance of functional composing.

Let's extend the data from the previous example and let's add another property named active. 
The data is as follows now:

```javascript
var data = [{id: 1, val: 5, active: true}, 
			{id: 2, val: 6, active: false }, 
			{id: 3, val: 2, active: true }];
```

We have this function called _getMaxIdForActiveItems(data)_ that expects a collection of objects, filters all active items and returns the max id from those filtered items.

```javascript
function getMaxIdForActiveItems(data) {
	var filtered = _.filter(data, function(item) {
		return item.active === true;
	});

	var items = _.pluck(filtered, 'val');
	return Math.max.apply(null, items);
}
```

What if we could transform the above code into something more elegant?
We already have the max and pluck functions in place, so all we need to do now is add the filtering:

```javascript
var getMaxIdForActiveItems = _.compose(max, pluck('val'), _.filter);

getMaxIdForActiveItems(data, function(item) {return item.active === true; }); // 5
```

*_.filter* has the same problem that also _.pluck had, meaning we can not partially apply it as the first argument is the collection. 
We can flip the arguments on filter, by wrapping the native filter implementation:

```javascript
function filter(fn) {
	return function(arr) {
		return arr.filter(fn);
	};
}
```

Another refinement is to add an _isActive_ function that simply expects an item and checks if the active flag is set to true.

```javascript
function isActive(item) {
	return item.active === true;
}
```

We can partially apply _filter_ with _isActive_, enabling us to call getMaxIdForActiveItems only with the data. 

```javascript
var getMaxIdForActiveItems = _.compose(max, pluck('val'), filter(isActive));
```

Now all we need to pass in is the data:

```javascript
getMaxIdForActiveItems(data); // 5
```

This also enables us to easily compose a function that returns the max id for any non active items:

```javascript
var isNotActive = _.compose(not, isActive);

var getMaxIdForNonActiveItems = _.compose(max, pluck('val'), filter(isNotActive));
```

### Roundup

Functional composing can be a lot of fun as seen in the previous examples plus leads to more elegant and reusable code if applied correctly. 

### Links

[lodash](https://lodash.com/)

[Hey Underscore, You're Doing It Wrong!](https://www.youtube.com/watch?v=m3svKOdZijA)

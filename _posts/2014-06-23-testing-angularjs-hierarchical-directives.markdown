---
layout: post
title:  "Testing AngularJS hierarchical Directives with Jasmine"
date:   2014-06-23 12:00:00
tags:   AngularJS, Javascript, Unit Tests, Karma, Jasmine
published: false
---

###The Basics

Based on the previous [Testing AngularJS Directives with Jasmine](http://busypeoples.github.io/post/testing-angularjs-directives) post we will extend the _collection_ directive.
The _collection_ directive renders a set of rows based on a given set.
In this example the existing directive will call another directive, the _item_ directive for every data object.
Before we continue, you might have a look at the [original code](http://plnkr.co/edit/c3YAZRfGju7IWlOnoBrH).

###Testing hierarchical Directives

Take a look at the adapted collection example, we are calling the item directive on every ng-repeat _<item data-item="item"></item>_.

```javascript
Module.directive('collection', function () {
	return {
		restrict: 'EA',
		transclude: true,
		replace: true,
		scope: {
			items: '=items'
		},
		controller: 'collectionController',
		template: '<div class="table table">' +
			'<div ng-transclude></div>' +
			'<div ng-repeat="item in items">' +
			'<item data-item="item"></item>' +
			'</div>' +
			'<div><button ng-click="addNewItem()">Add new Item</button></div>' +
			'</div>'
	};
});
```

The _item_ directive is very minimal, all it does it render the item itself.

```javascript
Module.directive('item', function () {
	return {
		require: '^collection',
		restrict: 'E',
		scope: {
			item: '='
		},
		replace: true,
		link: function ($scope, $element, $attrs, $tabsCtrl) {
			$scope.selectItem = $tabsCtrl.selectItem;
		},
		template:   '<div class="item-class" ng-class="{active: item.selected}">{{ item.title }} - id: {{ item.id }}' +
					' <button class="btn" ng-disabled="item.selected" ng-click="selectItem(item)">Set Active</button>' +
					'</div>'
	};
});
```

But what if we do not want to test both directives? Can we test the _item_ directive isolated from the collection directive?

The best approach is probably to mock the low level directive with $compileProvider.

Setting the directive to a very high priority and terminal to true will ensure that the mocked directive will be called while the real directive will never be executed.

```javascript
beforeEach(module('example', function($compileProvider) {
	$compileProvider.directive('item', function() {
	  var fake = {
	    priority: 100,
	    terminal: true,
	    restrict: 'E',
	    template: '<div class="fake">Not the real thing.</div>',
	  };
	  return fake;
	});
}));
```

Testing the directive:

```javascript
it('should create clickable items', function() {
	var items = elm.find('.fake');
	expect(items.length).toBe(2);
});
```

Other options include mocking low level directives via a directive factory
or overriding via higher priority and terminal true like in the following example
(which is probably the worst approach and has absolutely no advantage compared to the first approach)

```javascript
beforeEach(function() {
	angular.module('example').directive('item', function() {
		return {
			 priority: 100,
			terminal: true,
			template : '<div class="fake">Not the real thing.</div>'
		}
	});
});
```

###How to mock another directive's controller
In our example the _item_ directive depends on the _collection_ controller

```javascript
Module.directive('item', function () {
	return {
		require: '^collection',
		restrict: 'E',
		// etc...
```

There a couple of possible approaches here:

One way is to wrap the _collection_ directive around the _item_directive when calling $compile:

```javascript
beforeEach(inject(function($rootScope, $compile) {
	elm = angular.element(
	  '<div>' +
	  '<item data-item="dataset"></item>' +
	  '</div>');

	scope = $rootScope.$new();

	var collectionController = {
	  setItem: function() {}
	};

	scope.dataset = {id:1, title : 'testing', selected : false};
	elm.data('$collectionController', collectionController);

	$compile(elm)(scope);
	scope.$digest();
}));

it('should create one div item', function() {
	var items = elm.find('.item-class');
	console.log(elm);
	expect(items.text()).toContain('testing - id: 1');
});
```

###Links
[AngularJS Directives Documentation](https://docs.angularjs.org/guide/directive)

[Jasmine](http://jasmine.github.io/)

[Karma](http://karma-runner.github.io/0.12/index.html)

[ngDirective Testing Example](https://github.com/vojtajina/ng-directive-testing/)

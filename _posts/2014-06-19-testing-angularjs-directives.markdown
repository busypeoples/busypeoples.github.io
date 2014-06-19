---
layout: post
title:  "Testing AngularJS Directives with Jasmine"
date:   2014-06-19 12:00:00
tags:   AngularJS, Javascript, Karma, Jasmine
---

##The Basics
This post is simply a reminder on how to test **AngularJS** directives. There seems to be no problem testing a service or a controller in AngularJS,
but testing directives might appear a little more complex.

We are actually dealing with minor complexities here, as we need to do a little more than when writing unit tests for controllers or services.
We need to render the template, which can be easily achieved with the help of $compile and we need to verify that elements exists or are rendered the way they should be.

I will break this section into multiple parts, just to cover different possibilities and problems that you might have to face when testing directives.
The following topics should be covered in this post: dealing with a single directive + controller and directives with external
templates. A follow up post will concentrate on  hierarchical directives and testing strategies regarding directive dependencies.

Testing a reusable directive should be a high priority, just to make sure that extending or changing the implementation will not break
the current status. Sometimes a directive will be adapted a long the way just to fit a special case that pops up. Instead of having to create a new directive,
one might simply add a new attribute or extend existing functions to handle this new special case. This is not uncommon.

###Testing Directives - the standard

Before we can write any tests, let us look at a basic example for a directive. The _Collection_ directive will render a collection and display every item
as a row. The example is based on [Testing directive – the easy way](https://github.com/vojtajina/ng-directive-testing) by **Vojta Jina**, who also is part of the AngularJS team.
You might have a look at the test set up, in case you need a primer into setting up the configuration with Karma and Jasmine. The example also relies on jQuery, due to
the fact that we want to easily access dom elements and verify their existence or content. We could also use AngularJS own jQLite, but it is rather easier to go with aforementioned approach.

Well the view script is really trivial, we defined a collection element and added some custom text or message.

```html
<div ng-app="example">
    <div ng-controller="exampleController">
        <collection data-items="items">
           <b>Welcome. This is some custom message text.</b>
        </collection>
    </div>
</div>
```

The _exampleController_ in this case simply holds a collection of items. This can be neglected when testing, because we will never interact with
any external controllers, we will mock these items when testing the directives.

```javascript
Module.controller('exampleController', ['$scope', function ($scope) {
		$scope.items = [
			{id: 1, title: 'title a'},
			{id: 2, title: 'title b'},
			{id: 3, title: 'title c'},
			{id: 4, title: 'title d'}
		];
	}]);
```

This is all we need to actually be able to call the collection directive with a suitable data set.
The directive code is as follows:

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
		// templateUrl: 'tpl/tabs.html',
		template: '<div class="collection">' +
			'<div ng-transclude></div>' +
			'<div ng-repeat="item in items">' +
			'<div class="item-class" ng-class="{active: item.selected}">{{ item.title }} - id: {{ item.id }}' +
			' <button class="btn" ng-disabled="item.selected" ng-click="selectItem(item)">Set Active</button>' +
			'</div>' +
			'</div>' +
			'<div><button ng-click="addNewItem()">Add new Item</button></div>' +
			'</div>'
	};
});
```

The template in the basic implementation is really inlined into the javascript file. We use ngTransclude to add some custom text and we are
be able to test this part of the code very easily, as we can access predefined ids and the sort (as seen in an upcoming example). We also have a collectionController that offers a couple of
basic methods like setting the currently selected item as active or adding new items.

```javascript
Module.controller('collectionController', ['$scope', function ($scope) {
	var items = $scope.items || [];
	var that = this;

	init();

	function init() {
		if ($scope.items.length > 0) $scope.items[0].selected = true;
	}

	$scope.selectItem = this.selectItem = function (item) {
		angular.forEach(items, function (item) {
			item.selected = false;
		});
		item.selected = true;
	};

	$scope.addNewItem = function () {
		var i = {id: items.length, title: 'Added new item with id : ' + items.length};
		that.selectItem(i);
		items.push(i);
	};
}]);
```

This is really it, we have a directive that will simply render a given set of items into a set of divs.

Testing this directive with current conditions:

+ We have no templateUrl defined.

+ We are defining the html inline.

+ We also have no directives calling other directives.


Let us have a look at setting up the tests.
We can use _$compile_ to render the html and _$digest_ for updating the template with the new data.
This all we really need.


```javascript
describe('Collection Directive', function () {
	var elm, scope;

	beforeEach(module('example'));

	beforeEach(inject(function ($rootScope, $compile) {
		elm = angular.element(
			'<div>' +
				'{{ customMessage }}' +
				'<collection-basic data-items="items">' +
				'</collection-basic>' +
				'</div>');

		scope = $rootScope.$new();

		scope.customMessage = '<div class="custom-message">f u</div>';
		scope.items = [{id:1, title:'title a'}, {id:2, title:'title b'}];
		$compile(elm)(scope);
		scope.$digest();
	}));
```

Also added the helper matcher for Vojta Jina for quickly testing if a given class exists. You can find the code for the [ng-directive-testing on github](https://github.com/vojtajina/ng-directive-testing/blob/start/test/helpers.js).

```javascript
// https://github.com/vojtajina/ng-directive-testing/blob/start/test/helpers.js
beforeEach(function() {
    this.addMatchers({
        toHaveClass: function(cls) {
            this.message = function() {
                return "Expected '" + angular.mock.dump(this.actual) + "' to have class '" + cls + "'.";
            };

            return this.actual.hasClass(cls);
        }
    });
});
```

Our first test asserts that all div elements have been created and that the first two elements really contain the correct text.
There is not too much to explain here, it is really that straight forward.

```javascript
it('should create  items', function () {
	var items = elm.find('.item-class');
	expect(items.length).toBe(2);

	// could be defined into a test for itself
	// avoid multiple asserts - this is only for demonstration
	expect(items.eq(0).text()).toContain('title a');
	expect(items.eq(1).text()).toContain('title b');
});
```

We can also test if the first element has been set to active when initially loading the directive.

```javascript
it('should set active class on first item', function () {
	scope.$digest();
	var items = elm.find('div.item-class');

	expect(items.eq(0)).toHaveClass('active');
	expect(items.eq(1)).not.toHaveClass('active');
});
```
And we can even test if a click on the "Set Active" button will really set the current item to active.

```javascript
it('should change active item when edit button is clicked', function () {
	var items = elm.find('.item-class');

	items.eq(1).find('button').click();

	expect(items.eq(0)).not.toHaveClass('active');
	expect(items.eq(1)).toHaveClass('active');
});
```

We could also add tests the verify that custom message has been rendered, tests that verify we can add a new item and so on.

###Testing Directives that use templateUrl
By using the templateUrl option inside a directive, we can avoid having to inline html into our javascript code.
This is really helpful when the template simply does more than warp a property inside a div tag.

Our previous collection directive now uses templateUrl to load the html:

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
		templateUrl: 'tpl/collection.html'
	};
});
```

With the basic karma/jasmine setup you will run into a problems, because the templates will never get loaded instead you have to deal the following error:
 _Error: Unexpected request: GET tpl/collection.html_

There is a very simple solution to the aforementioned problem: karma's ng-html2js preprocessor which will enable Karma to automatically generates the js file and adds
the html into $templateCache, which can also be done by hand if needed.

To install run the following command.

```
npm install karma-ng-html2js-preprocessor --save-dev
```

It is also required to adapt the karma configuration (in case you use karma with Jasmine), for more details
see [Testing AngularJS directive templates with Jasmine and Karma](http://daginge.com/technology/2013/12/14/testing-angular-templates-with-jasmine-and-karma/)

```javascript
preprocessors: {
	'path/to/templates/*.html': ['ng-html2js']
},

// we will be accessing this by module name later on in Jasmine
ngHtml2JsPreprocessor: {
	moduleName: 'templates'
},

// list of files / patterns to load in the browser
files: [
	'path/to/templates/*.html'
],
```

We need to include the module before running the tests with _beforeEach(module('templates'))_ , where 'templates' is defined via configuration
(you could name it 'foo' or whatever you like).


```javascript
describe('Collection Directive', function () {
	var elm, scope, fooBar;

	beforeEach(module('example'));

	// 'templates' can be anything, is defined via karma config.
	beforeEach(module('templates'));
```

By adding _karma-ng-html2js-preprocessor_, updating the karma configuration and including the module before each test inside our Jasmine tests
we are able to test the directives that rely on templateUrl.

###Roundup
This post should be an introduction into testing AngularJS directives with Jasmine.
There will be a follow up post concentrating on testing hierarchical directives.

###Links
[AngularJS Directives Documentation](https://docs.angularjs.org/guide/directive)

[Jasmine](http://jasmine.github.io/)

[Karma](http://karma-runner.github.io/0.12/index.html)

[ngDirective Testing Example](https://github.com/vojtajina/ng-directive-testing/)

[Testing AngularJS directive templates with Jasmine and Karma](http://daginge.com/technology/2013/12/14/testing-angular-templates-with-jasmine-and-karma/)
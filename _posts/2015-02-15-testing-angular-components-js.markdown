---
layout: post
title:  "Testing Components in AngularJS"
date:   2015-02-15 12:00:00
tags:   AngularJS, JavaScript, Components, Karma, Jasmine
published: true
---

### Introduction

There seems be a trend towards component based thinking in AngularJS. 
Especially rethinking the ng-controller/scope/view approach has been the basis for a lot of articles, post, talks and discussions
in the past couple of months.

This post is based on the previous [Component Based Thinking in AngularJS](http://busypeoples.github.io/post/thinking-in-components-angular-js)
and is quasi a follow up where the focus is solely on testing. You can find the original code **[here](http://plnkr.co/edit/mWwL7mCZlf3dvU44PefD?p=preview)**.

The following section will assume prior knowledge on testing in general and **Jasmine** and **Karma** in specific and will not go into 
detail on how to install or setup Karma and Jasmine. More information on setting up the testing environment can be found [here](http://karma-runner.github.io/0.12/intro/installation.html) for example.


### Testing the \<item\> Component

The original example consisted of an **\<itemsContainer\>** that composed **\<itemsList\>** and **\<searchBox\>** components.
Both the \<itemsList\> and the \<searchBox\> component also composed **\<item\>** components.

The low level item directive can easily be tested as we don't have any controller logic to consider, all the component does is
receive an object with a name and an activity status. Based on the given item object it renders a title consisting of the item name as well as a checkbox which is either checked or unchecked
depending on the _active_ property.

So the first step is to create the test skeleton. By using the _beforeEach_ function we can import the _app_ module, create a new _scope_ and
_compile the element_ with the proper scope. This has the benefit of only needing to define those steps once instead 
of having to redefine the setup in every single test. See the code below for clarification.

```javascript

describe('Item component', function() {
  var element, scope;

  beforeEach(module('app'));

  beforeEach(inject(function(_$rootScope_, _$compile_) {
    $compile = _$compile_;
    $rootScope = _$rootScope_;

    scope = $rootScope.$new();

    element = angular.element('<item data-set="item" on-click="ctrl.callback({item: item})"></item>');

    $compile(element)(scope);

  }));

  it('should display the controller defined title');
  it('should call the controller defined callback');
});

```

Currently there are two formulated tests that should describe the expected \<item\> component behaviour. 
The first test needs to verify that a passed item's name is really rendered when calling the item component.
By adding an item to the scope and calling _scope.$digest()_, which simply updates the scope properties on the 
previously created element, we are able to verify if the expected title has been rendered via Jasmine's _toContain()_ method.

```javascript
  it('should display the controller defined title', function() {
    var expected = 'Some rendered text';
    scope.item = {
      name: expected,
      active: false
    };

    scope.$digest();

    expect(element.text()).toContain(expected);
  });
```

The second test needs to simulate a click behaviour as it expects a pre defined callback to be triggered when clicking on an 
_item_ checkbox. This test setup also loads **jQuery** so simulating a click on an element can simply be achieved by selecting
an element and applying the _click()_ call.

```javascript
 it('should call the controller defined callback', function() {
    scope.ctrl = {
      callback: jasmine.createSpy('callback')
    };

    scope.$digest();

    element.find('input[type=checkbox]').eq(0).click();

    expect(scope.ctrl.callback).toHaveBeenCalled();
  });
```

It should be noted that the callback function was mocked by using Jasmine's _createSpy_ method.
This approach comes with the benefit that we can test if the callback has been called by simply using the _toHaveBeenCalled()_ method.

<p><iframe style="border: 1px solid #999; width: 100%; height: 500px; background-color: #fff;" src="http://embed.plnkr.co/RZ2MlOiH76VoFcIX7C6U/item.test.js" height="300" width="320" allowfullscreen="allowfullscreen" frameborder="0"></iframe></p>

### Testing the \<itemsList\> and \<searchBox> Component

The tests for the itemsList and searchBox components are very similar to the previously created item tests.
One important difference though is the fact that both the itemsList as well as the searchBox component rely on an external template defined via 
the directive's _templateUrl_ property.

There are different approaches to include an external template into the to be tested directive.

One possibility is to setup Karma to serve the templates via the _karma-ng-html2js-preprocessor_. More information can be found [here](https://github.com/karma-runner/karma-ng-html2js-preprocessor).
The other approach, which we will actually use in this example, is to add a string template into **$templateCache** inside the _beforeEach_ function. This ensures that the template is available 
when the directive test runs. More information on $tempateCache can be found [here](https://docs.angularjs.org/api/ng/service/$templateCache).

```javascript
  beforeEach(inject(function($templateCache) {
    $templateCache.put('items-list.html',
      '<div class="items-list">' +
      '<h3>{{ctrl.title}}</h3>' +
      '<span ng-if="ctrl.items.length == 0">No items available.</span> ' +
      '<ul class="items"> ' +
      '<li ng-repeat="item in ctrl.items"> ' +
      '<item data-set="item" on-click="ctrl.onClick({item: item})"></item> ' +
      '</li> ' +
      '</ul> ' +
      '</div>'
    );
  }));
```

The \<itemsList\> component tests are straight forward. The scope is extended with an items array to verify that the items are really 
rendered as single item elements. The rest is similar to the previous section, including testing if a passed in title is really rendered.
The full code below:

<p><iframe style="border: 1px solid #999; width: 100%; height: 500px; background-color: #fff;" src="http://embed.plnkr.co/RZ2MlOiH76VoFcIX7C6U/itemsList.test.js" height="300" width="320" allowfullscreen="allowfullscreen" frameborder="0"></iframe></p>

One interesting aspect to consider when testing the \<searchBox\> component is a that user input has to be simulated.
This is due to the fact that a callback is triggered via the input element's _ng-change_ property. 
Simulating the input can be achieved by accessing the input's ngModelController and changing the view value by using _$setViewValue()_.

```javascript
  it('changing the search string should trigger the controller defined callback', function() {
      scope.ctrl = {
          updateFilter : jasmine.createSpy('updateFilter')
      };

      scope.$digest();

      var input = element.find('#search-box').eq(0);
      var ngModelCtrl = input.controller('ngModel');
      ngModelCtrl.$setViewValue('view');

      expect(scope.ctrl.updateFilter).toHaveBeenCalledWith('view');
  });
```

<p><iframe style="border: 1px solid #999; width: 100%; height: 500px; background-color: #fff;" src="http://embed.plnkr.co/RZ2MlOiH76VoFcIX7C6U/searchBox.test.js" height="300" width="320" allowfullscreen="allowfullscreen" frameborder="0"></iframe></p>


### Testing the ItemsContainerController

The ItemsContainer composes all the previously tested components. 

Testing the **ItemsContainerController** ensures that the data updates are handled as expected and 
that the lower level components receive the correct data to function properly.
Further the container also gets an ItemsService injected, which we will mock when testing
the overall component behaviour. Mocking can be achieved by creating a dummy service object for example. 
The mocked object will always return the same set of items.

```javascript
  beforeEach(function() {
    items = [{
      id: 1,
      name: 'view',
      active: true
    }, {
      id: 2,
      name: 'model',
      active: true
    }, {
      id: 3,
      name: 'scope',
      active: false
    }];

    mockedItemsService = {
      fetchAll: function() {
        return items;
      },
      update: function(item) {
        return items;
      }
    };
  });
```

One interesting aspect is that we can test a _'controller as'_ controller by simply declaring the 
controller with the 'controller as' syntax when creating the controller via $controller.

```javascript
  beforeEach(inject(function($controller, _$rootScope_, _$compile_) {
    $compile = _$compile_;
    $rootScope = _$rootScope_;

    scope = $rootScope.$new();
    controller = $controller('ItemsContainerController as ctrl', {
      $scope: scope,
      ItemsService: mockedItemsService
    });
  }));
```

The rest is obvious, we are testing the default behaviour when the itemsContainer component is rendered as well 
as the _updateFilter_ and  _switchStatus_ functions.

<p><iframe style="border: 1px solid #999; width: 100%; height: 500px; background-color: #fff;" src="http://embed.plnkr.co/RZ2MlOiH76VoFcIX7C6U/preview" height="300" width="320" allowfullscreen="allowfullscreen" frameborder="0"></iframe></p>


The full code is available **[here](http://plnkr.co/edit/RZ2MlOiH76VoFcIX7C6U?p=preview)**.

### Links

[Component Based Thinking in AngularJS](http://busypeoples.github.io/post/thinking-in-components-angular-js)

[Component-Based AngularJS Directives](https://www.airpair.com/angularjs/posts/component-based-angularjs-directives)

[Directive components in ng](https://docs.angularjs.org/api/ng/directive)

[Jasmine](http://jasmine.github.io/)

[karma-runner/karma-jasmine on GitHub](https://github.com/karma-runner/karma-jasmine)

[$tempateCache](https://docs.angularjs.org/api/ng/service/$templateCache)

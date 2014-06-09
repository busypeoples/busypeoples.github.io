---
layout: post
title:  "Promises in AngularJS"
date:   2014-06-08 12:00:00
tags:   AngularJS, Javascript, Unit Tests, Karma, Jasmine
published: false
---

###The promise concept in javascript

A promise in short is the future result of an operation or a placeholder for a successful result or an error including a reason why the error happened. The promise can either be resolved or rejected once, meaning a rejected promise can not be resolved again  and vice versa.

In general the promise concept tries to solve the problems that occure when dealing with asynchronous actions. One workaround has always been to use callbacks to handle async operations.
While callbacks solve the problem at first glance, they introduce other problems including having callbacks call other callbacks as soon as more than one asynchronous event is needed and they depend on one another.

Error handling becomes complex as every asynchronous operation would have to be handled explicitly, thus losing the ability to have a try/catch block.
Promises offer the option to add success and error callbacks that will be called depending on the outcome and enabling to simulate a try/catch block even when dealing with asynchronous events.

Another advantage that comes with using promises is the ability to compose mulitple operations and have them run sequential or even in parrallel by using Q.all for example.

We also have to differentiate between Promise/A and Promise/A+ specs, while the first only follows a minimal definition, it's the A+ specs that extends the definition heavily.
There are multiple libraries that implement the promise concept including [Q](https://github.com/kriskowal/q), [when.js](https://github.com/cujojs/when) and [Bluebird](https://github.com/petkaantonov/bluebird) (all implementing the Promises/A+ spec).

###Promises in AngularJS
AngularJS makes use of promises in multiple parts of the framework, including $http, $routeProvider and $timeout. AngularJS has its own promise library called **$q**, which is a minimalistic implementation of the Q library.
Similar to jqLite which offers a subset of the jQuery library. The only difference here is that the framework will use jQuery in case jQuery is loaded before AngularJS, but will not use a different promise library if the promise library is loaded before AngularJS.
$q is also tightly interwoven with **$rootScope.Scope** to resolve and reject promises in a more timely manner without having to redraw the browser.

###$q
The $q api exposes four methods: _defer_, _reject_, _when_ and _all_. For more details on the $q api see the [documentation](https://docs.angularjs.org/api/ng/service/$q).

###Deferred
As mentioned above, a new deferred instance is created when calling **$q.defer()**.
We can easily access the associated promise via the promise property in the deferred object. Further deferreds expose an api for defining the outcome of an operation: resolve (successful), reject (unsuccessful) and notify (updating the status of an operation).

###Promise
A Promise instance is automatically created when **$q.defer()** has been called. The Promise api offers the _then_ method with which we can react to the outcome of an operation. We can add a success, error and even a notification callback.
It is important to note that _then_ always returns a new Promise.
We will go into more detail up next.

###Basics examples

```javascript
var foo = $http.get('some/api/path');
console.log(foo); // foo will not have the expected result...
```

A better approach would be to use _then_ and define a success and error callback, just to handle the outcome of an http call accordingly.

```javascript
$http.get('some/api/path')
	// on success...
	.then(function(result) {
		foo = result.data;
		console.log(foo);
	},
	// on failure...
	function(errorMsg) {
		console.log('Something went wrong: ' + errorMsg);
	});
```

Next, let us have a look at how to write a factory that depends on promises.

```javascript
angular.module('Foo')
  .factory('ServiceApi', ['$resource', '$q', function($resource, $q) {
    function ServiceApi() {

      this.query = function() {
          var deferredObject = $q.defer();

          // retrieve the information...
          // no caching here. but can easily be added.
          $resource('path/to/api/')
            .query()
            .$promise
            .then(function(result) {
              deferredObject.resolve(result);
            }, function(errorMsg) {
                deferredObject.reject(errorMsg);
            });

          return deferredObject.promise;
      };
    }

    return new ServiceApi();
  }]);

```

The example relies on $resource to fetch certain information via a defined api.

```javascript
$resource('path/to/api/')
        .query()
        // access the promise
        .$promise
        // define success and error callbacks.
        .then(function(result) {
```

The _query()_ method returns a promise.

```javascript
return deferredObject.promise;
```

The controller (as seen in the next example) defines a success and an error callback via the then() method.

```javascript
angular.module('Foo')
  .controller('FooController', ['$scope', 'ServiceApi', function($scope, ServiceApi) {
   ServiceApi.query().then(function( result ) {
     $scope.isLoaded = true;
     $scope.result = result;
   }, function(error) {
     $scope.error = 'has failed... ' + error;
   });
}]);
```

###Using $q to handle multiple requests
AngularJS with its dependency injection makes it too easy to rely on $scope. When working with controllers one simply injects $scope and then heavily uses $watch to trigger actions as soon as certain collections or properties or objects change.
Complexity arises as soon as the controller relies on multiple asynchronous events. Imagine one might need different rest calls to resolve and a service needs to have all data loaded before it can return a certain result needed inside the view. Things become overly complicated.
Having multiple watchers including different ifs and returns in multiple methods. The controller becomes messy and hard to maintain. Logic might be spread all over the place.
Especially when we only need the data loaded once. Watchers can quickly become a performance problem too, plus having to keep state when what has loaded can lead to even more useless code.
This is where $q.all() really comes handy and solves multiple problems. $q.all handles the initial data loading and as soon as all the data has been loaded, we can simply trigger an init() method
or define necessary actions via callback.

```javascript
angular.module('Foo')
  .controller('FooController', ['$scope', '$http', '$q',
    function($scope, $http, $q) {

   promiseA = $http.get('api/path/one');
   promiseB = $http.get('api/path/two');
   promiseC = $http.get('api/path/three');

    $q.all([promiseA, promiseB, promiseC])
      .then(function(results) {
      // do something here
      // this is the new init...
      console.log(results[0], results[1], results[2]);
    });
```

Or alternatively define an object and access the results via properties (f.e. results.a etc.):

```javascript
     $q.all({a: promiseA, b: promiseB, c: promiseC})
      .then(function(results) {
        console.log(results.a, results.b, results.c);
      });
```
Promises can help to solve the initial loading problem and minimize complexity when dealing with asynchronous operations especially when multiple calls (f.e. Rest calls) are involved.

It is important to note that as soon as one promise is rejected or throws an error, the error callback is triggered.

```javascript
  $q.all([promiseA, promiseB, promiseC])
    .then(function(results) {
      console.log(results[0], results[1], results[2]);
    }, function(errorMsg) {
	// if any of the previous promises gets rejected
	// the success callback will never be executed
	// the error callback will be called...
      console.log('An error occurred: ', errorMsg);
    }
  );
```

###Chaining Promises
Another advanced feature that might come handy in certain situations is the ability to handle multiple asynchronous calls sequentially.
This is especially useful when one rest call depends on information that another rest call response has to retrieve.
Imagine querying a user via an api to retrieve a certain user id for querying another api with this specific id.
Take a look at the following code:

```javascript
angular.module('Foo')
  .controller('FooController', ['$scope', '$resource',
    function($scope, $resource) {
       $scope.user = {};

        $resource('/api/path/')
          .query()
          .$promise
          .then(function(result) {
            $scope.user.id = result;
            return $resource('/api/path/two/' + $scope.user.id).query().$promise;
          })
          .then(function(result) {
            $scope.user.details = result;
            return $resource('/api/path/three/' + $scope.user.details).query().$promise;
          })
          .then(function(result) {
            $scope.user.somethingElse = result;
          })
```
Further a catch and a finally can be added to cover all possible outcomes.
Finally will always be called and can be used for certain clean up actions or other code that should always be executed no matter
if the response was successful or not. The catch is executed as soon as a single exception or failure happens somewhere up the chain.

```javascript
		  // will be called as soon as an exception
		  // or failure happens further up the chain
          .catch (function(errorMsg) {
            console.log("Something went wrong : " + errorMsg);
          })
           // will always be executed
          .finally(function() {
            console.log('This will always be called...');
          });
    }
  ]);
```

This example is based on $resource, we could also have simply returned any value via the chain or used $http or even rolled out
our own promise implementation. Here an alternative approach, just to clarify the promises chaining.
The previous case always returned a promise, which meant that the next then() only got executed when the promise was resolved.

```javascript

var deferredObj = $q.defer();
var promise = deferredObj.promise;

promise
.then(function(result) {
  $scope.user.id = result;
  return { name: 'foo' };
})
.then(function(result) {
  $scope.user.details = result;
  return { something: 'bar'};
})
.then(function(result) {
  $scope.user.somethingElse = result
})
.catch (function(errorMsg) {
  console.log("Something went wrong : " + errorMsg);
})
.finally(function() {
  console.log('This will always be called...');
});

// resolve the deffered...
deferredObj.resolve(101);
```

###Testing Promises
Finally let us have a look at how to test a service or controller that depends on promises and how to mock promises.
We will write controller tests that rely on the previously created ServiceApi.

```javascript
describe('Test FooController', function() {

  var $scope, $rootScope, $q, controller, mockedApiService, queryDeferred;
  var expectedResponse = [{ name: 'foo'}, { name: 'bar'}];

  beforeEach(module('Foo'));

  beforeEach(inject(function(_$rootScope_, _$q_, $controller) {
    $q = _$q_;
    $rootScope = _$rootScope_;
    $scope = $rootScope.$new(),

```

We can simply mock the ApiService and define a query method that simply returns a promise.
By doing so we can now explicitly resolve or reject the deffered object, enabling us to test different scenarios very easily (as shown later).

```javascript
    mockedApiService = {
      query: function() {
        queryDeferred = $q.defer();
        return queryDeferred.promise;
      }
    }

	// use the Jasmine spyOn method
	// or use and.callThrough() when using Jasmine 2.0
    spyOn(mockedApiService, 'query').andCallThrough();

    // inject the mocked Service into the controller.
    controller = $controller('FooController', {
      '$scope': $scope,
      'ServiceApi': mockedApiService
    });

  }));
```

Finally some examples for testing code that depends on promises in AngularJS.

```javascript
  describe('When information has been successfully loaded', function() {

	// we can now easily resolve or reject the deffered
	// enabling us to test different scenarios.
    beforeEach(function() {
      queryDeferred.resolve(expectedResponse);
      // make sure to call $apply
      // else the tests will have a different outcome than expected.
      $rootScope.$apply();
    });

    it('ApiService.query() shold have been called', function() {
      expect(mockedApiService.query).toHaveBeenCalled();
    });

    it('the ApiService should have returned the correct response', function() {
      expect($scope.result).toBe(expectedResponse);
    });

    it('isLoaded should be true', function() {
      expect($scope.isLoaded).toBeTruthy();
    });

  });

  describe('When information has not been successfully loaded', function() {

	// example for rejecting the deffered
    beforeEach(function() {
      queryDeferred.reject('went wrong');
      $rootScope.$apply();
    });

    it('there is an error message', function() {
      expect($scope.error).toBe('has failed...went wrong');
    });

    it('the ApiService should have returned the correct response', function() {
      expect($scope.result).toBeUndefined();
    });

    it('isLoaded should be false', function() {
      expect($scope.isLoaded).toBeFalsy();
    });

  });

});
```

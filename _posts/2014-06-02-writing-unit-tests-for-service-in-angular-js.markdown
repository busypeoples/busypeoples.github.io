---
layout: post
title:  "Writing Unit Tests for an AngularJS Service"
date:   2014-06-02 21:30:30
tags:   AngularJS, Javascript, Unit Tests, Karma, Jasmine
---

The following post will assume prior knowledge of how to set up [karma](http://karma-runner.github.io/0.12/index.html) and write tests with [Jasmine](http://jasmine.github.io/).
This is just a short introduction into getting started with writing tests for AngularJS services. 
AngularJS offers multiple ways for creating a [provider](https://docs.angularjs.org/guide/providers).
Depending on what the service class should accomplish one can choose between a service, factory or provider recipe. If no configuration or new instance is needed than a service provider should do the job.
It does not matter what provider type is chosen, the tests will run for all the previously mentioned.

The first example is really trivial:

We have no dependencies, we just want to test the service.

<p><iframe style="border: 1px solid #999; width: 100%; height: 240px; background-color: #fff;" src="http://embed.plnkr.co/A0Q8FiibMwEMr1qoxm31/script.js" height="240" width="320" allowfullscreen="allowfullscreen" frameborder="0"></iframe></p>
	
The first test was to verify that the service was loaded properly and that calling the getData method returned the correct string.
The same would work if the service was an angular.service or angular.provider.

Now what if the service needed to communicate with a backend server using $http? Service gets $http injected via AngularJS dependency injection.
But in our tests we do not really want to send any requests, we want to interact with a mocked http service. We do not want to test if http works, http might even fail, also causing our service to fail.
To avoid any problems or to isolate the to be tested service we will mock all dependencies when testing.
The good part is that angular even provides a low level http provider called $httpBackend, that can be easily injected into the test.
This is very useful as we don not even have to care about mocks in this case.

First the extended service class.

<p><iframe style="border: 1px solid #999; width: 100%; height: 300px; background-color: #fff;" src="http://embed.plnkr.co/E4Snya7H3CMK4sQwrlqJ/script.js" height="300" width="320" allowfullscreen="allowfullscreen" frameborder="0"></iframe></p>
	

And the accompanying tests. We can define callbacks when a certain expected call happens, even verify that no outstanding expectations or outstanding requests exist. This is possible because httpBackend gets injected via the Angular DI.
Due to the fact we can return any values we want and we really only test the service class and not the dependencies.

<p><iframe style="border: 1px solid #999; width: 100%; height: 400px; background-color: #fff;" src="http://embed.plnkr.co/E4Snya7H3CMK4sQwrlqJ/test.js" height="160" width="320" allowfullscreen="allowfullscreen" frameborder="0"></iframe></p>
	

Our final example will focus on a possible setup where one service might depend on another service.
We are dealing with mocks again, only this time we also have to create the mocks.

<p><iframe style="border: 1px solid #999; width: 100%; height: 320px; background-color: #fff;" src="http://embed.plnkr.co/qXqitB5MO3T4hshYL4Pw/script.js" height="320" width="320" allowfullscreen="allowfullscreen" frameborder="0"></iframe></p>

<p><iframe style="border: 1px solid #999; width: 100%; height: 400px; background-color: #fff;" src="http://embed.plnkr.co/qXqitB5MO3T4hshYL4Pw/test.js" height="400" width="320" allowfullscreen="allowfullscreen" frameborder="0"></iframe></p>

Finally an alternative approach, this one uses jasmine to create a spy object, which is used to mock the service and return expected data via the andReturn() method.

<p><iframe style="border: 1px solid #999; width: 100%; height: 400px; background-color: #fff;" src="http://embed.plnkr.co/tleSYIhjU7dZSkWvwhwv/test.js" height="400" width="320" allowfullscreen="allowfullscreen" frameborder="0"></iframe></p>

Testing services in AngularJS is easy to accomplish or at least easy to setup as shown above. We can isolate dependencies when testing services, no matter if the dependency is an AngularJS provider or a self implemented service. 

---
layout: post
title:  "$asyncValidators, $touched and ngMesages: The State Of Forms In AngularJS 1.3"
date:   2015-01-29 12:00:00
tags:   AngularJS, Javascript
published: false
---

### The Basics

AngularJS 1.3 introduced a number of new features and improvements comapred to 1.2.
The major change is probably the introduction of the validator pipeline, which makes validation handling more obvious without having to use 
$parsers and $formatters. The later now only deal with transforming values between the model and the view.
The other big feature is ngMessages which will improve handling validation errors, one of the more chaotic parts in AngularJS up until now.
Forms and Inputs still have $error, $valid, $invalid, $pristine and $dirty properties, nothing changed here. 
Which means if your forms worked in 1.2 they will also work 1.3. The following should be a round up of the most important new features.
 
### $validators
AngularJs 1.3 already features a large number of validators out of the box, including validators fo HTML5 attributes like required, min and max  as well as well as for 
input types like number, url, email and more. Visit the documentation for a more detailed listing. 
With the introduction of $validators adding custom validations becomes more straight forward. In 1.2 one had to use the $parsers/$fomatters to add a validation function
and explicitly set the validatity via the ngModel controller.

In 1.3 the ngModel controller now includes the $validators object. To add an new validation rule one can simply add a new property to the $validators object:

```javascript
	ngModel.$validators.customValidation = function(value) {
		// return true;
	};
```

To create a custom validator we must now create a custom directive and require the ngModel controller to add the validator to ngModel.$validators. 
The validation function itself has to return a boolean value and that's simply it.

```javascript

	app.directive('customValidator', function($q) {

		return {
			require : 'ngModel',
			link : function($scope, $attr, $elem, controller) {
				
				controller.$validators.customValidator = function(value) {
					return value && value.length > 8;				
				};
			}
		}
		
	});
	
```

On the html side of things the custom validator is added as an attribute to the input. Angular will run the validators one after the other. 
In the following example _required_ would run first, then _minlenght_ and finally the custom validator. Obviously this makes sense of course.


```html

<input type="text" ng-model="example" name="example" required minlength="5" custom-validator />

```

*Quick Summary: register a validation to ngModel.$validators not to $parsers or $formatters. The validation function only has to return a boolean value.*


### $asyncValidators

If you need to validate a certain input againt a restful api in your application then $asyncValidators will come handy.
The approach is similar to the previous exmaple only that instead of returning a boolean value, the validation function is expected to return a promise.
The following example uses the new es6 $q constructor to return the promise.


```javascript

app.directive('asyncDateValidator', function($timeout, $q) {
		return {
			require: 'ngModel',
			link: function($scope, $elem, $attr, controller) {
				controller.$asyncValidators.asyncValidator = function(value) {
					return asyncDateValidator(value);
				}
			}
		};
	});
```

The corresponding HTML code reveals nothing new here. It should be noted that the async validators will get triggered as soon as all the regular validators have passed.
This will prevent any unnecessary http calls for example. 

```html

<input type="text" ng-model="example" name="example" required minlength="5" custom-validator custom-async-validator />

```

*Quick Summary: register a asynchronous validation function to ngModel.$asyncValidators. The validation function has to return a promise.*

### $pending
When one ore more asynchronous validations are triggered, both the $valid as well as $invalid properties are set to undefined and the newly introduced property $pending
will be added to the Form and model signaling which async operations are in effect at the moement. As soon as all asynchronous operations have been resolved $pending gets removed
and the $valid and $invalid properties are set to the current validity.

```html

<div ng-if="yourForm.someField.$pending" class="loader">loading...</div>

```

*Quicktip: use $pending to display certain messages or animations when an asynchronous operation or operations are in effect.*


### $parsers
Parsers/Formatters deal with transforming values between the model and the view, and while they also handled validation logic in 1.2 and below they don't handle 
any validation logic anymore. This is left to the aforementioned $validators/$asyncValidators combo.  
To clarify what $parser do, imagine your model calculates with seconds while the user types in hours. 
To solve the problem of transforming the user input into the required model format we would create a custom directive, that requires ngModel. This enables us to register
transform functions to $parsers and $formatters.

```javascript
	app.directive('hourSecondsConverter', function() {
		var SECONDS_IN_HOUR = 3600;
		
		return {
			require: 'ngModel',
			link: function($scope, $attrs, $element, modelCtrl)  {
		
				modelCtrl.$parsers.push(function(viewValue) {
					return viewValue * SECONDS_IN_HOUR;
				});
				
				modelCtrl.$formatters.push(function(modelValue) {
					return modelValue / SECONDS_IN_HOUR;
				});
				
			}
		};
	});
```

*Quicktip: Use $parsers to convert the view value to the corresponding model value. Remove any validation logic, this should be handled by the validator chain.*

### $formatters
Nothing has really changed when dealing with ngModel.$formatters. See the above section for more informtion.

*Quicktip: Use $formatters to convert the model value to the corresponding view value. Remove any validation logic, this should be handled by the validator chain.*

### ngMessages
The newly introduced ngMessages module should make error display handling easier.   
The ngMessage script file itself has to be added to your project and included as a dependency to the application, it's not available out of the box.
The main features include error order tracking, displaying error message after error message and also includes the possibility to define globale error messages to be 
reused all over the application. 

Previously managing validation errors via ng-if or ng-show, which meant writing a lot of code, making things get very messy at times.

```html
<div>
	<div ng-if="main.example.$error.minlength">Min length: 5</div>
	<div ng-if="main.example.$error.required">Required!</div>
	<div ng-if="main.example.$error.customValidator">Required and atleast 8 chars!</div>
	<div ng-if="main.example.$error.asyncValidator">Async validtor</div>
	<div ng-if="main.example.$pending">fetching data...</div>
</div>
```

The previous code would change to this:

```html
<div ng-messages="main.example.$error">
	<div ng-message="required">Required!</div>
	<div ng-message="minlength">Min length: 5</div>
	<div ng-message="customValidator">Required and atleast 8 chars!</div>
	<div ng-message="asyncValidator">Async validtor</div>
</div>
<div ng-if="main.example.$pending">fetching data...</div>
```

Further more ngMessages enables to define message templates and include them when showing any errors.

```html
	<div ng-message="required">Required!</div>
	<div ng-message="minlength">Min length: 5</div>
	<div ng-message="customValidator">Required and atleast 8 chars!</div>
	<div ng-message="asyncValidator">Async validtor</
```

Reusing the template is simply achieved via the ng-messages-path attribute.

```html
<div ng-messages="main.example.$error" ng-messages-include="messages.html">
</div>
```

Making error messgaes reusable means less code and if you need to override single messages this can also be easily achieved by really
simply overriding the existing message:

```html
<div ng-messages="main.example.$error" ng-messages-include="messages.html">
	<div ng-message="minlength">Custom Error Message: Min length: 5</div>
</div>
```

Displaying multiple messages can be achieved by adding the ng-message-multiple attribute:

```html
<div ng-messages="main.example.$error" ng-messages-multiple>
	<div ng-message="required">Required!</div>
	<div ng-message="minlength">Min length: 5</div>
	<div ng-message="customValidator">Required and atleast 8 chars!</div>
	<div ng-message="asyncValidator">Async validtor</div>
</div>

```


### $touched

Prior to the introduction $touched we could use $dirty to identify if a user has interacted with an input, which also meant we would be able to display errors 
when $dirty was set to true. With 1.3 with can now also react as the input field is focused and then blurred. 
Basically $touched is set to true as soon as the user interacts with the input field and loses focuse the $touched is set to true.
This addition to ngModel enables us to display errors as soon as the focus is lost on a given input field.

```html
	<div ng-if="main.example.$touched">
		<div ng-if="main.example.$error.required">Required!</div>
		<div ng-if="main.example.$error.minlength">Min length: 5</div>
	</div>
```

*Quicktip: use $touched to display error messages when an input field is focued and then blurred.*

### $submitted 

The FormController has  a new property $submitted which obviously indicated that the form has been submitted.
Prior to 1.3 we would have to implement our mechanism to handle form submits if we wanted to display error messages after the user has submitted the form.
For example we would set an indicator to true in the submit function, which gets called via ng-click on the submit button.

```html
	<button ng-click="submitAction()">Submit</button>
	
	<div ng-if="submitted">
		<-- show errors --!>
	</div>
```

```javascript
	function submitAction() {
		$scope.submitted = true;
	}
```

The previous code can by reduced to something similar to this:

```html
	<button ng-click="submitAction()">Submit</button>
	
	<div ng-if="yourForm.$submited">
		<-- show errors --!>
	</div>
```

### Round Up

Form handling and validation in AngularJS has always been one of the more chaotic parts of the framework. Especially the validation part was partly messy and confusing. at times 
The introduction of the validator pipeline and ngMessages will improve development and validation in particular.
For more information on forms checkout the links below.

### Links

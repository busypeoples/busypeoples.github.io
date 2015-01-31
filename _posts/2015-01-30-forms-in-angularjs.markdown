---
layout: post
title:  "$asyncValidators, $touched and ngMessages: Form Validation In AngularJS 1.3"
date:   2015-01-30 12:00:00
tags:   AngularJS, Javascript
published: true
---

### The Basics

Form validation handling has always been a complex and rather confusing task in **AngularJS**.
AngularJS 1.3 has introduced a number of improvements and changes that should make things a little easier from now on.
The major change is probably the introduction of the **validator pipeline**, which makes validation handling more obvious without having to use 
$parsers and $formatters as a workaround. The later should only be used for transforming values between the model and the view.

The other big feature is **ngMessages** which should improve validation error display handling, one of the more chaotic parts in AngularJS up until now.
Forms and Inputs still have $error, $valid, $invalid, $pristine and $dirty properties, nothing changed here. 
Which means if your forms worked in 1.2 they will also work 1.3. The following should be a round up of the most important new features.
 

### $validators

A large number of validators already come out of the box including validators for HTML5 attributes like **_required_**, **_min_** and **_max_**  as well as well as for 
input types like **_number_**, **_url_**, **_email_** and more, [consult the documentation for supported input types](https://docs.angularjs.org/api/ng/directive/ngModel). 
Adding own custom validators is done by registering validation functions to **$validators**, which makes the process more straight forward. 
In 1.2 one had to use **$parsers/$formatters** to add a validation function and explicitly set the validity via the ngModel controller.

To add an new validation rule one can simply add a new property to the $validators object
and define a function that handles the validation logic:

```javascript
ngModel.$validators.customValidation = function(value) {
	// return true;
};
```

To create a custom validator we need to create a custom directive first that also requires the **ngModelController**.
The controller is needed to add the validator to the $validators object. 
The validation function itself has to **_return a boolean_** value and that's simply it.

```javascript
app.directive('customValidator', function($q) {
	return {
		require : 'ngModel',
		link : function($scope, $attr, $elem, ctrl) {
			ctrl.$validators.customValidator = function(value) {
				// validation logic
				// return true/false;				
			};
		}
	}
});
```

On the html side of things the custom validator is added as an attribute to the input. Angular will run the validators one after the other. 
In the following example _required_ would run first, then _minlength_ and the custom validator. Obviously this makes sense.
There need to validate on minimum length if the required validation failed in the first place.


```html

<input type="text" ng-model="example" name="example" required minlength="5" custom-validator />

```

### Overriding Built-in Validators

Overriding built-in AngularJS validators is simple due to the fact that the aforementioned validators are also registered via **$validators**.
The following example would override the built in time validation.

```javascript
app.directive('customTimeValidator', function($q) {
	return {
		require : 'ngModel',
		link : function($scope, $attr, $elem, ctrl) {
			ctrl.$validators.time = function(value) {
				// return something...				
			};
		}
	}
});
```


### $asyncValidators

If you need to validate a certain input against a restful api in your application then **$asyncValidators** will come handy.
The approach is similar to the previous example only that instead of returning a boolean value, 
the validation function is expected to **_return a promise_**.


```javascript
app.directive('asyncDateValidator', function($timeout, $q) {
	return {
		require: 'ngModel',
		link: function($scope, $elem, $attr, ctrl) {
			ctrl.$asyncValidators.asyncValidator = function(value) {
				// should always return a promise
				//return asyncDateValidator(value);
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


### $pending

When one ore more asynchronous validations are triggered, both the **_$valid_** as well as **_$invalid_** properties are set to undefined and the newly introduced property **$pending**
will be added to the Form and model signaling which async operations are in effect at given moment. 
As soon as all asynchronous operations have been resolved $pending gets removed
and the $valid and $invalid properties are set to the current validity.

```html

<div ng-if="yourForm.someField.$pending" class="loader">loading...</div>

```


### $parsers

Parsers/Formatters deal with transforming values between the model and the view, and while they were also used to handle validation logic in previous versions they shouldn't handle 
any validation logic anymore. This is left to the aforementioned **_$validators/$asyncValidators_** combo.  
To clarify what **$parsers** does, imagine your model calculates with seconds while the user types in hours. 
To solve the problem of transforming the user input into the required model format we would create a custom directive, that requires ngModel. This enables us to register
transform functions to $parsers and $formatters.

```javascript
app.directive('hourSecondsConverter', function() {
	var SECONDS_IN_HOUR = 3600;

	return {
		require: 'ngModel',
		link: function($scope, $attrs, $element, ctrl)  {
	
			ctrl.$parsers.push(function(viewValue) {
				return viewValue * SECONDS_IN_HOUR;
			});
			
			ctrl.$formatters.push(function(modelValue) {
				return modelValue / SECONDS_IN_HOUR;
			});
		}
	};
});
```

### $formatters

Nothing has really changed when dealing with ngModel.$formatters. See the above section for more informtion.


### ngMessages

The newly introduced **ngMessages** module should make error display handling easier.   
The ngMessage script file itself has to be added to your project and included as a dependency to the application, it's not available out of the box.
The additional script can either be installed via **_bower_** or **_npm_**.
The main features include error order tracking and the possibility to define global error messages to be 
reused all over the application. 

Previously managing validation errors was mostly achieved via **_ng-if_** or **_ng-show_**, 
which meant writing a lot of code and making things get messy along the way, especially when a large number of validation rules applied.

```html
<div>
	<div ng-if="main.example.$error.required">Required Message</div>
	<div ng-if="main.example.$error.minlength">Min length Error Message</div>
	<div ng-if="main.example.$error.customValidator">Custom Error Message</div>
	<div ng-if="main.example.$error.asyncValidator">Custom Async Error Message</div>
	<div ng-if="main.example.$pending">Fetching Data...</div>
</div>
```

The previous code can be rewritten using the **ng-messages** directive:

```html
<div ng-messages="main.example.$error">
	<div ng-message="required">Required Message</div>
	<div ng-message="minlength">>Min length Error Message</div>
	<div ng-message="customValidator">Custom Error Message</div>
	<div ng-message="asyncValidator">Custom Async Error Message</div>
</div>
<div ng-if="main.example.$pending">Fetching Data...</div>
```

Further more, ngMessages enables the developer to define _message templates_ and including them in multiple forms.

```html
	<div ng-message="required">Required Message</div>
	<div ng-message="minlength">>Min length Error Message</div>
	<div ng-message="customValidator">Custom Error Message</div>
	<div ng-message="asyncValidator">Custom Async Error Message</div>
```

Reusing the template is simply achieved via the **_ng-messages-path_** attribute.

```html
<div ng-messages="main.example.$error" ng-messages-include="messages.html">
</div>
```

Making error messages reusable means less code and if you need to override single messages this can also be easily achieved by
simply overriding the existing message:

```html
<div ng-messages="main.example.$error" ng-messages-include="messages.html">
	<div ng-message="minlength">Custom Error Message: Min length is...</div>
</div>
```

Displaying multiple messages can be achieved by adding the **ng-message-multiple** attribute:

```html
<div ng-messages="main.example.$error" ng-messages-multiple>
	<div ng-message="required">Required Message</div>
	<div ng-message="minlength">>Min length Error Message</div>
	<div ng-message="customValidator">Custom Error Message</div>
	<div ng-message="asyncValidator">Custom Async Error Message</div>
</div>
```


### $touched

Prior to the introduction of **$touched**, **$dirty** was set to true as soon as a user had interacted with an input.
$touched adds the capability to react when the input field is focused and then blurred.
Basically $touched is set to true as soon as the user interacts with the input field and loses focus.
This addition to ngModel enables us to display errors as soon as the focus is lost on a given input field.

```html
<div ng-messages="main.example.$touched" ng-messages-multiple>
	<div ng-message="required">Required Message</div>
	<div ng-message="minlength">>Min length Error Message</div>
	<div ng-message="customValidator">Custom Error Message</div>
	<div ng-message="asyncValidator">Custom Async Error Message</div>
</div>
```


### $submitted 

The FormController has  a new property **$submitted** which obviously indicates that the form has been submitted.
Prior to 1.3 we would have to implement our own mechanism if we wanted to explicitly display error messages 
only after the user has submitted the form.
For example we would set an indicator to true inside the submit function, which gets called via ng-click on the submit button.

```html
<button ng-click="submitAction()">Submit</button>

<div ng-if="submitted">
	<!-- show errors -->
</div>
```

```javascript
function submitAction() {
	$scope.submitted = true;
}
```

There is no need to explicitly handle the submitted part via controllers from now on, this can achieved by testing the $submitted property.

```html
<button ng-click="submitAction()">Submit</button>

<div ng-if="yourForm.$submitted">
	<!-- show errors -->
</div>
```


### Round Up

Form handling and validation in AngularJS has always been one of the more chaotic parts of the framework. 
Especially the error displaying part was partly messy and confusing at times. 
The introduction of the $validators pipeline and ngMessages directive will improve form handling and validation in particular.
For more information on forms checkout the links below.


### Quick Summary

+ Register validation functions to ngModel.$validators not to $parsers or $formatters. The validation function has to return a boolean value.

+ Register an asynchronous validation function to ngModel.$asyncValidators. The validation function has to return a promise.

+ Use $parsers to convert the view value to the corresponding model value. Remove any validation logic, this should be handled by the validator chain.

+ Use $formatters to convert the model value to the corresponding view value. Remove any validation logic, this should be handled by the validator chain.

+ Use $pending to display certain messages or animations when an asynchronous operation or operations are in effect.

+ Use $touched to display error messages when an input field is focused and then blurred.


### Links

[Supported Input Types](https://docs.angularjs.org/api/ng/directive/ngModel)

[Forms Guide](https://docs.angularjs.org/guide/forms)

[ngModelController Documentation](https://docs.angularjs.org/api/ng/type/ngModel.NgModelController)

[ngModel Options](https://docs.angularjs.org/api/ng/directive/ngModelOptions)

[Directive components](https://docs.angularjs.org/api/ng/directive)

[ngMessages](https://docs.angularjs.org/api/ngMessages)
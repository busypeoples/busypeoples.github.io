---
layout: post
title:  "Reusable Chart Components With AngularJS and D3: An Introduction"
date:   2015-03-15 12:00:00
tags:   AngularJS, D3.js, Javascript, Components
published: true
---

### Introduction

The following should be an introduction to combining **D3.js** with **AngularJS** to create reusable chart components.
This is not intended to be an introduction on D3.js nor AngularJS, there is a large number of resources to help getting of the ground
with either. For example the [official D3 website](http://d3js.org/) and the extensive [AngularJS documentation](https://docs.angularjs.org/).

The prime objective of this post is to create a **\<chart\>** component that composes bar and axis components and further more
enables configuring the transitions of any component. The result should be a markup that 
describes the chart as well as the possibility to hook into the transition process and override default transition behaviours.

In the case of a bar chart, we would like to be able to declare the number of axis elements, their positions and the bar transitions 
when the data gets updated. The markup should look something like this:

```html
<chart id="chart" data-data="main.data" data-config="main.config">
    <bars config-key="bars"></bars>
    <axis position="left" config-key="bar"></axiss>
    <axis position="bottom" config-key="foo"></axis>
</chart>
```

This is the expected result:

<p><iframe style="border: 1px solid #999; width: 100%; height: 650px; background-color: #fff;" src="http://embed.plnkr.co/t3if5Z1hDapXFp4O5QPa/preview" height="550" width="320" allowfullscreen="allowfullscreen" frameborder="0"></iframe></p>


### Building A Basic \<Bar\> Component

Building the basic directive is trivial, the complex part is hooking into D3's **enter**, **update**, **exit** life cycle to 
define bar specific transitions.
Imagine being able to define the bars transitions via an outside controller or service for example. The callback
definitions might look similar to this:

```javascript
var config = {
    bars: {
          onEnter: function(config) {
            this.duration(1000)
              .attr('opacity', 0);
          },
          onTransition: function(config) {
            this.duration(1000)
              .attr('opacity', 1)
              .attr("x", function(d) {
                return config.x(d.x);
              })
              .attr("width", config.x.rangeBand())
              .attr("y", function(d) {
                return config.y(d.y);
              })
              .attr("height", function(d) {
                return config.height - config.y(d.y);
              });
          },
          onExit: function(config) {
            this.duration(1000)
              .attr('opacity', 0);
          }
     }
}
```

The above code reveals three callback definitions to enable transition handling when D3 enters, updates and removes bars.
A configuration is also passed into the callback function, so we can access _x_, _y_, _xAxis_ or _yAxis_ definitions inside the callback in case the 
transitions rely on the previous. These are computed by the **\<chart\>** directive and injected into the callback.
This approach opens up the possibility to define a number of bar components and individually define the transition 
behaviour. This also means removing any hard coded assumptions on how transitions should take place. 

Our **bar directive** needs to take care of receiving the needed data and configuration and rendering the bars as well as 
keeping them up to date. To make the directive work out of the box, we will also need a basic transition configuration, as a quasi fallback
when no explicit transition is defined.

The directive itself doesn't need a controller as everything will take place inside the **link** function.
Remember the markup:

```html
 <bars config-key="bars"></bars>
```

We need to define a _config-key_ attribute to later retrieve the proper configuration and we will also need to 
define an _update_ function which gets called when the data or configuration has changed.

```javascript
function barsLinkFn($scope, element, attrs, ctrl) {

    var key = attrs.configKey || 'undefined';
    
    function update(data) {
        var config = data.config, data = data.data;
    
        var bars = config.svg.selectAll(".bar")
            .data(data);
    
        var entered = bars.enter().append("rect")
            .attr('class', 'bar')
            .attr('opacity', 0);
    }
}
```

The _update_ function selects and updates the bars. Nothing special so far. The bars will not show up as we never defined any height,
width, x or y attributes.
The next step is to hook into the enter, update and exit cycle. This can be achieved by creating a service that combines the bar
with the callbacks to be executed when entering lifecycle specific points.

### Handling Specific Transitions

To decouple the actual transition handling from the element selection part, we will create a service that
expects an element and a number of callbacks to be applied on the given element. This approach comes with the benefit
of not having to write the same transition handling code in every chart component-directive.
The following is based on the the excellent blog post [exploring reusability with D3.js](http://bocoup.com/weblog/reusability-with-d3/)

The service calls the specific transitions via D3's _call_ method, which then triggers the transaction.  

```javascript
charts.service('ChartTransitions', function() {

    this.transition = function(entering, elements, transitions, config) {
        var onEnter = entering.transition();
        var transition = elements.transition();
        var exit = elements.exit();
        var onExit = exit.transition();
        
        if (transitions.onEnter) onEnter.call(transitions.onEnter.bind(onEnter, config));
        if (transitions.onTransition) transition.call(transitions.onTransition.bind(transition, config));
        if (transitions.onExit) onExit.call(transitions.onExit.bind(onExit, config));
    };
  
});
```

All the bar component needs to do now, is call the transition service with the required parameters.

```javascript

ChartTransitions.transition(entered, bars, transitions, config);

```

See the [code](http://plnkr.co/edit/t3if5Z1hDapXFp4O5QPa) for the full implementation. 

### Handling Data Updates 

Now that we have implemented the basic bar component we need a way to handle the data and corresponding updates and make sure 
every component receives the data needed to function. Any data is passed on to the  **\<chart\>** component via the _data_ property and the config respectively
via the _config_ property.

The **chart component** needs to take care of creating the initial SVG element, handling the data, computing the x, y, xAxis and yAxis properties and 
passing it on to any child component that has registered itself to be notified via an update.

The registration part can be handled by the directive controller, everything else should be defined inside 
the link function. As soon as the data changes, the _notify_ function is called which then simply tells the controller 
to notify all subscribers with the given data and configuration.

```javascript
function notify() {
    var data = $scope.ctrl.data || {};
    
    config.xAxis = d3.svg.axis()
        .scale(config.x)
        .orient("bottom");
    
    config.yAxis = d3.svg.axis()
        .scale(config.y)
        .orient("left");
    
    config.x.domain(data.map(function(d) {
        return d.x;
    }));
    config.y.domain([0, d3.max(data, function(d) {
        return d.y;
    })]);
    
    ctrl.notify({
          data: data,
          config: config
    });
}

$scope.$watch('ctrl.data', notify);
```

Check the [code](http://plnkr.co/edit/t3if5Z1hDapXFp4O5QPa) example for all implementation details. 

<p><iframe style="border: 1px solid #999; width: 100%; height: 650px; background-color: #fff;" src="http://embed.plnkr.co/t3if5Z1hDapXFp4O5QPa/preview" height="550" width="320" allowfullscreen="allowfullscreen" frameborder="0"></iframe></p>


###Roundup 

The main focus of this post was to demonstrate a possible way to compose different chart components into 
bigger components. The other main aspect was to hook into D3s lifecycle and enable transition configuration 
from outside the component, opening up the possibility to define individual transition behaviors for a number of 
components. 

Obviously this is just an introduction into the topic and the code examples need more refining and should therefore 
be seen as a primer into building a more solid approach.

A followup post will focus on adding more chart component types and a long the way refactoring the code for 
more reusability.

### Links

[D3](http://d3js.org/)

[Enter, Update, Exit in D3](https://medium.com/@c_behrens/enter-update-exit-6cafc6014c36)

[Exploring Reusability with D3.js](http://bocoup.com/weblog/reusability-with-d3/)

[AngularJS Directives](https://docs.angularjs.org/guide/directive)

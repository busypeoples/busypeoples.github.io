---
layout: post
title:  "Building a &lt;bar-chart&gt; element with D3.js and Polymer"
date:   2014-06-21 12:00:00
tags:   Polymer, D3.js, Javascript
published: false
---

###The Basics
**Polymer** is about custom elements, as everything is an element from a Polymer point of view.
Polymer is not the only library that enables creating custom elements, we could achieve the same with _AngularJS directives_ or _ember.js components_ for example.
For a very detailed explanation of what the differences between Polymer elements and angular directives check [this answer on stackoverflow](http://stackoverflow.com/questions/18089075/what-is-the-difference-between-polymer-elements-and-angularjs-directives)

Why would we even consider trying to build custom elements in the first place?

„Elements are the building blocks of the web“

Well the concept of web components is a standard that will be implemented into browsers natively,
only that Polymer already offers the capability to create these elements.

_„When we say “element”, we mean a real element, with all of the great properties of a built-in element.
And why limit elements to UI? Some of the properties of elements are UI-specific, but most of them aren't.
 Elements can serve as a generic package for reusable functionality.“_

([From the Polymer introduction](http://www.polymer-project.org/docs/start/everything.html))

Polymer consists of three conceptual layers: using as well as creating elements and the platform itself.
The platform leverages modern concepts like object.observe including the fallback when modern features are not implemented in a given browser,
by implementing the api in javascript.

In one of the [previous posts](http://busypeoples.github.io/post/testing-d3-with-jasmine/) we had written a bar chart in **D3.js** for testing purposes.
We will reuse large parts of the code, as this entry should focus on creating a *&lt;bar-chart&gt;* element.

Now imagine anyone could simply use the <bar-chart> element to render the bar chart.
This would make great reuse of code and would enable non developers to create charts without having to know the underlying implementation details.

###Creating the <bar-chart> element

Defining a Polymer element is done with a couple lines of code actually:

```html
<polymer-element name="bar-chart" attributes="originalData switchData">
    <template>
      <span class="bar-chart-example">
       <svg id="barchart" width="{{ width }}" height="height"></svg>
      </span>
      <div>
        <button class="btn" id="clickable">Flip Data</button>
      </div>
    </template>
	<script>
		// the javascript part obviously ...

		Polymer('bar-chart', {
            created: function() {
                //  when created ...
            },
            ready: function() {
                // when ready...
            }
          });
    </script>
</polymer-element>
```

We obviously defined a name for the polymer-element via the name attribute .
We also defined which attributes we would accept: _attributes="originalData switchData"_
and further more we also set up the template, which only consists of a span containing an svg and a button element.


The _barChart_ function depends on a pre defined svg element, this might also be implemented by having the bar chart create the svg,
but attention should be kept on the fact that [D3.js might have problems working in the shadow dom](http://stackoverflow.com/questions/20557913/d3-js-wont-work-in-a-shadowdom).

The _render_ methods expects an array containing the relevant data and then calls the appropriate bar and axis methods.
This is just a basic implementation of a d3 bar chart enabling transformations when data changes.


```javascript
function barChart(element) {
	var data = [];
	var that = {};
	var margin = {
	  top: 40,
	  right: 0,
	  bottom: 40,
	  left: 40
	};
	var h = 500 - margin.top - margin.bottom,
	  w = 500 - margin.top,
	  x, y;

	var svg = d3.select(element)
	  .append('g')
	  .attr("transform", "translate(" + margin.left + ", " + margin.top + ")");

	// add axis
	svg.append("g")
	  .attr("class", "x axis")
	  .attr("transform", "translate(0," + h + ")");

	svg.append("g")
	  .attr("class", "y axis");

	that.render = function(data) {
	  if (!data) return;

	  x = d3.scale.ordinal().rangeRoundBands([0, w], .05);
	  x.domain(data.map(function(d) {
	    return d.date;
	  }));

	  y = d3.scale.linear().range([h, 0]);
	  y.domain([0, d3.max(data, function(d) {
	    return d.value;
	  })]);

	  xAxis = d3.svg.axis()
	    .scale(x)
	    .orient("bottom");

	  yAxis = d3.svg.axis()
	    .scale(y)
	    .orient("left");


	  this.updateBars(data);
	  this.updateAxis();

	};

	that.updateBars = function(data) {
	  // add bars
	  var bars = svg.selectAll('.bar').data(data);

	  bars.enter().append('rect');

	  bars.exit().transition()
	    .duration(500)
	    .attr("height", 0)
	    .remove();

	  bars
	    .transition()
	    .duration(500)
	    .attr('class', 'bar')
	    .attr("x", function(d) {
	      return x(d.date);
	    })
	    .attr("width", x.rangeBand())
	    .attr("y", function(d) {
	      return y(d.value);
	    })
	    .attr("height", function(d) {
	      return h - y(d.value);
	    });
	};

	that.updateAxis = function() {
	  svg.selectAll("g.x.axis")
	    .call(xAxis);

	  svg.selectAll("g.y.axis")
	    .call(yAxis);
	};

	return that;
}
```

The Polymer part of the code might be more interesting.
We defined width and height and we can access the properties inside the template _&lt;svg id="barchart" width="&#123;&#123;width&#125;&#125;" height="&#123;&#123;height&#125;&#125;"&gt;&lt;/svg&gt;_.

The _created_ method simply initializes our data sets. The basic implementation can handle two data sets. These data sets are passed
on via properties.

```javascript
Polymer('bar-chart', {
    width: 500,
    height: 500,

    created: function() {
      this.data = this.data || [];
      this.other = this.other || [];
    },
    ready: function() {
    //
    }
  });
```

A lot more is happening inside the _ready_ method, for example we have to parse the string properties back into an array and we also initiated the bar chart,
the button and the button click event.
But this is all we need, to enable the bar chart to render from now on.

```javascript
ready: function() {
	var that = this;
	this.switched = false;

	// parse the string back into an array
	this.data = JSON.parse(this.originalData);
	this.other = JSON.parse(this.switchData);

	// find the svg element
	this.elem = this.$.barchart;

	// create and render the bar-chart
	this.c = barChart(this.elem);
	this.c.render(this.data);

	// define the btn element
	this.btn = this.$.clickable;

	// set to hidden if no second data set is set
	this.btn.hidden = !this.other;

	// define on click callback
	// switch between the two data sets...
	d3.select(this.btn).on('click', function() {
	var data = that.switched? that.data : that.other;
		that.c.render(data);
		that.switched = !that.switched;
	});
 }
```

And this is how we use the bar-chart element:

```html
<bar-chart originalData='[{"date":"04","value":"100"}, {"date":"05","value":"144"}{"date":"06","value":"123"}]'>
</bar-chart>
```

Or if we wanted to switch between two data sets, we would pass the second data set via switchData property.

```html
<bar-chart
	originalData='[{"date":"04","value":"100"}, {"date":"05","value":"144"}{"date":"06","value":"123"}]'
    switchData='[{"date":"03","value":"100"}, {"date":"09","value":"124"}, {"date":"11","value":"189"}]'>
</bar-chart>
```

###Roundup
This is an initial look into what can be done with D3.js and Polymer.
We can implement different d3 visualizations and create the corresponding elements from there on.
We might have a set of elements like &lt;pie-chart&gt; or &lt;stacked-area-chart&gt; making it easy to reuse elements over and over again
and quickly combining elements like legends with charts to create interesting data visualizations.
What should be considered is the fact that Polymer is in development and that therefore results might vary between browsers.

<p><iframe style="border: 1px solid #999; width: 100%; height: 550px; background-color: #fff;" src="http://embed.plnkr.co/XcTf5GmGLhAiVImAKHGQ" height="550" width="320" allowfullscreen="allowfullscreen" frameborder="0"></iframe></p>

###Links
[Polymer](http://www.polymer-project.org/)

[D3.js](http://d3js.org/)

[Difference between AngularJS directives and Polymer elements](http://stackoverflow.com/questions/18089075/what-is-the-difference-between-polymer-elements-and-angularjs-directives)

[Polymer – creating custom elements](http://www.polymer-project.org/docs/start/creatingelements.html)
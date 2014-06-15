---
layout: post
title:  "Testing D3.js With Jasmine"
date:   2014-06-15 12:00:00
tags:   D3.js, Javascript, Unit Tests, Karma, Jasmine
published: true
---

###Getting Started
Everyone who works with javascript and data visualization has to at some point come across **D3**.
[D3.js](http://d3js.org/) is a data driven javascript library for Dom manipulating, which comes with a large set of advanced features, enabling the quick calculation and rendering of very large data sets.

Or to quote the documentation: _"D3 allows you to bind arbitrary data to a Document Object Model (DOM), and then apply data-driven transformations to the document. For example, you can use D3 to generate an HTML table from an array of numbers. Or, use the same data to create an interactive SVG bar chart with smooth transitions and interaction."_

The [demos on the website](https://github.com/mbostock/d3/wiki/Gallery) present quick transformations and calculation involving very large sets of data.
While working with D3 is very well documented, there does not seem to be too much information about testing the generated elements.

We will actually only need a very basic configuration to get and up ready with testing D3.js data visualizations.

This post will assume previous knowledge of how to work with D3.js and basic api experience, as we will be focusing on the testing side of things.

We will be writing a basic bar chart and along the way add tests or rather try to see what we can really
test with **Jasmine** and what we might no be able to verify.

###Testing the SVG
We will start things off by creating a simple __svg__.

```javascript
var svg = d3.select('body').append('svg')
             .attr('height', '500')
             .attr('width', '500')
            .append('g')
             .attr("transform", "translate("0, 0")");
```

This code will not produce any visible results but if you inspect the dom you will see that the svg element has been created:

```html
<svg height="500" width="500">
	<g transform="translate(0, 0)"></g>
</svg>
```

We should rewrite the previous code to make it testable:

```javascript
function barChart() {
	var that = {};

	that.render = function() {
	   var svg = d3.select('body').append('svg')
	         .attr('height', '500')
	         .attr('width', '500')
	        .append('g')
	         .attr("transform", "translate(0, 0)");

	};

	return that;
}
```

We will simply wrap the previous SVG creation code into a _render_ method. This will enable us to call the _render_ method before every test
and also enables us to remove the SVG after the test has run.

Writing the first tests is easy. We can't test too much at this point, but we can definitely verify if the SVG has been created and if it contains the correct
height and width dimensions.

```javascript
describe('Test D3.js with jasmine ', function() {
  var c;

  beforeEach(function() {
    c = barChart();
    c.render();
  });

  afterEach(function() {
    d3.selectAll('svg').remove();
  });

  describe('the svg' ,function() {
    it('should be created', function() {
        expect(getSvg()).not.toBeNull();
    });

    it('should have the correct height', function() {
      expect(getSvg().attr('width')).toBe('500');
    });

    it('should have the correct width', function() {
      expect(getSvg().attr('width')).toBe('500');
    });
  });

  function getSvg() {
    return d3.select('svg');
  }

});
```

We are able to test a very basic render method already.

###Testing the data
We want to create a basic bar chart, so we will work with simple data objects containing a month and a value for that month.
For example our data might look like this: _{ date : '2014-01', value : 100 }_

First let us add a possibility to set data via simple setter/getter :

```javascript
function barChart() {

	// previous code...

	var data = null;

	that.setData = function(d) {
		data = d;
	};

	that.getData = function() {
		return data;
	}
```

And we can simply test the setter and getter:

```javascript
describe('Test D3.js with jasmine ', function() {
	// previous tests...

   describe('working with data' ,function() {
    it('should be null if no data has been specified', function() {
        expect(c.getData()).toBeNull();
    });

    it('should be able to update the data', function() {
      var testData =  [{ date: '2014-01', value: 100}, {date: '2014-02', value: 215}];
      c.setData(testData);
      expect(c.getData()).toBe(testData);
    });
  });
});
```

Currently we can create an SVG and add data. In the next section we will start creating the bar elements and verify that the elements have the correct
width, height and other properties.

###Testing elements
As soon as we have the data loaded and the base SVG element has been created, we can now start focusing on creating the bar elements.
Up until now we have had no visual feedback, this will change after this section, as we should be able see the bars soon.
We will need to restructure the previous code a little bit.

```javascript
function barChart() {
	var that = {};
	var data = null;
	var h = 500 - 80, w = 500, svg, x, y;

	// setter, getter etc.

	that.render = function() {

		// previous code ...

		x = d3.scale.ordinal().rangeRoundBands([0, w], .05);
		x.domain(data.map(function(d) {
		  return d.date;
		}));

		y = d3.scale.linear().range([h, 0]);
		y.domain([0, d3.max(data, function(d) {
		  return d.value;
		})]);

		// add bars

		var bars = svg.selectAll('.bar').data(this.getData());
		bars
			.enter().append('rect')
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

    // etc.
```

This code will create the bars according to the given data (we still do not have any axis), but for testing the bars we can already
add a new set of tests.

```javascript

// extend beforeEach to load the correct data...
beforeEach(function() {
	var testData =  [{ date: '2014-01', value: 100}, { date: '2014-02', value: 140}, {date: '2014-03', value: 215}];
	c = barChart();
	c.setData(testData);
	c.render();
});

describe('create bars' ,function() {
	it('should render the correct number of bars', function() {
		expect(getBars().length).toBe(3);
	});

	it('should render the bars with correct height', function() {
		expect(d3.select(getBars()[0]).attr('height')).toBeCloseTo(420);
	});

	it('should render the bars with correct x', function() {
		expect(d3.select(getBars()[0]).attr('x')).toBeCloseTo(9);
	});

	it('should render the bars with correct y', function() {
		expect(d3.select(getBars()[0]).attr('y')).toBeCloseTo(0);
	});
});

// added a simple helper method for finding the bars..
function getBars() {
	return d3.selectAll('rect.bar')[0];
}

```

Next, we will add the x and y axis.

```javascript
that.addAxis = function() {
	  // add axis
	svg.append("g")
	  .attr("class", "x axis")
	  .attr("transform", "translate(0," + h + ")");

	svg.append("g")
	  .attr("class", "y axis");

	var xAxis = d3.svg.axis()
	.scale(x)
	.orient("bottom");

	var yAxis = d3.svg.axis()
	.scale(y)
	.orient("left");

	svg.selectAll("g.x.axis")
	  .call(xAxis);

	svg.selectAll("g.y.axis")
	  .call(yAxis);
};
```

We can also write the x and y axis tests:

```javascript

// assert that the x and y axis have been created
describe('test Axis creation', function() {
	it('should create a xAxis', function() {
	     var axis = getXAxis();
	     expect(axis.length).toBe(1);
	});

	it('should create a yAxis', function() {
	   var axis = getYAxis();
	   expect(axis.length).toBe(1);
	});
});

// add axis helpers...
function getXAxis() {
	return d3.selectAll('g.x.axis')[0];
}

function getYAxis() {
	return d3.selectAll('g.y.axis')[0];
}
```


###Testing transitions
The bar chart already renders correctly upon given the correct set of data.
Further all tests should be up and running and passing successfully.

Finally we will also try to build some transitions and also test them.
Jasmine offers helpful methods for async operations like _waitFor_, _wait_ and _runs_.

We will use the _waits_ method to delay the test being executed. By defining _waits(1000)_ and wrapping the expected result inside the _runs_ method
 we can make sure that the transition has already ended. Also see the documentation for more information on [asynchronous specs in jasmine](https://github.com/pivotal/jasmine/wiki/Asynchronous-specs)

In our example implementation we defined a delay time of 500ms, for the test we will set waits to 1000ms.
If you want to see the chart being created on every test, you could also set the wait up to f.e. 3000ms, to really see the chart being created and destroyed again.

We should also add a simple button somewhere, as we will test a transition by simply setting new data as well as simulating a button click.

```html
<button id="switch">Switch Data</button>
```

Further we need an on click listener which might simply set a new data set, or switch to the old data.

```javascript
// predefined ...
var oldData = [];
var newData = [];
var switched = false;

d3.select('#switch').on('click', function() {
    var data = switched? oldData : newData;
    c.setData(data);
    c.render();
    switched = !switched;
});
```

When clicking the button, you should see that a new SVG has been created with the correct data and correct number of bars.
But we are rather interested in creating a transition, which means we do not want to create new SVGs, rather operate
on the existing SVG and transform bars as soon as the data changes.
We will need to refactor the _render()_ method.

First let us adapt the barChart function, we should add margins and create the SVG right from the start:

```javascript
function barChart() {

	var that = {};
	var data = null;
	// add margin definitions
	var margin = {
		top: 40,
		right: 0,
		bottom: 40,
		left: 40
	};

	// calculate the correct height and width dimensions
	var h = 500 - margin.top - margin.bottom, w = 500 - margin.top, x, y;

	var svg = d3.select('body').append('svg')
		.attr('height', '500')
		.attr('width', '500')
		.append('g')
		.attr("transform", "translate(" + margin.left + ", " + margin. top +")");

	// add axis and add them only once. moved from addAxis method into quasi the init.
    svg.append("g")
        .attr("class", "x axis")
        .attr("transform", "translate(0," + h + ")");

    svg.append("g")
        .attr("class", "y axis");

	// the setter/getter, render method etc...
```

The good news is that all tests will run and that we only operate on a single SVG.
The axis already changes when clicking the button. But we still need to adjust the bar part.
Currently the bars will not update. We will rewrite the code, by creating an _updateBars_ method.
This will take care of selecting all bars, updating the data and removing any bars if necessary.

```javascript

that.updateBars = function() {
	// add bars
	var bars = svg.selectAll('.bar').data(this.getData());

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

```

By adding _transition_ and _duration_ to the bars selection, any bar specific test will fail because we expect certain elements to be there
when executing the tests. This can be easily fixed by making Jasmine wait for a defined period of time.
As previously mentioned, the _waits_ method will enable us to wait until the the transition has finished before we assert certain elements to exist.

```javascript
// simple helper that wraps any expects into a runs method
// runs will be executed as soon as waits has expired.
function waitForElement(fn, time) {
	time = time || 1000;
	waits(time);
	runs(fn);
}
```

And now we can simply adapt the bars specific test code:

```javascript
describe('create bars' ,function() {
	it('should render the correct number of bars', function() {
		waitForElement(function() {
			expect(getBars().length).toBe(3);
		});
	});

	it('should render the bars with correct height', function() {
		waitForElement(function() {
			expect(d3.select(getBars()[0]).attr('height')).toBeCloseTo(420);
		});
	});

	it('should render the bars with correct x', function() {
		waitForElement(function() {
			expect(d3.select(getBars()[0]).attr('x')).toBeCloseTo(9);
		});
	});

	it('should render the bars with correct y', function() {
		waitForElement(function() {
			expect(d3.select(getBars()[0]).attr('y')).toBeCloseTo(0);
		});
	});
});
```

All tests should run successfully and the chart already transforms on button click.

[Here is the current code for the bar chart](https://gist.github.com/anonymous/d43a0a06d5c60773ea7b). We might consider adding an _init_ function and we
 could also have _setData_ automatically call the _render()_ method instead of having to call it explicitly.
This post focuses on the testing side of things, so we will leave it at this for now.
See the [chart in action](http://jsbin.com/pifulavu/7/edit) or check the complete [code](https://gist.github.com/anonymous/d43a0a06d5c60773ea7b).

<a class="jsbin-embed" href="http://jsbin.com/pifulavu/7/embed?output">JS Bin</a><script src="http://static.jsbin.com/js/embed.js"></script>

We still have to add one more transformation specific test as we need to verify that the bars have changed as soon as the data has been updated (via setData or on button click).

```javascript
describe('test transformation', function() {
	it('should update bars when data has changed', function() {
		var newData =  [{ date: '2014-02', value: 122}, {date: '2014-03', value: 145}];
		c.setData(newData);
		c.render();

		waitForElement(function() {
			expect(getBars().length).toBe(2);
		}, 2000);
	});
});
```

The test is self explanatory, we simply set a new set of data and call the _render_ method.
We know that the original number of bars should be three because the _beforeEach_ method already set an array consisting of three data objects.
The new data set only consists of two objects meaning that we should also be able to verify that two bars have been created.
The wait time has been set to 2000ms this time, just to make sure everything is finished when we assert the expectations.

###Round up
This post should be an introductory in testing D3 visuals with Jasmine.
There might be other tools that do a better job at testing the visuals, it is highly advised to search for more information
on D3 and testing or simply experiment with the possibilities and adapt what works best.


###Links
[D3.js](http://d3js.org/) The D3 library itself, including extensive documentation and examples

[Jasmine](http://jasmine.github.io/) The Jasmine testing framework

[Karma](http://karma-runner.github.io/0.12/index.html) Karma is a test runner, in case you need test automation



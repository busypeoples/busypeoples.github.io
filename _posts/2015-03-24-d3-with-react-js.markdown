---
layout: post
title: "Reusable Chart Components with React And D3"
date:   2015-03-24 12:00:00
tags:   React, D3, Javascript, web components
published: true
---

### Introduction

The following should be an introduction to combining **D3** with **React** to create reusable chart components.
This is not intended to be an introduction into D3 nor React, there is a large number of resources to help getting off the ground
with either frameworks, for example [this](http://d3js.org/) for D3 and [this](https://facebook.github.io/react/) for React.

D3s approach to data visualization fits well with the React way of building UI components and App structuring.
React encourages to figure out how to structure a number of components to enforce a data flow that moves from top down, meaning
that lower level components receive data and render it at best and only keep state if needed, but never manipulating any data that might 
affect the higher up components.

Another strong correlation between the libraries is their respective component lifecycle. D3 has **enter**, **update** and **exit**.

React has **componentWillUpdate**, **componentDidUpdate** and **componentWillUnmount**, enabling us to map the D3 lifecycle directly to the React one.

The following is just an experiment to see how far we can replace the D3 rendering with React. 

### Rendering 

We will build a simple bar chart that consists of a number of rect elements.
The basic building blocks for the example consist of a basic App container that composes a Chart component, setting the original height and 
width and passing the current data via props.

```html
var Chart = React.createClass({
    render: function() {
        return (
            {% raw %} <svg width={this.props.width} 
                 height={this.props.height} >
              {this.props.children}
            </svg>{% endraw %} 
        );
    }
});


// some basic data
var all = [/* some data */];
var filtered = [/* some data */];
    
    
var App = React.createClass({
    getDefaultProps: function() {
        return {
          width: 500,
          height: 500
        }
    },
    
    getInitialState: function() {
        return {
          data: all
        }
    },
    
    showAll: function() {
      this.setState({data : all})
    },
    
    filter: function() {
      this.setState({data: filtered});
    },
    
    render: function() {
        return (
          <div>
            <div className="selection">
              <ul>
                <li onClick={this.showAll}>All</li>
                <li onClick={this.filter}>Filter</li>
              </ul>
            </div>
            <hr/>
            <Chart width={this.props.width} 
                   height={this.props.height}>
              <Bar data={this.state.data} 
                          width={this.props.width} 
                          height={this.props.height} />
            </Chart>
          </div>
        );
    }
});
 
React.render(
    <App /> ,
    document.getElementById('app')
);


```

An initial look at the Chart component will reveal nothing special here. We are simply passing in the height, width and data via props.
The Chart component doesn't do anything special here. The only noteworthy aspect here is that we are creating the svg element.

```html
var Chart = React.createClass({
    render: function() {
        return (
            <svg width={this.props.width} 
                 height={this.props.height} >
              {this.props.children}
            </svg>
        );
    }
});

```

The most interesting part in the initial example is the Bar component code.
This is where D3 and React connect for the first time. 

To prevent any re-rendering we added the _shouldComponentUpdate_ method, which 
compares the current data with the upcoming data. In case the data has not changed, we simply return false and 
skip the rendering. Think in a D3 context, when the data has not changed, D3 simply doesn't do anything either.

The main action takes place inside the _render_ function, where we use D3 to define the needed _y-_ and _x-Scale_
and calculate the expected _x_, _y_, _height_ and _width_ values for every single rect element we want to render.
We create an array of rect elements and wrap them inside a g element.


```html

var Bar = React.createClass({
  getDefaultProps: function() {
    return {
      data: []
    }
  },

  shouldComponentUpdate: function(nextProps) {
      return this.props.data !== nextProps.data;
  },

  render: function() {
    var props = this.props;
    var data = props.data.map(function(d) {
      return d.y;
    });

    var yScale = d3.scale.linear()
      .domain([0, d3.max(data)])
      .range([0, this.props.height]);

    var xScale = d3.scale.ordinal()
      .domain(d3.range(this.props.data.length))
      .rangeRoundBands([0, this.props.width], 0.05);

    var bars = data.map(function(point, i) {
      var height = yScale(point),
          y = props.height - height,
          width = xScale.rangeBand(),
          x = xScale(i);

      return (
        <Rect height={height} 
              width={width} 
              x={x} 
              y={y} 
              key={i} />
      )
    });

    return (
          <g>{bars}</g>
    );
  }
});    

```

We have already replaced React with D3 at this point, meaning that React now renders the given data and D3 supplies
the needed functions to calculate the proper properties needed for calculating the height, width and so on and so forth.

Up until now we have the Bar component ready to render the Rect elements.
The Rect class code is very straight forward actually, all we need to do is return the actual rect element with provided 
properties.

```html
var Rect = React.createClass({
    getDefaultProps: function() {
        return {
            width: 0,
            height: 0,
            x: 0,
            y: 0
        }
    },
    
    shouldComponentUpdate: function(nextProps) {
      return this.props.height !== nextProps.height;
    },
    
    render: function() {
        return (
          <rect className="bar"
                height={this.props.height} 
                y={this.props.y} 
                width={this.props.width}
                x={this.props.x}
          >
          </rect>
        );
    },
});

```

This was a proof of concept of replacing the original D3 chart rendering part with React.
The code needs more refining obviously, including adding propTypes for example.

One final aspect that needs covering is the transition part. We need a way to simulate the original D3 transitions when the data changes.

### Adding Transitions

React offers the possibility to add _setInterval_ via the lifecycle method _componentDidMount_.
We will use this aforementioned method to add an interval that updates the state at a given interval enabling us to calculate
the current height at every iteration.

We will also use a simple mixin lifted from the [React documentation](https://facebook.github.io/react/docs/reusable-components.html)
that handles intervals and removing the intervals when the component is unmounted.

Further more we will need to keep state of the current height and compare it with the via props defined height. 
As long as current state height is not equal to the defined props height we will keep updating the state, triggering a re-rendering of the Rect component a long the way.

This approach enables us to create defined transitions. For smoothing out the transition we also added the _d3.ease_ method when calculating the
height. This makes the transition more natural as opposed to a linear transformation.

For the complete implementation check the code below.

```html
var SetIntervalMixin = {
  componentWillMount: function() {
    this.intervals = [];
  },
  setInterval: function() {
    this.intervals.push(setInterval.apply(null, arguments));
  },
  componentWillUnmount: function() {
    this.intervals.map(clearInterval);
  }
};

var Rect = React.createClass({
    mixins: [SetIntervalMixin], 
    getDefaultProps: function() {
        return {
            width: 0,
            height: 0,
            x: 0,
            y: 0
        }
    },
    
    getInitialState: function() {
      return {
        milliseconds: 0,
        height: 0
      };
    },
    
    shouldComponentUpdate: function(nextProps) {
      return this.props.height !== this.state.height;
    },
    
    componentWillMount: function() {
      console.log('will mount');
    },
    
    componentWillReceiveProps: function(nextProps) {
      this.setState({milliseconds: 0, height: this.props.height});
    },
    
    componentDidMount: function() {
      this.setInterval(this.tick, 10);
    },
    
    tick: function(start) {
      this.setState({milliseconds: this.state.milliseconds + 10});
    },
    
    render: function() {
      var easyeasy = d3.ease('back-out');
      var height = this.state.height + (this.props.height - this.state.height) * easyeasy(Math.min(1, this.state.milliseconds/1000));
      var y = this.props.height - height + this.props.y;
        return (
          <rect className="bar"
                height={height} 
                y={y} 
                width={this.props.width}
                x={this.props.x}
          >
          </rect>
        );
    },
});

```

Here is the final example including the code.

<p><iframe style="border: 1px solid #999; width: 100%; height: 750px; background-color: #fff;" src="http://embed.plnkr.co/sHShqy/preview" height="550" width="320" allowfullscreen="allowfullscreen" frameborder="0"></iframe></p>

### Roundup

This was an initial approach in combining D3 and React and taking control of the rendering process.
The provided example doesn't cover any transitions where the number of Rect element changes. This can be easily achieved by
adding x and width property calculations just like shown with the y and height properties.


Thank you to [@thinkfunctional](https://twitter.com/thinkfunctional) for reading the draft and providing valuable feedback.

### Links

[React] (https://facebook.github.io/react/)

[D3](http://d3js.org/)

[D3 and React - the future of charting components?](http://10consulting.com/2014/02/19/d3-plus-reactjs-for-charting/)

[Integrating D3.js visualizations in a React app](http://nicolashery.com/integrating-d3js-visualizations-in-a-react-app/)

[Ways of Integrating React.js and D3](http://ahmadchatha.com/writings/article1.html)

[SetIntervalMixin](https://facebook.github.io/react/docs/reusable-components.html)




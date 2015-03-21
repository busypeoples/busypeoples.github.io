---
layout: post
title:  "Understanding the React Component Lifecycle"
date:   2015-03-21 12:00:00
tags:   React, Javascript
published: true
---

### Introduction

**React** enables to create components by invoking the React.createClass() method which expects a _render_ method and triggers 
a lifecycle that can be hooked into via a number of so called lifecycle methods.

This short article should shed light into all the applicable functions.

Understanding the component lifecycle will enable you to perform certain actions when a component is created or destroyed.
Further more it gives you the opportunity to decide if a component should be updated in the first place and to react to 
props or state changes accordingly.

### The Lifecycle

To get a clear idea of the lifecycle we will need to differentiate between the **initial** creation phase, 
where the component is created, and state and props changes triggered **updates** as well as the component **unmoutings** phase.

#### Initialization

<img src="../../img/lifecycle_init.png" width="600px" alt="Initialization Lifecycle" title="Initialization Lifecycle">

From looking at the image above we can see that the first two methods being called are _getDefaultProps_ and _getInitialState_.
Both methods are only called once when initially rendering the component.

The **_getInitialState_** method enables to set the initial state value, that is accessible inside the component via *this.state*.

```javascript
getInitialState: function(){
    return { /* something here */};
}
```

Analogously **_getDefaultProps_** can be used to define any default props which can be accessed via *this.props*.

```javascript
getDefaultProps: function(){
    return { /* something here */};
}
```

Another two methods that only get called when initializing a component are _componentWillMount_ and _componentDidMount_.

**_componentWillMount_** is called before the _render_ method is executed. 
It is important to note that setting the state in this phase will not trigger a re-rendering.

The **_render_** method returns the needed component markup, which can be a single child component or null or false (in case you don't want 
any rendering).

This is the part of the lifecycle where props and state values are interpreted to create the correct output.
Neither props nor state should should be modified inside this function. This is important to remember, as by definition
the _render_ function has to be pure, meaning that the same result is returned every time the method is invoked.

As soon as the render method has been executed the **_componentDidMount_** function is called. 
The DOM can be accessed in this method, enabling to define DOM manipulations or data fetching operations.
Any DOM interactions should always happen in this phase not inside the _render_ method. 

#### State Changes

State changes will trigger a number of methods to hook into. 

<img src="../../img/lifecycle_state.png" width="600px" alt="State Changes Lifecycle" title="State Changes Lifecycle">

**_shouldComponentUpdate_** is always called before the _render_ method and enables to define if a re-rendering is needed or
can be skipped. Obviously this method is never called on initial rendering. A boolean value must be returned.

```javascript
shouldComponentUpdate: function(nextProps, nextState){
    // return a boolean value
    return true;
}
```

Access to the upcoming as well as the current props and state ensure that possible changes can be detected
to determine if a rendering is needed or not.

**_componentWillUpdate_** gets called as soon as the the _shouldComponentUpdate_ returned true. Any state changes via this.setState are
not allowed as this method should be strictly used to prepare for an upcoming update not trigger an update itself.

```javascript
componentWillUpdate: function(nextProps, nextState){
    // perform any preparations for an upcoming update
}
```
Finally **_componentDidUpdate_** is called after the _render_ method. Similar to the _componentDidMount_, this method can be 
used to perform DOM operations after the data has been updated.

```javascript
componentDidUpdate: function(prevProps, prevState){
    // 
}
```

Check the [code](http://plnkr.co/edit/t3if5Z1hDapXFp4O5QPa) example for more detailed insights. 

#### Props Changes

Any changes on the props object will also trigger the lifecycle and is almost identical to the 
state change with one additional method being called. 

<img src="../../img/lifecycle_props.png" width="600px" alt="Props Changes Lifecycle" title="Props Changes Lifecycle">

**_componentWillReceiveProps_** is only called when the props have changed
and when this is not an initial rendering. 
_componentWillReceiveProps_ enables to update the state depending on the existing and upcoming props, 
without triggering another rendering. One interesting thing to remember here is that there is no equivalent method for the state as state changes 
should never trigger any props changes.

```javascript
componentWillReceiveProps: function(nextProps) {
  this.setState({
    // set something 
  });
}
```

The rest of the lifecycle reveals nothing new here and is identical to the state change triggered cycle.

### Unmounting

<img src="../../img/lifecycle_unmount.png" width="600px" alt="Unmount Lifecycle" title="Unmount Lifecycle">

The only method we haven't touched yet is the _componentWillUnmount_ which gets called before the component is removed 
from the DOM. This method can be beneficial when needing to perform clean up operations, f.e. removing any timers defined in _componentDidMount_.

Check the [code](http://plnkr.co/edit/t3if5Z1hDapXFp4O5QPa) example for a more detailed insight. 

<p><iframe style="border: 1px solid #999; width: 100%; height: 350px; background-color: #fff;" src="http://embed.plnkr.co/0cN0tu/script.jsx" height="550" width="320" allowfullscreen="allowfullscreen" frameborder="0"></iframe></p>


### Links
[React Component Lifecycle Documentation](http://facebook.github.io/react/docs/component-specs.html)








# Intelligent Mixins With Backbone

A common problem you will face when developing Backbone applications is deciding
where to put shared logic. At first blush, inheritance (via `extend`) can solve
most of your problems. When you have a group of similar classes, simply make a
common ancenstor and have them all inherit it. But what happens when you have a
group of *unrelated* classes that need a similar feature? This is where the
[Mixin pattern](http://en.wikipedia.org/wiki/Mixin) becomes incredibly useful.

For the use of this article, we will be making a simple mixin that shows a pop-up
alert message with some text when a method is called.

## First Attempt

Our first foray into mixing in functionality to our views will be quite simple.
First, we will create an object to house our grouped functions, and then we will
attach it to our Backbone view:

```javascript
var alertMixin = {
    _alert: function(msg) {
        alert(msg);
    }
};
```

As you can see, the implementation of the mixin itself is very simple. It is
just a wrapper object housing our `_alert` method. Now, let's create a Backbone
view:

```javascript
var ViewOne = Backbone.View.extend({});
_.extend(ViewOne.prototype, alertMixin);
```

Our view is just an empty class, but the interesting part is the `_.extend`.
Here, we are adding all the properties of `alerts` onto the prototype of our
class. Now, whenever we make a new instance of `ViewOne` we will have access to
`_alert`.

```javascript
var view = new ViewOne();
view._alert('Hello World');
// Alert pops up with 'Hello World'
```

Right out the gate, this is very powerful, as we can keep adding onto the prototype
of `ViewOne` by adding more mixins to the extend.

```javascript
_.extend(ViewOne.prototype, MixinOne, MixinTwo /* ,  ... */);
```

## Second Attempt

One issue (depending on how you look at it) with the above method is that the mixins
are destructive. If two of your mixins implement the same method, the last
one wins, no matter what. We want to give the flexibility to the mixin to decide
if it will overwrite any methods that exist already, or extend them.

To do this, we will allow our mixins to be either objects or functions. The
objects will behave just as before, not taking any consideration into account
when extending the base view. The function mixin will get past the view it is
getting mixed into to perform conditional logic.

```javascript
var Mixin = function(view /*, mixins... */) {
    var mixins = _.toArray(arguments).slice(1);

    _.each(mixins, function(mixin){
        // If the mixin is a function, execute it and save it's return value
        var proto = _.isFunction(mixin) ? mixin(view) : mixin;

        // Check to see if the result of the mixin is an object
        if (proto && _.isObject(proto)) {
            _.extend(view.prototype, proto);
        }
    });
};
```

Let's take a look at what's happening here. We have a function that takes a view as its first
parameter and any number of things after it. The first line of the function
grabs the [tail](http://underscorejs.org/#rest) of the arguments array, which we
then iterate over. For each of these mixins we execute it (if it is a function)
or simply save its value. Finally, we extend our view with the restult of the
mixin. So, how would we use this method?

```javascript
var alertsOne = {
    _alert: function() {
        alert('I am an object mixin!');
    }
};

var alertsTwo = function(view) {
    var _alert = function() {
        alert('I am a function mixin!');
    };

    var oldAlert = view.prototype._alert;

    view.prototype._alert = oldAlert ? _.compose(_alert, oldAlert) : _alert;
};

Mixin(ViewOne, alertsOne, alertsTwo);
```

Now our mixins can be either functions or objects. In this case, the
function is actually checking to see if the `_alert` method exists on the views
prototype beforehand. If it does exist, we make a new method that is simply the
composition of the old method with the new method. Now, when `_alert` is called,
first `oldAlert` executes and then the new `_alert` executes. In this fashion,
the mixins are responsible for perserving methods (or not) if they desire.

## Third Attempt

Now, let's extend this concept a bit further. What if we wanted our alert mixin
to automatically pop up the alert one second after the view is initialized? With
our current pattern, we would need to bake that timer aspect into the
individual view as follows:

```javascript
var ViewOne = Backbone.View.extend({
    initialize: function() {
        var self = this;
        setTimeout(function() {
            self._alert('Hello World');
        }, 1000);
    }
});

var alertMixin = {
    _alert: function(msg) {
        alert(msg);
    }
};

Mixin(ViewOne, alertMixin);
```

The issue here is we have now increased the coupling of our view to the `alert`
mixin. In our third attempt, we will decrease the coupling of logic as well as
increase the cohesion of our mixin.

Another thing you may notice is the timing of the execution of our second attempt. The `Mixin` Method is called when the scripts are first parsed and
executed by the broweser. We would have substantially more flexibility if we
could defer this until the individual classes get initialized. To do this, we are
going to add a little bit of sugar to the base `Backbone.View` class:

```javascript
var OriginalBackboneView = Backbone.View;
var OriginalBackboneViewExtend = Backbone.View.extend;

Backbone.View = function(options) {
    var args = arguments;
    OriginalBackboneView.apply(this, args);

    // Standardize views to always add these options to the prototype
    options = options || {};
    this.mixins = options.mixins || this.mixins || [];

    Mixin.apply(this, arguments);

    this.delegateEvents();
};

_.extend(Backbone.View.prototype, OriginalBackboneView.prototype);
Backbone.View.extend = OriginalBackboneViewExtend;
```

Here, we are actually overwriting the original `Backbone.View` object with a new
class. We still call the original constructor; however we add in the check for a
`mixins` property on the view object. We then call our `Mixin` method, but
it will need some slight modifications:

```javascript
var Mixin = function() {
    var args = arguments;

    var mixins = (_.isFunction(this.mixins))
                    ? this.mixins.apply(this)
                    : this.mixins;

    var self = this;

    _.each(mixins, function(mixin){
        // If the mixin is a function, execute it and save it's return value
        var proto = _.isFunction(mixin) ? mixin.apply(self, args) : mixin;

        // Check to see if the result of the mixin is an object
        if (proto && _.isObject(proto)) {
            _.extend(self, proto);
        }
    });
};
```

Here, we have some subtle yet very important differences. First, we no longer
pass in the view class as a paramater. Instead we are passing the *instance* of
the view as the scope. What does this mean? It means our mixin function is now
the equivalent of a `Backbone.View`'s `initialize` method.

Second, we no longer pass in the mixins as a paramter to the `Mixin` method--instead, we look for a `mixin` property on the view object. With all this said
and done, though, how can we actually use this?

```javascript
var alertMixin = function(options) {
    console.log('mixin called');

    setTimeout(function() {
       this._alert('Hello World!');
    }.bind(this), 1000);

    return {
        _alert: function(msg) {
            alert(msg);
        }
    };

};

var ViewOne = Backbone.View.extend({
    mixins: [alertMixin]
});
```

This pattern gives you an incredible ammount of flexibility. For example, let's say you want
to pass options to your mixins. You can either make the mixin a high-order
function that accepts options and returns a mixin, or you could simply attach
options to your view and reference them from within the mixin.

## Moving Forward

At WillowTree, mixins have played very nicely with creating cohesive code. We
have several exciting mixins that we will be sharing over the coming weeks that
you can start using actively in your own Backbone projects.

## Demo

To run this code locally, simply clone the gist on
[GitHub](https://gist.github.com/kaw2k/8dca8655ed082d7c70da)



---
title: The Magic of Ember's `.get`
layout: post
category: ember
tags: ember
---

You've probably seen the `get` method in countless Ember examples and used it all over your code. How well do you really know it, though?

_I will be referencing the [Ember guides](http://emberjs.com/guides/), the [Ember API](http://emberjs.com/api/), and the [Ember source code](https://github.com/emberjs/ember.js) including quoted snippets. These were copied at the time of writing (Ember 1.8.1) and may have changed since._

---

The [Ember guides](http://emberjs.com/guides/object-model/classes-and-instances/#toc_initializing-instances) say the following about `get` and its sibling `set`:

> When accessing the properties of an object, use the `get` and `set` accessor methods [...] Make sure to use these accessor methods; otherwise, computed properties won't recalculate, observers won't fire, and templates won't update.

This reasoning explains the important of using `set` for setting properties, but glosses over why you might want to use `get` when reading properties. The `Ember.Observable` [API documentation](http://emberjs.com/api/classes/Ember.Observable.html#method_get) has more information:

> Retrieves the value of a property from the object.
>
> This method is usually similar to using `object[keyName]` or `object.keyName`, however it supports both computed properties and the unknownProperty handler.
>
> Because `get` unifies the syntax for accessing all these kinds of properties, it can make many refactorings easier, such as replacing a simple property with a computed property, or vice versa.

We'll look at these shortly.

---

# `Ember.Observable`'s `.get`

Most of the time you see calls to the `get` method in Ember applications, it's in the context of calling it on an object that uses `Ember.Observable` (such as an instance of `Ember.Object` or one of its subclasses).

The `fullName` computed property is a common [example](http://emberjs.com/guides/object-model/computed-properties/#toc_computed-properties-in-action) where `get` is used:

{% highlight javascript %}
App.Person = Ember.Object.extend({
    // these will be supplied by `create`
    firstName: null,
    lastName: null,

    fullName: function () {
        return this.get('firstName') + ' ' + this.get('lastName');
    }.property('firstName', 'lastName')
});
{% endhighlight %}

In this example, `fullName` looks up the values of the properties `firstName` and `lastName` on the `Person` instance (`this` in the function context) using `get`.

The example goes on to show how you how to look up the `fullName` property on an instance of this object:

{% highlight javascript %}
var ironMan = App.Person.create({
  firstName: "Tony",
  lastName:  "Stark"
});

ironMan.get('fullName'); // "Tony Stark"
{% endhighlight %}

Both of these are examples of `Ember.Observable`'s `get`. We can see how it works by looking at [its source code](https://github.com/emberjs/ember.js/blob/v1.8.1/packages/ember-runtime/lib/mixins/observable.js#L139-L141)[^1]:

[^1]: The actual ES6 source code imports `get` from `ember-metal/property_get`, but I've replaced it with the globals `Ember.get` call in the example for simplicity

{% highlight javascript %}
get: function(keyName) {
    return Ember.get(this, keyName);
},
{% endhighlight %}

Quite simply, it calls `Ember.get` with `this` as the first argument and passes along `keyName`. We'll take a look at the differences between calling `.get` on an object and calling `Ember.get` ourselves (passing in the object as the first argument) in the next section.

## Advantages of `get`

### Implementation Abstraction

When introducing colleagues or students to Ember.js, one of the most common points of confusion is understanding why to use `get` when in many cases direct access would work (*e.g.*, `this.get('firstName')` rather than `this.firstName`). The greatest benefit is the abstraction of whether the property is a static value or a computed one.

Let's take a look at the following contrived example:

{% highlight javascript %}
App.Person = Ember.Object.extend({
    // Person's height in inches, supplied by `create`
    height: null,

    /* ... */
});

var ironMan = App.Person.create({
    firstName: "Tony",
    lastName:  "Stark",
    height:    73
});
{% endhighlight %}

We have two ways of accessing this height programmatically:

{% highlight javascript %}
ironMan.height;        // 73
ironMan.get('height'); // 73
{% endhighlight %}

Why would I want to use the latter over the former? Let's expand our example and assume that Tony Stark decided to design a powered suit of armor. Clearly donning such a suit would change his height[^2]. Let's rewrite our implementation to support this suit by making height a computed property:

[^2]: [Iron Man (Anthony "Tony" Stark) &ndash; Marvel Comics Database](http://marvel.wikia.com/Iron_Man_%28Anthony_%22Tony%22_Stark%29)

{% highlight javascript %}
App.Person = Ember.Object.extend({
    // Person's height in inches, supplied by `create`
    naturalHeight: null,

    isWearingSuit: false,
    suitHeight: 5,

    height: function () {
        var height = this.get('naturalHeight');

        if (this.get('isWearingSuit')) {
            height += this.get('suitHeight');
        }

        return height;
    }.property('naturalHeight', 'suitHeight', 'isWearingSuit'),

    /* ... */
});

var ironMan = App.Person.create({
    firstName:     "Tony",
    lastName:      "Stark",
    naturalHeight: 73
});
{% endhighlight %}

How do our two programmatic solutions behave now?

{% highlight javascript %}
ironMan.height;        // undefined
ironMan.get('height'); // 73
{% endhighlight %}

Depending on the version of Ember you're using, you'll either get back `undefined` or the computed property function when you try accessing it directly.

In traditional JavaScript, you'd likely change height's implementation to be a getter, and expose it as something like `getHeight` which does the calculations necessary. However now you'd have to change the implementation of every single instance that accessed `height` to call `getHeight` instead.

What if `suitHeight` had to be loaded from or calculated elsewhere? Perhaps in the future we'll have to support multiple suits and `suitHeight` depends on which suit is currently being worn. We would just have to change the definition of `suitHeight` and our implementation of `height` remains the same.

Ember adheres to the [Uniform Access Principle](https://en.wikipedia.org/wiki/Uniform_access_principle) and lets you access all properties through the same notation (`get`), abstracting away the implementation of how the value is obtained (whether it's a static value, a computed function, or a cached value of a computed function). This allows us to change the underlying implementation later without having to change each place we access it[^3].

[^3]: You can find another good example of Ember and the Uniform Access Principle in [Robin Ward's comparison of EmberJS and AngularJS](http://eviltrout.com/2013/06/15/ember-vs-angular.html), specifically the sections "What’s wrong with Javascript primitives as models?", "Modeling this in AngularJS", and "Using getters and setters"

Switching between static values and computed values might seem like a rare occurence, but in reality it only gets more common as your applications get larger and more complex.

By always using `get` we don't have to worry about it.

#### Handling `null` and `undefined` Values

Assume we extend `App.Person` to include the ability to have a [POJO](https://en.wikipedia.org/wiki/Plain_Old_Java_Object) pet such as:

{% highlight javascript %}
App.Person = Ember.Object.extend({
    // Person's pet
    pet: null,

    /* ... */
});

var johnSmith = App.Person.create({
    firstName: "John",
    lastName:  "Smith",
    pet: {
        name: "Salazar",
        type: "snake",
        age:  3
    }
});
{% endhighlight %}

We can look up properties directly on this object using `.` as a path operator. For example, to get John's pet's name:

{% highlight javascript %}
johnSmith.pet;             // Object { name: "Salazar", type: "snake", age: 3 }
johnSmith.get('pet');      // Object { name: "Salazar", type: "snake", age: 3 }
johnSmith.pet.name;        // "Salazar"
johnSmith.get('pet').name; // "Salazar"
johnSmith.get('pet.name'); // "Salazar"
{% endhighlight %}

This all works as expected.

What about a similar person who has no pet?

{% highlight javascript %}
var janeDoe = App.Person.create({
    firstName: "Jane",
    lastName: "Doe",
});
{% endhighlight %}

Let's try to get her pet's name:

{% highlight javascript %}
janeDoe.pet;             // null
janeDoe.get('pet');      // null
janeDoe.pet.name;        // TypeError: Cannot read property 'name' of null
janeDoe.get('pet').name; // TypeError: Cannot read property 'name' of null
janeDoe.get('pet.name'); // null
{% endhighlight %}

Though this is another contrived example, I'm sure you can imagine situations where an object along a path is `undefined`, `null`, or otherwise missing. In this case, although (and because) `janeDone.pet` returns `null`, trying to get any properties on it results in a `TypeError`.

Notice the last example above, `janeDoe.get('pet.name')`, returns `null`. Ember's `get` can handle any object along the path being `null` or `undefined` and will return `null` or `undefined` (respectively) for the whole thing.

Not only does this simplify `null`/`undefined` checks, but also helps you deal with these edge cases in a more uniform way (no more having to catch `TypeError`s!).

# `Ember.get`

We previously saw that calling `.get` on objects that use `Ember.Observable` ends up calling a global `Ember.get` internally, passing the object as the first argument. You also have the ability to call `Ember.get` directly.

This gives us two advantages:

  1. We can take advantage of `get` with regular POJOs
  2. We can handle the object itself being `null` or `undefined`

## `get` With Regular POJOs

We can now use `Ember.get` to get the power of `get` in situations where we're dealing with POJO objects:

{% highlight javascript %}
var pet = johnSmith.get('pet');
pet.get('name');               // Uncaught TypeError: undefined is not a function
Ember.get(pet, 'name');        // "Salazar"
{% endhighlight %}

This lets us work with POJOs as easily as we do Ember objects, as well as abstracting away what types the objects they really are.

## `get` With a `null` or `undefined` Object

If we use the global `Ember.get`, we can simplify this even further. Consider the following:

{% highlight javascript %}
johnSmith.get('pet.name');        // "Salazar"
Ember.get(johnSmith, 'pet.name'); // "Salazar"

var johnSmith = null;
johnSmith.get('pet.name');        // Uncaught TypeError: Cannot read property 'get' of null
Ember.get(johnSmith, 'pet.name'); // undefined
{% endhighlight %}

By using the global `Ember.get`, we handle the case of the object itself being `null` or `undefined`.

We can now turn code that looks like:

{% highlight javascript %}
if (person && person.pet && person.pet.name) {
    // Do something
}
{% endhighlight %}

Into simply:

{% highlight javascript %}
if (Ember.get(person, 'pet.name')) {
    // Do something
}
{% endhighlight %}

---

# Extras

## `this.getProperties`

Sometimes you might want to get multiple properties at once. You can do this by calling `getProperties` and passing a list of property names as arguments (or an array of said names).

{% highlight javascript %}
tonyStark.getProperties("firstName", "lastName");   // { firstName: "Tony", lastName: "Stark" }
tonyStark.getProperties(["firstName", "lastName"]); // { firstName: "Tony", lastName: "Stark" }
{% endhighlight %}

## `this.getWithDefault`

Sometimes you might want to retrieve a property, but use a default value if that property is not available. It's quite common to use a technique such as the following:

{% highlight javascript %}
var defaultValue = 3,
    myProperty   = obj.get("someProperty") || defaultValue;
{% endhighlight %}

While concise, this format fails if the property value might end up being falsey (`false`, `0`, `-0`, `NaN`, `""`, `null`, `undefined`) as that would result in the default value being used instead. A much more robust solution is to take advantage of `getWithDefault`:

{% highlight javascript %}
var myProperty = obj.getWithDefault("someProperty", 3);
{% endhighlight %}

`getWithDefault` only uses the default value of the property lookup returns `undefined`, which is often exactly the behavior we wanted.

Unfortunately there is no globals `Ember.getWithDefault` available.

## `unknownProperty`

There is one further benefit to `get` that we have yet to discuss and that's `unknownProperty` and its sibling `setUnknownProperty`. These allow you to provide methods to be called when a nonexistant property is looked up (as part of a `set` or `get`) on an object and gives you a chance to manually provide/set values.

For example:

{% highlight javascript %}
App.UnknownPropertyExample = Ember.Object.extend({
    unknownProperty: function (keyName) {
        return keyName + " not found!";
    }
});

var example = App.UnknownPropertyExample.create();

example.get("anything"); // "anything not found!"
{% endhighlight %}

While it might lead to a useful and elegant design in [some situations](https://github.com/yapplabs/ember-buffered-proxy/), it tends to obscure where values are coming from or being set to, and thus shouldn't be used unless there's no other way to accomplish what you're looking for.

---

# Further Reading
- Ember documentation:
  - [Ember<wbr>#get](http://emberjs.com/api/classes/Ember.html#method_get)
  - [Ember.Observable<wbr>#get](http://emberjs.com/api/classes/Ember.Observable.html#method_get)
  - [Ember.Observable<wbr>#getProperties](http://emberjs.com/api/classes/Ember.Observable.html#method_getProperties)
  - [Ember.Observable<wbr>#getWithDefault](http://emberjs.com/api/classes/Ember.Observable.html#method_getWithDefault)

---

---
title: Performant Array Computed Properties
layout: post
category: ember
tags: ember
---

Take a look at the following examples of a `remainingTodos` array computed property:

{% highlight javascript %}
// Without prototype extensions -- will be using this style for the rest of the post
remainingTodos: Ember.computed('todos.@each.isDone', function () {
    var todos = this.get('todos');
    return todos.filterBy('isDone', false)
})

// With prototype extensions
remainingTodos: function () {
    var todos = this.get('todos');
    return todos.filterBy('isDone', false)
}.property('todos.@each.isDone')

// With a computed macro
remainingTodos: Ember.computed.filterBy('todos', 'isDone', false)
{% endhighlight %}

In this example we are filtering an array of `Todo`s to only those with an `isDone` of `false`. While this seems simple and correct (and indeed is), it has the potential to become a performance bottleneck!

In fact, this potential exists for almost every computed property that has an array as a dependency with `@each`. This is exacerbated in situations where the array is subjected to multiple stages of computed properties (sorting, filtering, one or more mappings).

One of my applications has to load thousands of rows of data and do some simple transformations, which should be no problem for a modern browser. Despite this we started receiving bug reports that our UI load times [exceeded 3 seconds in some cases](assets/images/performant-array-cps/slow.png), sometimes even (for some more extreme cases) taking several minutes or more! We tried reducing the number of items that were rendered at once (with paging or [`ember-list-view`](https://github.com/emberjs/list-view)) but that seemed to have no effect.

I asked about this on the [Ember Community Slack](https://ember-community-slackin.herokuapp.com/), and I was lucky to have [Martin Muñoz (mmun)](https://github.com/mmun) take the time to explain how array computed properties work and how to use them efficiently. After applying what we learned, we were able to reduce our UI load time to [under half a second in all but the most extreme cases](assets/images/performant-array-cps/fast.png) (of course, there's always more work to be done!). Much of this post is summarizing our discussion.

---

## Observers

Though this post is primarily focused on computed properties, `Ember.Observer`s use a similar dependency declaration and much of this applies to them, too. If you're using observers, though, you may want to consider [whether an observer is the right answer to your problem](https://www.youtube.com/watch?v=vvZEddrClAQ).

---

## Quick Review

Before we can discuss how to get the most out of your array computed properties, let's review the array dependency options we have available to us today and when they'd trigger recomputation.

If you're already familiar with array computed properties and the difference between `array.[]` and `array.length`, feel free to [skip ahead](#achieving-performance).

### `array`

The most basic is treating an array like any other property. This approach works great for most cases, but will only trigger a recomputation when the array object is changed. That is, the whole array has to be swapped out for another array.

### `array.length`

Sometimes knowing when an array is swapped out isn't enough; you also want to recompute when the length of the array changes. This is where `array.length` is helpful. This is useful if your property should update when the length of the array changes but the content of the array doesn't matter. One example of this is showing a total count ("You have 3 Todo items left").

The important thing to note is that this [won't trigger if the array length doesn't change](https://github.com/emberjs/ember.js/blob/v1.13.5/packages/ember-runtime/lib/mixins/enumerable.js#L1123-L1127), even if items are added or removed from the array.

### `array.[]`

And that's where `.[]` comes in. It does all of the above, but also guarantees it will trigger whenever items are added or removed from the array (even if the length doesn't end up changing).

### `array.@each.someProp`

For many properties, knowing when items are added/removed from the array doesn't cut it; you also need to know when properties on items in the array change. For example, you might be summing up a bunch of values, or providing a filtered version of the array (as we do in `remainingTodos` above).

In these cases we care about the array being swapped out, items being added to or removed from the array, and if a relevant property changes on any item in the array. We can use `.@each.someProp` to accomplish this.

Using `@each` is only supported once per dependency string, so you can't do something like `students.@each.assignments.@each.grade`.

### `array.@each.{someProp,anotherProp}`

Property brace expansion was [added in Ember 1.4](http://emberjs.com/blog/2014/02/12/ember-1-4-0-and-ember-1-5-0-beta-released.html#toc_property-brace-expansion) and provides a shorthand for observing multiple properties on the same object.

{% highlight javascript %}
// With property brace expansion
Ember.computed('array.@each.{foo,bar,baz}'

// Without property brace expansion
Ember.computed('array.@each.foo', 'array.@each.bar', 'array.@each.baz'
{% endhighlight %}

You can even do `{some,many}.really.{crazy,interesting}.things`, though you probably shouldn't.

### `array.@each`

You may occasionally see `@each` at the end of a dependency as a leaf property. This works and is equivalent in behavior to `array.[]`, but has much worse performance and should be avoided.

### Matrix

Let's take a look at some operations you might perform with arrays and which dependency patterns they'd trigger:

Operation | `array` | `array.length` | `array.[]` | `array.@each.foo`
--------- | :-----: | :------------: | :----------: | :---------------:
`this.set('array', newArray)` | ✔ | ✔ | ✔ | ✔
`array.pushObject(newObject)` | ✘ | ✔ | ✔ | ✔
`array.removeObject(newObject)` | ✘ | ✔ | ✔ | ✔
`array.setObjects(newArray)` | ✘ | ✔ | ✔ | ✔
`array.replace(0, 1, [ newObject ])` | ✘ | ✘ | ✔ | ✔
`array.set('firstObject.foo', 3)` | ✘ | ✘ | ✘ | ✔

---

## Achieving Performance

You don't have to concern yourself when depending on `array`, `array.length`, or `array.[]`. But if you're using `array.@each.someProp`, you may need to take one of the following steps to avoid a performance overhead if you expect a need to deal with very large arrays.

The key to performance when dealing with arrays is using the correct array dependencies for your application. Lots of work has been done to help reduce rendering work (*e.g.*  [`ember-list-view`](https://github.com/emberjs/list-view)), but sometimes the most intensive work is data processing. As it turns out, even when the operations you are conducting on the array are extremely efficient and fast, the overhead from attaching observers to each object in an array adds up and can quickly become nontrivial.

We can address this in one of two ways. Sometimes we can get away with a simpler dependency; other times we have to use the little-known `notifyPropertyChange`.

### Simplifying Dependencies

My initial mental model of computed property dependencies was that a computed property should depend on whatever values it uses in its computation. We saw this above with `remainingTodos` depending on the `isDone` property of every `Todo`.

This fit my understanding of a property, such as `isDone`, being changed in any way (a push from the back-end, someone opening Ember Inspector and tinkering with data, etc.) causing the UI to update and reflect the new state appropriately.

Unfortunately, while using `@each` to depend on properties that are part of the calculation seems "correct," this approach turns out to be naïve in some cases. This is especially true if your array goes through several transformation steps which use `@each`. In our application we had a `filter` step, a `sort` step, another `filter` step, and then several `map`s. The overhead from observing every item in every array at every step slowed our app to a crawl.

Consider how values in your application change under typical usage. How can the objects in the array change? How can their properties? Some properties might be user configurable, but you might find that some computed properties depend on data that does not change (during normal usage).

Our application, being a graph-based dashboard, was mostly visual representations of time series data. Only minimal aspects of our model were expected to change, and data was updated by a new data array replacing the old array. In this case, despite `array.@each.someProp` as a dependency feeling "correct," it was unnecessary as `array` worked just as well.

In our case, we were able to simplify our dependencies to avoid using `@each` for properties that never change independently of the object/array, and our app became fast again. The question to ask yourself is whether or not these properties will change under normal application usage.

Sometimes, however, normal application usage includes changing properties on objects in an array. For example, we'd expect `isDone` to change on `Todo` objects in the `todos` array. In this case, changing `isDone` should cause `remainingTodos` to recompute, and simplifying the dependency to just observe `todos` wouldn't work. In these circumstances, we can use `notifyPropertyChange`.

### Using `notifyPropertyChange`

Every class that implements `Ember.Observable` (including `Ember.Object`, `Ember.Array`, `Ember.Component`, and most Ember classes you're likely to be working with) comes with a very useful `notifyPropertyChange` method. This method allows you to, as the name suggests, trigger a change notification even if an object or property hasn't changed! In fact, [this is how `.[]` and `.length` work within Ember](https://github.com/emberjs/ember.js/blob/v1.13.5/packages/ember-runtime/lib/mixins/enumerable.js#L1123-L1127).

We can use this to avoid having to observe all the objects in an array for property changes and get away with depending only on `array` and get the performance benefits of not having to set up all those observers.

We can do this by, anywhere in our app where we change a property that should trigger recomputation, calling `this.notifyPropertyChange('array')`. For example, here's how we might implement some actions that handle `Todo` toggling:

{% highlight javascript %}
actions: {
    completeAll: function() {
        this.get('todos').setEach('isDone', true);
        this.notifyPropertyChange('todos');
    },
    toggleTodo: function (todo, isDone) {
        todo.set('isDone', isDone);
        this.notifyPropertyChange('todos');
    }
}
{% endhighlight %}

We can then rewrite our `remainingTodos` to avoid using `@each`:
{% highlight javascript %}
remainingTodos: Ember.computed('todos', function () {
    var todos = this.get('todos');
    return todos.filterBy('isDone', false)
})
{% endhighlight %}

While this approach can improve your performance, remember that it comes with some future refactoring risk:

 - You must be careful to always call `notifyPropertyChange` in each action you add or computed properties won't update.
 - It may cause computed properties which only care about the array being swapped out to needlessly recompute.

---

## Summary

If you're building a small- to medium-sized Ember.js application, you likely don't have to worry about array observer overhead and should choose dependencies based on what values a computed property uses in its calculation.

If your app deals with very large arrays, you may have to employ one of the more advanced techniques: simplifying dependencies to avoid `@each` or using `notifyPropertyChange`.

The difference between these two mental models is the change of perspective from asking "what is needed for the computation of this property?" to "what can a user change that would result in this property needing to be recomputed?"

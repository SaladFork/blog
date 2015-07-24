---
title: Performant Array Computed Properties
layout: post
category: ember
tags: ember
---

Take a look at the following examples of an array computed property:

{% highlight javascript %}
// With prototype extensions
remainingTodos: function () {
    var todos = this.get('todos');
    return todos.filterBy('isDone', false)
}.property('todos.@each.isDone')

// Without prototype extensions
remainingTodos: Ember.computed('todos.@each.isDone', function () {
    var todos = this.get('todos');
    return todos.filterBy('isDone', false)
})

// With a computed macro
remainingTodos: Ember.computed.filterBy('todos', 'isDone', false)
{% endhighlight %}

In this example we are filtering an array of Todos to only ones with an `isDone` of `false`. While this seems simple and correct (and indeed is), there is potential for it to become a significant performance bottleneck!

In fact, this potential exists for almost every computed property that has an array as a dependency. This is exacerbated in situations where the array is subjected to multiple stages of computed properties (sorting, filtering, one or more mappings).

One of my applications has to load thousands of rows of data and do some simple transformations, which should be no problem for a modern browser. We started receiving bug reports that our UI load times [exceeded 3 seconds in some cases](assets/images/performant-array-cps/slow.png), and even (for some more extreme cases) taking several minutes or more! We tried reducing the number of items that were rendered at once (with paging or [`ember-list-view`](https://github.com/emberjs/list-view)) but it seemed to have no effect.

I asked on the [Ember Community Slack](https://embercommunity.slack.com/)[^1] and was lucky to have [Martin Muñoz (mmun)](https://github.com/mmun) take the time to explain how array computed properties work and how to use them efficiently. After applying what we learned, we were able to reduce this load time to [under half a second in all but the most extreme cases](assets/images/performant-array-cps/fast.png) (of course, there's always more work to be done!). Much of this post is summarizing our discussion.

[^1]: You'll need to [get an invite](https://ember-community-slackin.herokuapp.com/) to join

To demonstrate this issue, I've created a simplified version of it in a JSBin:

`[- JSBin example showing fast and slow renders -]`


---

## Observers

Though this post is primarily focused on computed properties, `Ember.Observer`s also use a similar dependency declaration and much of this applies to them, too. Though if you're using observers you really should [rethink whether an observer is the right answer to your problem](https://www.youtube.com/watch?v=vvZEddrClAQ).

---

## Quick Review

Before we can discuss how to get the most out of your array computed properties, let's review the options we have available to us today and when they'd trigger recomputation.

### `array`

Only when the array object is changed. Whole thing has to be swapped out.

### `array.length`

All of the above (array swapped out), and also when the length of the array chagnes. This is useful if your property should update when the length of the array changes but doesn't care about the content of the array. One example of this is showing a total count ("You have 3 Todo items left").

It is important to note that this [won't trigger if the array length doesn't change](https://github.com/emberjs/ember.js/blob/896d667c3abd1dc9078b49c51d77d37b0b10e528/packages/ember-runtime/lib/mixins/enumerable.js#L978-L982).

### `array.[]`

And that's where `.[]` comes in. It does all of the above, but will trigger whenever an item is added or removed from the array (even if the length doesn't end up changing).

### `array.@each.someProp`

Can't do `@each.@each`

### `array.@each.{someProp,anotherProp}`

Property brace expansion was [added in Ember 1.4](http://emberjs.com/blog/2014/02/12/ember-1-4-0-and-ember-1-5-0-beta-released.html#toc_property-brace-expansion) and provides a shorthand for observing multiple properties on the same object.

{% highlight javascript %}
// With property brace expansion
.property('array.@each.{foo,bar,baz}')      // with prototype extensions
Ember.computed('array.@each.{foo,bar,baz}') // without prototype extensions

// Without property brace expansion
.property('array.@each.foo', 'array.@each.bar', 'array.@each.baz')
Ember.computed('array.@each.foo', 'array.@each.bar', 'array.@each.baz')
{% endhighlight %}

You can even do `{some,many}.really.{crazy,interesting}.things`, though you probably shouldn't.

### Matrix

Let's take a look at some operations you might perform with arrays and which dependency patterns they'd trigger:

Operation | `array` | `array.length` | `array.[]` | `array.@each.foo`
--------- | :-----: | :------------: | :----------: | :---------------:
`this.set('array', newArray)` | ✔ | ✔ | ✔ | ✔
`array.pushObject(newObject)` | ✘ | ✔ | ✔ | ✔
`array.removeObject(newObject)` | ✘ | ✔ | ✔ | ✔
`array.setObjects(newArray)` | ✘ | ✔ | ✔ | ✔
`array.replace(0, 1, newObject)` | ✘ | ✘ | ✔ | ✔
`array.set('firstObject.foo', 3)` | ✘ | ✘ | ✘ | ✔

---

## Achieving Performance

The key to performance when dealing with arrays is using the correct array dependencies. Lots of work done to reduce rendering work via [`ember-list-view`](https://github.com/emberjs/list-view), but sometimes the work isn't in the rendering but in data processing.

As it turns out, even when the operations you are conducting on the array are extremely efficient and fast, the overhead from attaching observers to each object for its properties can slow down your application to the point of being unusable.

`array` and `array.length` are usually OK, but others you need to consider how they're used.

Sometimes we can get away with a simpler dependency, other times have to use the little-known `notifyPropertyChange`.

### Simpifying Dependencies

Unfortunately while using `@each` all the time seems more "correct" and allows you to go to change any property and have the page update and remain valid, this approach is naïve.

Rule of thumb is whether or not these properties will change under normal application usage. 

Sometimes, however, normal application usage includes changing properties on objects in an array. For example, we'd expect `isDone` to change on `Todo` objects in the `todos` array.

### Using `notifyPropertyChange`

---

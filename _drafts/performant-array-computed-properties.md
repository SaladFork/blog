---
title: Performant Array Computed Properties
layout: post
category: ember
tags: ember
---

---

## Quick Review

Before we can discuss how to get the most out of your array computed properties, let's review the options we have available to us today.

### `array`

Only when the array object is changed. Whole thing has to be swapped out.

### `array.[]`

All of the above (array swapped out), and also when an item is added or removed from the array.

### `array.length`

Like `.[]` but *won't* trigger if the length doesn't change (items are swapped).

### `array.@each.someProp`

Can't do `@each.@each`

### `array.@each.{someProp,anotherProp}`

---

## Achieving Performance

The key to performance when dealing with arrays is using the correct array dependencies. Lots of work done by [@raytiley](https://github.com/raytiley) to reduce rendering work via `ember-list-view`, but sometimes the work isn't in the rendering but in data processing.

Sometimes we can get away with a simpler dependency, other times have to use `notifyPropertyChange`.

### Simpifying Dependencies

### Using `notifyPropertyChange`

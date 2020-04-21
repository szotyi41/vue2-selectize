# vue2-selectize by Szotyi41

A [Selectize](http://selectize.github.io/selectize.js/) wrapper for VueJS 2.

## New Features

# Functions

 * slideToggle <bool> - If its true the options will slide
 * disableTriggerOnChange <function> - If its called the onChange event will not called anymore
 * enableTriggerOnChange <function> - If its called the onChange event will callend again
 * onItemRemove <function(items, removedItem)> - This function called when an item removed
 * setFocus <function>
 * setBlur <function>
 * setValue <function (value)>
 * setOptions <function (array or object of option)>
 * addOptions <function (array or object of option)>
 * addOption <function (object of option)>
 * setItems <function (object of items, if its true its added when option is not exists)> - Remove all items and add
 * addItems <function (object of items, if its true its added when option is not exists)> - Just add items, not remove
 * addItem <function (item, if its true its added when option is not exists)>
 * addOptionsIfNotExists <function (options)>
 * addOptionIfNotExists <function (option)>
 * addItemAsOption <function (option)> - Add item (param will option not item) Option has value and text field
 * createOnEnter <bool> - Create will run when you press enter and text is not empty
 * createOnBlur <bool> - Create will run when you click outside and text is not empty
 * debug <bool> - Enable debug mode

# Variables

 * inputText - Text in input
 * element - Element of select
 * options - Options array
 * focus - If focused



## Prerequisites
* jQuery >= 1.7.0

## Installation

```js
npm install --save jquery vue2-selectize
```

## Usage

```html
<selectize v-model="selected" :settings="settings"> <!-- settings is optional -->
  <option :value="1">One</option>
  <option :value="2">Two</option>
</selectize>
```
```js
import Selectize from 'vue2-selectize'

export default {
  components: {
    Selectize
  },
  data() {
    return {
      settings: {}, // https://github.com/selectize/selectize.js/blob/master/docs/usage.md
      selected: 1
    }
  }
}
```
```scss
// Include your preferred theme in your main SASS file (or your component's <style lang="scss"/> section).

@import "~selectize/dist/css/selectize.bootstrap3.css";
```

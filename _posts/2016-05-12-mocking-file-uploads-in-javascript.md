---
layout: post
title: Mocking File Inputs in JavaScript and Ember.js
keywords: mocking, file input, file, javascript, sinon.js, qunit, stubs, filereader, components, ember.js, testing, integration
---

*Originally posted in [Intuo Engineering](http://nerds.intuo.io/2016/05/12/mocking-file-uploads-in-javascript.html).*

Testing integral parts of web browsers that rely on user interaction is always tricky and often requires a cumbersome implementation. File inputs are no exception. Consider the following JavaScript snippet that is triggered when we add a file to an input field.

```html
<script type="text/javascript">
  function handleFiles(files) {
    var file = files[0];

    // Do something with selected file
  }
</script>

<input type="file" onchange="handleFiles(this.files)" />
```

Unit-testing the `handleFiles` function is sufficient for some cases, but what happens if your goal is an integration test for an Ember.js Component? Manually selecting a file in every test build while the test awaits your input is *obviously* not a good solution. On the other hand, mocking the interaction by triggering events and stubbing methods on the HTML FileReader API is a very good alternative to manual labour. 11 out of 10 developers would agree ;)

### Setting up an Ember.js Component

*Note that the examples and logic are not neccesarilly limited to Ember.js, but are rather generic to each file input. Ember is just too awesome not to use it all the time.*

Let's define our Ember.js Component. Its sole purpose will be rendering of a file input field and listening on the `change` event, which will be triggered when the user selects a file.

```javascript
// app/components/x-file-input.js

import Ember from 'ember';

export default Ember.Component.extend({
  tagName: 'input',
  type: 'file',
  attributeBindings: ['type', 'value'],

  addChangeListenerToElement: Ember.on('didInsertElement', function() {
    let input = this.$()[0];

    input.onchange = (event) => {
      let file = event.target.files[0];

      // Handle file or bubble up the action with `sendAction`
    };
  }
});
```

*The above component is basically the Ember equivalent to the code snippet from before, only with a lot more elegance. Make sure to add a listener tied to the DOM element and not to the Ember Component for your `change` event, since this call won't be picked up by event triggering with custom targets later in our tests!*

Let's move on to mocking the file selection. The goal of our component is to get the file, read it and send the content as a Data URL to another part of our app, e.g. for persisting. So let's implement that.

```javascript
// app/components/x-file-input.js

import Ember from 'ember';

export default Ember.Component.extend({
  tagName: 'input',
  type: 'file',
  attributeBindings: ['type', 'value'],

  addChangeListenerToElement: Ember.on('didInsertElement', function() {
    let input = this.$()[0];

    input.onchange = (event) => {
      let file = event.target.files[0];
      let reader = new FileReader();
      let fileName = file.name;

      reader.onload = (event) => {
        this.sendAction('handleFileAsDataURL', fileName, event.target.result);
      };

      reader.readAsDataURL(file);
    };
  })
});
```

The code above is pretty explanatory. After the `FileReader` successfully reads our file, we send the name and the content of the file to our action outside of the component. Pretty simple, ain't it? But let's try to test it.

### Mocking file input in tests

Mocking an event target has a bunch of pitfalls and might not work correctly later on, mostly due to API changes and security reasons. However, if you register your listener for the `change` event directly on your DOM element using the `onchange` callback, it should work correctly. This is exactly what we did on our Ember Component.

So let's set up our test and see if it works. We will be using the awesome [ember-sinon](https://github.com/csantero/ember-sinon) library to create some spies for file handling.

```javascript
// tests/integration/components/x-file-input-test.js

import { moduleForComponent, test } from 'ember-qunit';
import hbs from 'htmlbars-inline-precompile';
import sinon from 'sinon';
import jQuery from 'jquery';

moduleForComponent('x-file-input', 'Integration | Component | x-file-input', {
  integration: true
});

test('correcly mocks file input values', function(assert) {
  let handleFile = sinon.spy();

  let fileName = 'surprise.png';
  let content = 'data:image/png;base64,no-surprises-here';

  this.set('handleFile', handleFile);
  this.render(hbs`
   {% raw %}{{x-file-input id="file-input" handleFileAsDataURL=(action handleFile)}}{% endraw %}
  `);

  fillInFileInput('#file-input', { name: fileName, content: content });

  assert.ok(handleFile.calledOnce, 'Action `handleFile` should be called once.');
  assert.ok(handleFile.calledWithExactly(fileName, content), 'Action `handleFile` should be called with exact arguments');
});
```

First we set up our `handleFile` action as a spy, which evaluates whether the action was called and which arguments it was called with. We expect this action to be called after `FileReader` finishes reading the content of the file. The most important part is the `fillInFileInput` function. Let's define it.

```javascript
function fillInFileInput(selector, file) {
  // Get the input
  let input = jQuery(selector);

  // Get out file options
  let { name, type, content } = file;

  // Create a custom event for change and inject target
  let event = jQuery.Event('change', {
    target: {
      files: [{
        name: name, type: type
      }]
    }
  });

  // Stub readAsDataURL function
  let stub = sinon.stub(FileReader.prototype, 'readAsDataURL', function() {
    this.onload({ target: { result: content }});
  });

  // Trigger event
  input.trigger(event);

  // We don't want FileReader to be stubbed for all eternity
  stub.restore();
}

```

Our `fillInFileInput` function takes a selector and a file options hash as arguments. The selector is used to build a jQuery representation of the input for triggering events. Then we define our own change event, which will inject our target and mock the file selection. Lastly, we stub every `FileReader` instance to immediately resolve the file content and trigger the change event.

There's also a way to omit stubs on `FileReader` completely and just try to use a `Blob` with your file content, but we haven't investigated that just yet. Go for it, if you want.

Lastly, but most importantly, be sure to check if you're directly setting the `onchange` on DOM element and not some wrapper on top of it, like jQuery or Ember.js. Otherwise the custom target in an event might not be working for you and you'll beat your head against the wall for a good couple of hours :)

That's all folks! Happy testing!

*Explore the code and tests at [Ember Twiddle](https://ember-twiddle.com/4e1fadfc3f5015fa297fc64473099377?openFiles=tests.integration.components.x-file-input-test.js%2C){:target="blank"}.*

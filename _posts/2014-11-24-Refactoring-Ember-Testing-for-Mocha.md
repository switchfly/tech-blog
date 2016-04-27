---
layout: post
title: Refactoring Ember Testing for Mocha ... and Beyond!
author: Dan Gebhardt
author-image: dgebhardt.jpg
summary: Switchfly undertook a refactoring of Ember testing libraries to make them test-framework-agnostic. This opens the door for QUnit, Mocha, and other test frameworks to be used with Ember and Ember CLI.

---

As a consultant with [Tilde](http://tilde.io), I've been working with Switchfly for over a year to help modernize and improve their suite of [Ember.js](http://emberjs.com/) applications. Switchfly was an early adopter of Ember and has long recognized the advantages of embracing convention over configuration ("CoC"). When you're free from trivial choices and able to focus on core functionality, you can develop solutions rapidly. And with a large engineering team working across multiple projects, you really appreciate conventions that flatten the learning curve for developers new to a project.

The most recent phase of modernization we've undertaken involves a transition to ES6 modules and [Ember CLI](http://www.ember-cli.com/). Ember CLI is part of the [Ember 2.0 roadmap](https://github.com/emberjs/rfcs/pull/15) and, together with ES6 modules, "will become first-class parts of the Ember experience". As Ember Core Team member [Tom Dale](https://twitter.com/tomdale) recommends: "You should begin moving your app to Ember CLI as soon as possible."

The hurdle we faced immediately in adopting Ember CLI was its singular support for the [QUnit](http://qunitjs.com/) test framework. Switchfly's Ember apps are all tested with [Mocha](http://mochajs.org/). Rather than looking at a move to QUnit as a costly necessity, we saw this as an opportunity to introduce a degree of freedom in Ember CLI. As mentioned in [Eric Berry's](https://twitter.com/coderberry) excellent [EmberConf 2014 talk on testing](https://speakerdeck.com/coderberry/the-unofficial-official-ember-testing-guide), support for multiple test frameworks has been on the Ember testing roadmap all along. We just happen to be the first ones to pave this particular cowpath.

## Extraction of Ember Test Helpers

Our end goal was to build a Mocha-specific version of the unit testing library [Ember QUnit](https://github.com/rwjblue/ember-qunit), as well as an Ember CLI addon for Mocha equivalent to [Ember CLI QUnit](https://github.com/rwjblue/ember-cli-qunit).

Instead of forking Ember QUnit and duplicating a lot of its logic, I consulted with [Robert Jackson](https://twitter.com/rwjblue) of the Ember Core Team about alternative approaches. We decided to extract the core logic in Ember QUnit into a separate library that in turn could be used by testing-library-specific wrappers. The result is [Ember Test Helpers](https://github.com/switchfly/ember-test-helpers).

Ember Test Helpers is a test-framework-agnostic set of helpers for unit testing Ember applications. Its primary purpose is to provide classes that represent testing modules - a common concept across test frameworks. There's a base class for general purpose unit testing (`TestModule`), as well as derived classes with custom logic for testing components (`TestModuleForComponent`) and models (`TestModuleForModel`).

These modules provide a container for testing a particular "unit" of your application in isolation. The modules set up the context of each test, providing helpers to access the subject of the test (`this.subject()`) and other dependencies (such as a store for model tests).

## Ember QUnit and Ember Mocha

The next step was to rewrite Ember QUnit to use Ember Test Helpers and ensure that it didn't lose any functionality. After Ember QUnit was stable again, we moved on to our final goal of writing [Ember Mocha](https://github.com/switchfly/ember-mocha).

The end result is that unit tests can be written in QUnit:

```
import { test, moduleForComponent } from 'ember-qunit';

moduleForComponent('gravatar-image', 'GravatarImageComponent', {
  // specify the other units that are required for this test
  // needs: ['component:foo', 'helper:bar']
});

test('it renders', function() {
  expect(2);

  // creates the component instance
  var component = this.subject();
  equal(component._state, 'preRender');

  // renders the component on the page
  this.render();
  equal(component._state, 'inDOM');
});
```

And equivalent tests can be written in Mocha:

```
import { describeComponent, it } from 'ember-mocha';

describeComponent(
  'gravatar-image',
  'GravatarImageComponent',
  {
    // specify the other units that are required for this test
    // needs: ['component:foo', 'helper:bar']
  },
  function() {
    it('renders', function() {
      // creates the component instance
      var component = this.subject();
      expect(component._state).to.equal('preRender');

      // renders the component on the page
      this.render();
      expect(component._state).to.equal('inDOM');
    });
  }
);

```

Please see the READMEs of each project for full usage details.

## Ember CLI Integration

Although Ember CLI comes with Ember QUnit enabled by default, and all of its testing blueprints were written for QUnit, it was straightforward to write an Ember CLI addon to override these defaults.

[Ember CLI Mocha](https://github.com/switchfly/ember-cli-mocha) contains Mocha-specific blueprints that replace all of the default tests in Ember CLI. After installation, all the standard generators (e.g. `ember generate component x-foo`) will create tests for Mocha instead of QUnit.

The instructions for removing QUnit and installing Ember CLI Mocha are included in the README.

## Composeability

Ember is often unfairly portrayed as "monolithic" when in fact the framework and its ecosystem are well architected into composeable layers. As our work on testing illustrates, it's possible to internally refactor these layers without breaking interfaces. I hope others will continue our work to support even more testing frameworks ( perhaps [Jasmine](http://jasmine.github.io/) ), and everyone will continue to introduce flexibility up and down the Ember stack.

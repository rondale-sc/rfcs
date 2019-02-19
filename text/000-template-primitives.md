- Start Date: (fill me in with today's date, YYYY-MM-DD)
- Relevant Team(s): (fill this in with the [team(s)](README.md#relevant-teams) to which this RFC applies)
- RFC PR: (after opening the RFC PR, update this with a link to it and update the file name)
- Tracking: (leave this empty)

# Template Primitives

## Summary

Expose low-level primitives for associating component templates and classes,
and customizing template scope.

These primitives unlock experimentation, allowing addons to provide
highly-requested features (such as single-file components) via stable, public
API.

## Motivation

This proposal is intended to unlock experimentation around two
highly-requested features:

1. Single-file components.
2. Module imports in component templates.

Although these features are the primary motivation, one benefit of
stabilizing low-level APIs is that they often enable developers to come up
with new, unexpected ideas.

### Single-File Components

In Ember components today, JavaScript code lives in a `.js` file and template
code lives in a separate `.hbs` file. Juggling between these two files adds
friction to the developer experience.

The template and JavaScript code are inherently coupled, and changes to one
are often accompanied by changes to the other. Separating them provides
little value in terms of improving reusability or composability.

Other component APIs eliminate this friction in different ways. React uses
JSX, which produces JavaScript values:

```js
export default class extends Component {
  state = { name: 'World' };

  render() {
    return <div>Hello {this.state.name}!</div>
  }
}
```

Vue defines the [`.vue` custom single-file component
format](https://vuejs.org/v2/guide/single-file-components.html):

```html
<template>
  <div>Hello {{name}}!</div>
</template>

<script>
export default {
  data() {
    return {
      name: 'World'
    }
  }
}
</script>
```

### Module Imports in Templates

One nice property of JSX is that it leverages JavaScript's existing scoping
semantics. React components are JavaScript values that can be imported and
referenced like any other JavaScript value. If an binding would be in scope
in JavaScript, it's in scope in JSX:

```js
import { Component } from 'react';
import OtherComponent from './OtherComponent';

export default class extends Component {
  render() {
    return (
      <div>
        /*
          OtherComponent is in scope because it was imported at the top of
          the file, so we know we can invoke it here as a component.
        */
        <OtherComponent />
      </div>
    );
  }
}
```

This compares favorably with Vue, where developers must learn a special
system for explicitly copying values from JavaScript scope into template
scope, and component names can be different between where they are imported
(`OtherComponent`) and where they are invoked (`other-component`):

```html
<template>
  <div>
    <other-component />
  </div>
</template>

<script>
import OtherComponent from './OtherComponent.vue'

export default {
  components: {
    OtherComponent
  }
}
</script>
```

Neither of these approaches is exactly right for Ember. Ideally, we'd find a
way to maintain the performance benefits of templates (like Vue), while
preserving the productivity and learnability benefits of JSX, where
JavaScript and "template" scope are one in the same.

### Unlocking Experimentation

Rather than deciding on the best syntax for single-file components in Ember
upfront, this RFC proposes a few new low-level APIs that addons could use to
experiment with different formats, ultimately compiling a given format into
JavaScript.

For example, imagine a file format that uses a frontmatter-like syntax to separate
template and JavaScript sections:

```hbs
---
import Component from '@glimmer/component';
import { inject as service } from '@ember/service';
import titleize from './helpers/titleize';
import BlogPost from './components/blog-post';

export default class MyComponent extends Component {
  @service session;
}
---

{{#let this.session.currentUser as |user|}}
  <BlogPost @title={{titleize @model.title}} @body={{@model.body}} @author={{user}} />
{{/let}}
```

An addon that supported this format would, at build time, transform the above
file into a JavaScript file that looks something like this:

```js
import Component from '@glimmer/component';
import { inject as service } from '@ember/service';
import titleize from './helpers/titleize';
import BlogPost from './components/blog-post';

import { createTemplateFactory } from '@ember/template-factory';

const template$1 = createTemplateFactory({
  "id": "ANJ73B7b",
  "block": "{\"statements\":[\"...\"]}",
  "meta": { "moduleName": "index.hbs" },
  "scope": () => ({ BlogPost, titleize })
});

class MyComponent extends Component {
  @service session;
}

setComponentTemplate(MyComponent, template);

export default MyComponent;
```

Note that the specific format shown here is just a hypothetical example. The
important thing is that we expose public JavaScript API for three things:

1. Compiling a template into JSON.
3. Associating a compiled template with its backing JavaScript class.
2. Specifying the values in scope for a given template at runtime.

By focusing on primitives first, we can iterate on different designs via
addons, before settling on a default format that we'd incorporate into the
framework.

## Detailed design

### Precompiling

It's important to understand how templates in Ember are packaged up and
turned into a format that can be run in the browser.

Glimmer supports two modes for compiling templates: an Ahead-of-Time (AOT)
mode where templates are compiled into binary bytecode, and a Just-in-Time
(JIT) mode where templates are compiled into an intermediate JSON, with final
bytecode compilation happening in the browser.

Ember uses Glimmer's JIT mode, so templates are sent to the browser as an
optimized JSON data structure. The `@ember/template-compiler` package provides
a helper function called `precompile` that turns raw template source code into
this "pre-compiled" JSON format.

```js
import { precompile } from '@ember/template-compiler';
const json = precompile(`<h1>{{this.firstName}}</h1>`);
```

In this proposal, we extend the `precompile` function to take an additional
parameter of scope.

### Template Scope

In a Glimmer template, "scope" refers to the list of items that can be
referenced. For example, when you type `{{@firstName}}`, `{{this.count}}`, or
`<UserAvatar />`, how do we figure out what each of those things refers to?
The answer to that depends on the template's scope.

Today, the scope of a template is controlled in one of three ways:

1. `this` is always in scope and refers to the component instance.
2. Arguments (like `@firstName`) are placed into scope when you invoke the component.
3. Identifiers that aren't `this` and don't start with `@` are looked up based on a series of lookup rules.

To allow the component 

--- 

How does Ember know which JavaScript class and template go together to define
a single component?

Today, the process of associating a component's template with its backing
class is not flexible enough to easily enable different file format experiments.

By default, templates are associated with a class implicitly by file name.
For example, the template at `app/templates/components/user-avatar.hbs` will
be joined with the class exported from `app/components/user-avatar.js`,
because they share the base file name `user-avatar`.

The benefit of this approach is that files are in predictable locations. If
you are editing a template and need to switch to the class, or are reading a
another template and want to see the implementation of the `{{user-avatar}}`
component, finding the right file to jump to should not be difficult.

Sometimes, though, Ember developers run into scenarios where they'd like to
re-use the same template across multiple components. While not widely
used, Ember supports overriding a component's template by defining the `layout`
property in the component class:

```js
import Component from '@ember/component';
import OtherTemplate from 'app/templates/other-template';

export default Component.extend({
  layout: OtherTemplate
})
```

Because memorizing the rules of how components and helpers are looked up can
be intimidating for new learners, we'd like to explore alternate APIs that
allow users to explicitly import values and refer to them from templates.

For example, a format for components could look something like this:

```
--- js ---
import Component from '@glimmer/component';
import t from 'ember-t-helper';
import UserAvatar from '../../UserAvatar';

export default class MyComponent {
  message = 'Hello world!';
}

--- hbs ---
```

Currently, it's not possible to write an addon that implements this file format using public APIs.
With this RFC, an addon could compile the above example into the following JavaScript:

```js
import Component from '@glimmer/component';
import t from 'ember-t-helper';
import UserAvatar from '../../UserAvatar';

export default class MyComponent {
  <template>
    <UserAvatar @greeting={{t this.message}} />
  </template>

  message = 'Hello world!';
}
```

In the first step, a build-time plugin analyzes this file and calls `preprocess` with the following arguments:

```js
import { precompile } from '@ember/template-helper';

const template = `<UserAvatar @greeting={{t this.message}} />`;
const scope = ['t', 'UserAvatar'];
const json = precompile(template, { scope });
```

Next, the original file is transformed to embed the wire format JSON returned from `precompile`:

```js
import Component, { setComponentTemplate, TemplateScope } from '@glimmer/component';
import t from 'ember-t-helper';
import UserAvatar from '../../UserAvatar';

class MyComponent {
  message = 'Hello world!';
}

const json = /* insert wire format JSON from previous step here */
// To reduce file size, identifiers can be elided by maintaining order of
// identifiers passed to `precompile` and scope array provided at runtime.
const scope = [() => t, () => UserAvatar];

setComponentTemplate(MyComponent, template, { scope }));

export default MyComponent;
```

```js
import Component, { setComponentTemplate, TemplateScope } from '@glimmer/component';
import { precompile } from '@ember/template-compiler';
import t from 'ember-t-helper';
import UserAvatar from '../../UserAvatar';

class MyComponent {
  message = 'Hello world!';
}

// The following two lines get replaced with the static JSON wire format output
// of the precompiler.
const template = `<UserAvatar @greeting={{t this.message}} />`;
const json = precompile(template, { scope: ['UserAvatar', 't'] });

const scope = TemplateScope({
  UserAvatar: () => UserAvatar,
  t: () => t
})


setComponentTemplate(MyComponent, precompile(template, { scope }));

export default MyComponent;
```

> This is the bulk of the RFC.

> Explain the design in enough detail for somebody
familiar with the framework to understand, and for somebody familiar with the
implementation to implement. This should get into specifics and corner-cases,
and include examples of how the feature is used. Any new terminology should be
defined here.

## How we teach this

> What names and terminology work best for these concepts and why? How is this
idea best presented? As a continuation of existing Ember patterns, or as a
wholly new one?

> Would the acceptance of this proposal mean the Ember guides must be
re-organized or altered? Does it change how Ember is taught to new users
at any level?

> How should this feature be introduced and taught to existing Ember
users?

## Drawbacks

> Why should we *not* do this? Please consider the impact on teaching Ember,
on the integration of this feature with other existing and planned features,
on the impact of the API churn on existing apps, etc.

> There are tradeoffs to choosing any path, please attempt to identify them here.

## Alternatives

> What other designs have been considered? What is the impact of not doing this?

> This section could also include prior art, that is, how other frameworks in the same domain have solved this problem.

## Unresolved questions

> Optional, but suggested for first drafts. What parts of the design are still
TBD?

- Start Date: 2019-02-22
- Relevant Team(s): Ember CLI / Fastboot
- RFC PR: (after opening the RFC PR, update this with a link to it and update the file name)
- Tracking:

# Enable Rehydration By Default

## Summary

Firstly, I'd like to take a brief minute to talk about what Rehydration is for the curious.  To do so I'm going to pull in some of the explanation from the commit that introduced it to the Glimmer VM:

> -- What is Rehydration?
>
> The rehydration feature means that the Glimmer VM can take a DOM tree
created using Server Side Rendering (SSR) and use it as the starting
point for the append pass.
>
> Rehydration is optimized for server-rendered DOMs that will not need
many changes in the client, but it is also capable of making targeted
repairs to the DOM:
>
> - if a single text node's value changes between the server and the
>  client, the text node itself is repaired.
> - if the contents of a `{{{}}}` (or SafeString) change, only the
>   bounds of that fragment of HTML are blown away and recreated.
> - if a mismatch is found, only the remainder of the current element
  is cleared.
>
> What this means in practice is that rehydration repairs problems
in a relatively targeted way, and doesn't blows away more content
than the contents of the current element when a mismatch occurs.
>
>Near-term work will narrow the scope of repairs further, so that
mismatches inside of blocks only clear out the remainder of the
content of the block.

<small>(from: https://github.com/glimmerjs/glimmer-vm/commit/316805b9175e01698120b9566ec51c88d075026a)<small>

Currently Fastboot rehydration is available behind a (rather lengthy) environment variable: `EXPERIMENTAL_RENDER_MODE_SERIALIZE=true`.  The Goal of this RFC is to enable this by default and explore the work necessary to accomplish that.

## Motivation

Much of the work done in FastBoot today is duplicated because rehydration is not enabled.  In fact, the current implementation of `ember-cli-fastboot` has an initializer ([here](https://github.com/ember-fastboot/ember-cli-fastboot/blob/master/addon/instance-initializers/clear-double-boot.js)) called `clear-double-boot` which removes all of the work done in SSR land so the Glimmer VM has a fresh start to begin creating the DOM. With Rehydration the work done on the server is not thrown away and instead re-used, so removing the `clear-double-boot` should result in faster websites.

Rehydration already works with a few notable gotchas that'll we cover below.  You can, in fact, see it in action at  outdoorsy.com.  To see some of the tell-tale signs that rehydration is happening can be seen if you go there and view the request that was returned over the network.  You'll see the FastBoot content along with many comment markers (they'll look like `<!--%+b:9%-->`) used to inform the rehydration element build how to rebuild the DOM without needing to clobber it.

## Detailed design

Most of this work can be used today, the work nessecary for implmenting this feature would be to remove the need for the environment variable.  However, there are some unresolved problems that still need answers before we can do that.

- [ ] Create a strategy for dealing with special elements ie `style`, `title`, and `pre`
    - These tags do not render their contents as DOM instead they print exactly what you see without interpreting.  Since (as mentioned in the motivation section) there are comment tags used by the rehydration builder they show up.  This is particularly noticeable in the `title` element where the browser will show those comments in the open tab.  There is currently a workaround, but we need a more complete solution before we ship.
    - https://github.com/glimmerjs/glimmer-vm/issues/796
- [ ] Make the way Ember is booted more stream lined.
    - In FastBoot Ember boots itself and produces the output via the `visit` API.  However, in the Browser we need to override the default element builder by monkey-patching `Ember.ApplicationInstance#_bootSync`.  If we always enable rehydration this could likely be hardcoded, but planning is needed to consider all options.  In an ideal world we might consider using `visit` in both places
- [ ] Allow user's to opt-out of rehydration
- [ ] Instrument rehydration such that its impact can be measured so user's can make informed decisions about its merit in their application.
    - See here for relevant issue: https://github.com/ember-fastboot/fastboot/issues/107

## How we teach this

I believe most of this should be transparent to the user.  However, the documentation would need to be updated to alert users to potential footguns.  And how to opt-out.

## Drawbacks


## Unresolved questions

> Optional, but suggested for first drafts. What parts of the design are still
TBD?

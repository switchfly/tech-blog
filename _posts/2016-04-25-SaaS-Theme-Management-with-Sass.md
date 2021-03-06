---
layout: post
author: Scott Galloway
author-image: scottgalloway.jpg
title: SaaS Theme Management with Sass
summary: An introduction into how we are handling client branding in our newest Shopping platform

---

# SaaS Theme Management with Sass

I serve as the UI Architect for our newest product suite, building upon modern technology/frameworks: EmberJS, Spring MVC, Bootstrap, Sass, Handlebars, etc. In this article, I want to share an overview of how I’ve structured our themes using Bootstrap and Sass.

## Overview

We leverage a single SaaS application to service any number of client branded or co-branded sites, supported on all common modern devices (desktop/laptop/tablet/smartphones). This presents a challenge in that this single app must be flexible enough to handle the flexibility required for each brand’s needs, which is supported by massive configurability and localization behind the scenes. For branding, I wanted to be very careful to balance theme customization with the repeatability / structural integrity expected of a SaaS platform, especially when it comes to [Responsive Web Design](https://en.wikipedia.org/wiki/Responsive_web_design).

## High Level Approach

As best I can, I try to take an Object Oriented CSS ([OOCSS](https://github.com/stubbornella/oocss/wiki)) approach, placing an emphasis on the separation of structure and skin.

For our purposes, a “theme” is a list of .scss variable definitions (.scss is a syntax variant of sass that I prefer due to it following traditional .css syntax more closely than .sass does; this is particularly helpful in a large team) that our application stylesheets consume later in the process. Here’s what our very high level theme flow looks like:

![Setup, Theme definition, Apply variables](https://cdn-images-1.medium.com/max/1600/1*5264DlV9S-QRZIFKpDWptQ.png)

1. **Setup:** We make mixins available to our themes (we use a combination of Bootstrap, Compass, and local mixins).
2. **Theme definition:** We extended Bootstrap’s theme with our own variables for things unique to our application. We then allow a brand theme to override our variable definitions.
3. **Apply our variables:** We take all the theme variables defined previously, and then apply them in our application stylesheets.

## Theme Definition

### Step 1 — Setup

I’m going to skip over the “Setup” as it’s pretty straight forward, so we can focus on steps 2 and 3.

### Step 2.1 — Application theme

This is where we define what our application styles consume. I think of theme files as my theme palette, and similar to a palette, should ONLY contain variables. We start by setting variables to define our default skin and general layout:

* Colors (grays, brand colors, state colors (warning, success, danger), etc)
* Typography (font family and style, heading sizes, links, line-heights, labels, etc)
* Common component style (Tables, buttons (and variants), forms, alerts (and variants), modals, tooltips, border-radii, pagination, section style, badges, etc)
* Custom application component style (accordions, date pickers, map markers, filter sliders, flight seat maps, pricing related styles, etc)
* Structure / layout (grid definition, [media query](http://www.w3schools.com/cssref/css3_pr_mediaquery.asp) breakpoints, base padding)

In CSS preprocessors like Sass, you can set a variable’s value to itself reference a variable. For example, you might define the primary brand color as blue, and inherit that definition in other variables:

```scss
$brand-primary: blue;
$btn-primary-bg-color: $brand-primary;
$link-color: $brand-primary;
$clickable-element-bg-color: $link-color;
```

Because of how I define variables that then are consumed later in increasingly specific components, I keep this file ordered from generic variables (colors and common style) to increasingly explicit (common components, with specific application components last).

### Step 2.2 — Brand theme

Now that we have our application defaults described, we can override any that we wish for a given brand. For example, above we defined our primary brand color to blue, but let’s say the primary brand color for this client is red. In their brand theme, they’d simply override our value with theirs:

```scss
$brand-primary: red;
```

And that’s it. Of course, in our Step 2.1 code example earlier, we also defined `$link-color` to `$brand-primary`, meaning by simply setting `$brand-primary` to red, links, btn-primary, and “clickable element” (this is a made up example component) are also now red. Our application theme made an assumption that link color (and btn-primary, etc) would commonly be the same color as your primary brand, but that may not be the case (I’ve found that assumption to be reasonable; ~10% of our themes override this with a specific link text color). Setting the link color for the brand is simple enough:

```scss
$brand-primary: red;
$link-color: darkred;
```

Because I want to maintain all the layout and structural behaviors, including what CSS selectors we use and when, **it’s vital that these theme files keep to setting variables ONLY as much as possible.** As soon as we begin writing CSS in these files, we begin to introduce outliers that make it difficult to expect consistent behaviors when rendering our pages.

**Important general theme rule: If a theme variable does not exist, it is not brand-able.** If a valid case exists for something to be brand-able that isn’t already, we have to take that in as a change request at the application level in order to provide appropriate support for that new feature.

#### Step 2.2.1 — Sub-brand Themes

A given client may have multiple sub-brands; for example, a credit card company or loyalty membership program may have several tiers (silver, gold, etc). But, these sub-brands rarely differ beyond setting a few colors; that is, they inherit their brand’s base theme. We do this by basically defining a `_base.scss` partial that defines all of the shared layout and default variable definitions, which we include in our brand theme files, and then set overriding variables specific to our sub-brand as needed in each sub-brand’s theme.

### Step 3 — Apply our variables

Now that we’ve set all the variables that define our theme, our application stylesheets take over. Here is where we CONSUME all our variables using CSS to select the elements to which each variable applies and assigning them as values to the appropriate property; for example:


```scss
//Links

a {
  color: $link-color;
  text-decoration: underline;

  &:hover,
  &:focus {
    color: $link-hover-color;
    text-decoration: none;
  }
}
```

`&` in Sass is a reference to it’s parent selector; above we have nested `&:hover` inside the braces for the `a` element selector, which when compiled, the .css will be outputted like this:

```css
a {
  color: darkred;
  text-decoration: underline;
}
a:hover,
a:focus {
  color: darkred;
  text-decoration: none;
}
```

Similar to how I (try to) organize themes by defining variables for increasingly specific elements, I define sass files for specific pages (if needed), for specific components (which I namespace to ensure selection of just that component), and then include these files in increasingly specific order:

1. Bootstrap styles
2. Application styles (base components + scaffolding, helper classes, animation, icon fonts, etc)
3. Page styles
4. Component styles
5. Responsive styles

## Costs and Benefits

### Costs

The first cost to this approach is that, at least in our case, there is a lot of style code complexity — there is a learning curve to understanding how things are connected, where things are defined and consumed, etc. As such, subject matter expertise is difficult to achieve; though as I’ll discuss in a moment, due to virtual elimination of UI related bugs, we need very little resources to maintain our application style code.

In previous products, we’ve allowed client theme .css to override base stylesheets. The costs to this approach are quite high;

* You must train client site administrators on site context: how will they know the proper selector chain for every link or heading, etc.
* It’s heavily reliant on the client admin’s understanding of CSS itself: syntax, specificity rules, risks/benefits of one solution over another, browser and device compatibility, etc.
* Bug counts are relatively high and theme quality might be impacted: as a result of the above risks, it’s easy to see how a client may have unknowingly added code that negatively effected pages/cases they weren’t aware of at the time.
* Custom client code makes it difficult to innovate the core product itself when you can’t expect specific outcomes for your new feature (e.g. your carefully described component is broken for a given client because they happen to use a selector that matches and absolutely positions something off canvas).
* All of this adds a fair amount of resource overhead both externally and internally, and yields a user experience that too is at risk, and frankly isn’t as Saas-like as I wanted our products to be.

> What the client asks for, is not necessarily what they want. - Scott Galloway

We exist to serve our clients, who themselves exist in some fashion to serve theirs. When requests for change come in, they are coming from a good place and are meant to benefit their users in some (hopefully measurable) way. But the expectation is, and should be, that a feature request should fit nicely within the context of the greater application and not negatively impact the experience of the end user.

If you’re reading this article, you understand that there is a massive amount of complexity related to providing your users a lovable web product, and it takes careful planning to maintain your product’s integrity. We can’t just add new feature code without really looking at it in the context of the application as a whole, and how it fits: Are there existing features that we should extend instead? If this is truly new, what existing UI patterns exist in our app that we will leverage here to ensure proper layout for all of our supported use cases / browsers / devices / etc?

### Benefits

> …a branding exercise might take 60 to hundreds of development hours; using this new approach, branding can take just 2.

Paired with defining reusable patterns, the above approach has already paid huge dividends in both setup and maintenance. Using our client stylesheet override approach, a branding exercise might take 60 to hundreds of development hours; using this new approach, branding might take just 2 hours. This is made possible by limiting a brand/theme to variable definition, and letting our application stylesheets perform all the heavy lifting of applying those values.

There are plenty of other great benefits with this approach too:

* Our brand related **bug counts have almost disappeared completely**.
* If a bug does exist, it’s almost certainly an application level bug that is handled by developers with subject matter expertise in our application stylesheets.
* Our development resources can be spent much more efficiently on product innovation and not branding or bug churn.
* Automation testing is simplified and more predictable.
* Our change request process is simplified in that we can now reference existing UI patterns for new features instead of needing to design (and prototype and test and iterate and develop) new ones.

## Closing

Of course there are lots of nuanced discussions we can have on all of this; one of which is the sheer size of our compiled stylesheets. Older IE browsers have an upper limit of 4096 style rules before the browser just stops parsing; I’m currently using [Bless](http://blesscss.com/) to split our compiled theme stylesheets into parseable chunks (in the future, I hope to better optimize assets such that I won’t need to do this any more).

Another thing we’ve also started doing is splitting style asset packaging into a [handheld](https://medium.com/@scottgalloway/do-you-really-mean-mobile-b103dfce2a36#.oh8i7qnh8) case and a base (not handheld; “tablet” and larger) case. This gives me much tighter control of the UI for handheld (aka “mobile”) users, but also in the style assets themselves, allowing me to limit the assets to what’s needed, or even change frameworks completely (e.g. maybe Bootstrap is too large for our handheld needs).

As a UI guy architecting a SaaS application that serves any number of brands, I’ve found very few articles online to help me solve some of the unique problems related specifically to what I’m trying to achieve. However, I can now say from experience that this idea has been a big success already, and I look forward to streamlining it even further.
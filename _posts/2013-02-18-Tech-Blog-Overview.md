---
layout: post
author: Scott Galloway
author-image: scottgalloway.jpg
title: Tech Blog Technical Overview
summary: A technical overview of this blog including libraries used and why particular decisions were made.

---

With the new Switchfly Technical Blog, I wanted to provide a quick overview of some of the technology decisions we made to support our goals, and why we made some of those choices.

## High Level Technical Goals

* Must be easy to host and update
* Must use RWD (Responsive web design) - i.e. the site scales appropriately to various device sizes
* Must allow technical and design flexibility while staying lightweight

## Hosting / Updating

I considered several options, including blog authoring tools like Blogger, Drupal, Wordpress, etc, but in the end those all felt a little cumbersome for what we needed. Additionally, creating a technical blog seemed to require a bit less structure than what those tools provided. In the end, I chose GitHub for a few notable reasons:

1. Switchfly uses Git for our version control
2. With [GitHub Pages](http://pages.github.com/), I could host the site easily
3. GitHub Pages utilizes [Jekyll](https://github.com/mojombo/jekyll/wiki) for site generation:
    * Templates for layouts of pages and partials
    * Utilizes -Textile- Markdown for post rendering
4. GitHub Pages utlizes [Liquid](http://liquidmarkup.org/) for logic and variables

### Why Textile and not Markdown?

A personal preference. Similar to my below discussion on .scss vs .sass, I find that the textile syntax is clearer; for example, heading levels in textile are noted as (h1. h2. h3., ...), and in markdown they are (#, ##, ###, ...); when you get a few heading levels down, it's pretty tough in markdown to figure out if you're at an H4 (####) level or H5 (#####) level without resorting to actually counting pound signs. Here is a nice [comparison](http://mojomojo.org/documentation/textile_vs_markdown). 

**Update:** Github Pages have deprecated support for Textile as of May 1st, 2016; so, our pages will be using Markdown now instead. Honestly, the syntax distinction is pretty trivial to overcome. :-)

### Sass Support

Another great out of the box feature of GitHub Pages is their support of [SASS](http://sass-lang.com/). I like to think of SASS as what will be (very likely in my view) supported in future CSS specs, but usable today through clever compilation. I generally prefer .scss syntax over .sass syntax, as I believe it reads much more like real CSS code than does .sass, even though .sass is a bit more terse. For use in a big team like ours, I think adhering closely to commonly understood syntax is best.

### Syntax Highlighting

We are using Pygments for our code highlighting here in our blog; I find it easy, and plenty nice.

## Responsive Web

Again, there are many framework options here to choose from including Bootstrap, Foundation, etc. In the end, I chose to go with [Skeleton](http://www.getskeleton.com/); here's why...

When I was looking at my options, I found that while frameworks like Bootstrap and Foundation did everything I needed, they were both more robust than what I required; notably, they included a UI library that I felt just wasn't necessary for a small blog site. Skeleton on the other hand is pretty much *only* a responsive CSS framework. Additionally, Skeleton doesn't really have any style applied to it - it does it's best in the area of browser resets for common elements such as typography, forms, etc, but the bulk of the benefit is a nice simple responsive grid.

Plus, weighing it at a small 28kb, it's plenty lightweight.

## Miscellaneous stuff

In addition to all of the above, I have baked in some other things into the site to allow for some fancier UI down the road:

* [JQuery](http://jquery.com) - for finding stuff on the page and doing fancy stuff with it
* [Modernizr](http://modernizr.com) - for feature detection
* [AddThis](http://www.addthis.com) - for article sharing and analytics

## Wrapping Up

In the end, I think we created a flexible, lightweight, and relatively simple blog for us to play with and share our learning with the community. Thanks for visiting!
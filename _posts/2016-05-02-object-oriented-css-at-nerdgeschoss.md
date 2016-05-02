---
tags: [css, sass]
title: Object Oriented CSS with SASS
abstract: Over time almost all css projects turn into a mess of selectors and strange sizing. Here is my approach of dealing with it.
---

# _Object Oriented CSS_ with SASS

Over time almost all css projects turn into a mess of selectors and strange sizing. Here is my approach of dealing with it. This is mostly adapted from on a post by Medium that you can find [here](https://medium.com/@fat/mediums-css-is-actually-pretty-fucking-good-b8e2a6c78b06).

## Writing _Components_

Almost everything on a page is a component. A component can either be sized by it's contents (like a `span`) or taking the full width that its parent is allowing (e.g. `div`). Components _never_ define their own width.

A component is named in `camelCase`, descendents of the component are prefixed by their component name. All selectors only use classes, never ids or tag names.

```css
.twitterCard {
  display: block;
}

.twitterCard-headline {
  font-size: 1.5rem;
}
```

with the corresponding HTML

```html
<div class="twitterCard">
  <div class="twitterCard-headline">
</div>
```

This can be shortened a lot by scss:

```scss
.twitterCard {
  display: block;

  &-headline {
    font-size: 1.5rem;
  }
}
```

Each component goes into its own file that is named like the component itself.

This namespacing tries to emulate the [shadow dom of web components](https://developer.mozilla.org/en-US/docs/Web/Web_Components/Shadow_DOM) as much as possible.

Most components are already responsive enough this way to adapt to mobile and desktop. All breakpoints are always defined for the best display of a single component instead of global breakpoints for specific devices (e.g. a `twitterCard` component might change some properties at 600px width, a `button` might change it's paddings at 200px).

## _Composition_ of Components

Components can contain other components, e.g. a `box` component containing a `mainText` component. These components should go in seperate files, even if they're always used together.

Components should never contain references to their subcomponents (e.g. a `box` that is styling a contained `button` component). Use inheritance instead.

## Bring in some _Inheritance_

Sometimes you need some variants of already created components. This can be done via the inheritance selector (`--`):

```scss

.button {
  color: blue;
}

.button--awesomeUpgrade {
  color: rebeccapurple;
}
```

In HTML always include the main class _and_ the modifier:

```html
<span class="button button--awesomeUpgrade">
```

## _Layouting_ Components on the Page

So far all components either have their childrens with or take the whole page's width. To add some layout, we use layout classes, defined with the `l-` prefix:

```scss
.l-grid {
  display: flex;
  flex-wrap: wrap;

  > * {
    flex-grow: 1;
    width: calc(25% - 30px)
  }
}
```

Then you can write your html like this

```html
<div class="l-grid">
  <div class="twitterCard"></div>
  <div class="twitterCard"></div>
  <div class="twitterCard"></div>
  <div class="twitterCard"></div>

  <div class="twitterCard"></div>
  <div class="twitterCard"></div>
  <div class="twitterCard"></div>
  <div class="twitterCard"></div>
</div>
```

to get a 2 row 4 column grid of `twitterCard` components.

This goes really well with the [slim templating language](http://slim-lang.com):

```slim
.l-grid
  .twitterCard
  .twitterCard
  .twitterCard
  .twitterCard

  .twitterCard
  .twitterCard
  .twitterCard
  .twitterCard
```

## General _Settings_

To add some basic settings for the page, each of our projects has a `variables.scss` containing all colors and sizes for the project:

```scss
// Colors
$color-text: #000;
$color-text-muted: #9195a3;
$color-background: #efefef;
$color-tint: #62c5e8;
$color-warning: #f5a623;

// Fonts
$font-headline: 'Sofia Pro', 'Avenir', Helvetica, Geneva, sans-serif;
$font-body: 'Minion Pro', Georgia, Times, serif;
$font-monospace: 'Menlo', Monaco, monospace;

// Breakpoints
$tablet: 740px;
$desktop: 1000px;
$widescreen: 1180px;

// Font Sizes

$font-size-headline: 1.5rem;
$font-size-body: 1rem;
$font-size-base: 17px;
```

Also there is a `typography.scss` that contains basic text styling (this is actually the only file in the project that references tag names):

```scss
html {
  font-size: $font-size-base;
  color: $color-text;
  font-family: $font-body;
  line-height: 1.3;
}

p {
  margin: 1.6em 0;
}
```

## _Recap_

Splitting up css based on components made big projects a lot more readable, espacially when working togher with bigger teams and longer timespans. Depending on a class name you can immediately see from which file it is and to which component it belongs.

Making components width-agnostic allows for fast refactorings and fast prototyping ("just add a grid of cards here"). Also it allows to have a living styleguide inside of your application just listing all the possible components (all of our projects have a `/styleguide` page to easily see all defined components).

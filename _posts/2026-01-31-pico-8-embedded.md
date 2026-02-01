---
title: "Embedding Pico-8 Games in Jekyll: Creating a Reusable Layout"
excerpt: Here's how I transformed Pico-8's exported HTML into a reusable Jekyll layout
layout: post
---

While building a portfolio for this page, the first thing I wanted to show was a very small pico-8 game I made for my two lovely nephews, showcasing my guinea pig Chantal, in an adventure called: "Chantal Panic".

<div style="text-align: center; margin: 30px 0;">
  <img src="{{ '/assets/images/chantal.jpg' | relative_url }}" alt="Chantal the guinea pig" style="border-radius: 8px; width: 180px;">
  <p style="margin-top: 10px; font-style: italic; color: #666;">isn't she cute?</p>
</div>


I needed a clean way to embed them without duplicating code for each game. Here's how I transformed Pico-8's exported HTML into a reusable Jekyll layout.

## What is Pico-8?

[Pico-8](https://www.lexaloffle.com/pico-8.php) is a fantasy console - a virtual game console with intentional limitations. Think of it like a Game Boy, but for modern developers. It comes with built-in tools for creating games:

-   **Code editor** - Write games in Lua
-   **Sprite editor** - Draw your graphics (128x128 pixels)
-   **Map editor** - Design your levels
-   **Sound editor** - Create music and sound effects
-   **Export to web** - Share your games on the web, whether it's on the official BBS, on itch.io or on any HTML page you like

The charm of Pico-8 is its constraints: 128x128 screen, 16 colors, 32KB cart size. These limitations force creativity and make it possible to actually finish games.

When you export a Pico-8 game to HTML, you get a complete web player that runs your game in any browser. That's what we're going to embed in Jekyll.

## The Problem

Pico-8's HTML export gives you everything in one file - a complete standalone web page. But when you're building a Jekyll site, you want:

-   Your site's consistent layout and navigation
-   Easy management of multiple games
-   No code duplication

## A Pico-8 Layout

### Step 1: Export from Pico-8

In Pico-8, export your cart to HTML:

{% highlight lua %}
  export mygame.html
{% endhighlight %}

This creates two files:

-   `mygame.html` - The complete page with player : it is always re-uses the same logic
-   `mygame.js`   - Your actual cart data

### Step 2: The Pico-8 Layout

Just create  `_layouts/pico-8.html`  and drop the complete Pico-8 player shell. The key insight was to  **use the entire exported HTML as the layout**, with one crucial modification:

Instead of hardcoding the cart filename, I made it dynamic using Jekyll's frontmatter:

{% highlight html %}
  <script type="text/javascript" src="{{ page.filepath }}.js"></script>
{% endhighlight %}

This single line makes the layout reusable. The exported HTML from Pico-8 normally has:

{% highlight html %}
  <script type="text/javascript" src="mygame.js"></script>
{% endhighlight %}

By replacing the filename with  `{{ page.filepath }}.js`, I can now specify which cart to load via frontmatter.

### Step 3: Creating a Game Page

Now I can create game pages in  `_projects/`  with minimal code:

{% highlight markdown %}
---
  short_name: chantalpanic
  name: Chantal Panic!
  filepath: chantalpanic
  description: This is a game made for my nephews, in honor of my guinea pig named Chantal.
  subdescription: In this game, the user needs to  avoid the cages, the two evil twins and collect carrots.
  label: pico-8
  layout: pico-8
---
{% endhighlight %}

That's it! The  `filepath`  parameter tells the layout which  `.js`  file to load.

### Step 4: Organizing Assets

For now I keep my cart  `.js` inside of the projects folder, but I could actually use a dedicated folder, as long as I
remember to update the `filepath` parameter:

{% highlight markdown %}
  └── _projects/
      ├── chantal-panic.md
      └── another-game.md
      ├── chantal-panic.js
      ├── another-game.js
{% endhighlight %}

## Why This Works

**The exported Pico-8 HTML is already perfect**  - it includes:

-   Canvas setup
-   Touch controls for mobile
-   Fullscreen support
-   Gamepad handling
-   All the Pico-8 player features

I didn't need to extract pieces or rewrite anything. I just:

1.  Took the entire exported HTML
2.  Wrapped it in my Jekyll layout structure
3.  Made the cart filename dynamic

## The Project Layout

The `_layouts/pico-8.html` does not inherit from the default layout. I found out that trying to wrap the Pico-8 player in a layout that inherits from the default layout break the embedded layout; the HTML provided by Pico-8 is good enough for my needs.

## Next Steps

With this layout system in place, I could now focus on something my nephews have been wanting for a while, but is a bit complex to put into place with a pico-8 game: a "best score" board that is not browser-contained.

To be continued in the future posts:
-   Communication between JavaScript and the embedded Pico-8 player
-   Persisting high scores across page loads for a static webpage

## Code

You can see the full implementation in my repository:
-   [_layouts/pico-8.html](https://github.com/ABraconnier/antoinebraconnier.github.io/blob/master/)

----------

_Part 1 of the "Pico-8 and Jekyll" series_

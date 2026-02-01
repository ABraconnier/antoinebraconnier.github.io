---
title: "How to interact with your embedded Pico-8 games in Jekyll"
description: Here's what I did to inject some external data into my Pico-8 games with javascript
layout: post
---

I coded a small Pico-8 game for my two lovely nephews, showcasing my guinea pig Chantal, in an adventure called "Chantal Panic". The first part of this series, where I explain how to embed the game into HTML is [here](https://www.antoinebraconnier.com/posts/2026-01-31-pico-8-embedded/).

My nephews asked for a high score board, so I had to figure out how to:

1. Inject some external data from my jekyll website into the embedded Pico-8 game
2. Make the game modify this data and persist it in a website that is normally static

This article will focus on the first question.

## The Problem

Pico-8's HTML export is a very long generated JS file: all the sprites, lua code is included in that file. It is not possible to inject data into that generated file.

## The Solution

I had two possibilities to inject external data into the Pico-8 game:

- **Use a the Cartridge data feature** - You can [save and re-load data](https://www.lexaloffle.com/dl/docs/pico-8_manual.html#CARTDATA) from a pico-8 cartridge. If the game has been exported to HTML, the data is stored in the browser's local storage, through the [indexedDB API](https://developer.mozilla.org/en-US/docs/Web/API/IndexedDB_API).

**The problem:** indexedDB is a very low-level API that's complex to use with JavaScript.

- **Use the GPIO feature** - The pico-8 API has some [functions](https://www.lexaloffle.com/dl/docs/pico-8_manual.html#GPIO) to interact with GPIO pins. GPIO stands for "General Purpose Input Output", they are hardware pins that you can use to communicate with the game.

**And luckily**, pico-8 simulates these GPIO pins in HTML exports using a simple JavaScript array!

Here is what the manual says:

```
Cartridges exported as HTML / .js use a global array of integers (pico8_gpio) to represent gpio pins. The shell HTML should define the array:

var pico8_gpio = Array(128);
```

So if we program our game to listen to these virtuals hardware pins, we can use this simple variable to store and / or modify this data that our game listens to! It's our best bet to achieve our goal.

## How It Works

Here's the data flow:

```
Jekyll (_data/best_scores.yml)
    ↓
JavaScript (pico8_gpio array)
    ↓
Pico-8 Game (peek from GPIO pins)
```

### Step 1: Make your game listen to GPIO pins

In your Pico-8 game, read from the GPIO memory region:

{% highlight lua %}
function fetchscores()
  -- GPIO pins are mapped to memory addresses 0x5f80 - 0x5fff
  -- We're using the first 4 bytes for our high score data

  local namea = chr(peek(0x5f80))  -- First letter
  local nameb = chr(peek(0x5f81))  -- Second letter
  local namec = chr(peek(0x5f82))  -- Third letter
  local amount = peek(0x5f83)      -- Score amount

  local name = namea..nameb..namec
  local score = {}
  score.name = name
  score.amount = amount
  return score
end
{% endhighlight %}

**What's happening:**
- `peek(0x5f80)` reads from GPIO pin 0
- `chr()` converts the byte value to a character
- We concatenate the three letters to form a name


### Step 2: Inject the data into the Pico-8 game thanks to Javascript

Now that my game is "plugged" to the GPIO, once exported to HTML, I can inject the data I want into the game thanks to Javascript. Isn't it amazing? I think so too!

I modified `_layouts/pico-8.html` to allow individual game pages to inject their GPIO setup.

Find the closing `</script>` tag (after all the Pico-8 player code) and add `{{content}}` right before the `<STYLE>` section:

{% raw %}
{% highlight html %}
  </script>
    {{content}}  <!-- Game-specific JavaScript goes here -->
  <STYLE TYPE="text/css">
{% endhighlight %}
{% endraw %}

This placement is crucial - it ensures `pico8_gpio` is defined **before** the game starts running.


### Step 3: Inject Data from Jekyll

Now in my game page (`_projects/chantal-panic.md`), I add JavaScript that reads from Jekyll's data and writes to the GPIO array:

{% highlight html %}
<script>
  // Create the magic array that Pico-8 will read from
  var pico8_gpio = new Array(128);

  // Fetch score from Jekyll data (with defaults if file doesn't exist yet)
  let bestScoreName = "{{ site.data.best_scores.chantalpanic.name | default: 'AAA' }}";
  let bestScoreAmount = parseInt("{{ site.data.best_scores.chantalpanic.score | default: 0 }}");

  // Convert the name into bytes (ASCII character codes)
  pico8_gpio[0] = bestScoreName.charCodeAt(0);  // First letter -> GPIO pin 0
  pico8_gpio[1] = bestScoreName.charCodeAt(1);  // Second letter -> GPIO pin 1
  pico8_gpio[2] = bestScoreName.charCodeAt(2);  // Third letter -> GPIO pin 2
  pico8_gpio[3] = bestScoreAmount;              // Score -> GPIO pin 3
</script>
{% endhighlight %}

**How it works:**
1. Jekyll processes the template and injects `{{ site.data.best_scores.chantalpanic.name }}`
2. JavaScript converts the name "BOB" to bytes: `[66, 79, 66]`
3. Pico-8 reads these bytes with `peek()` and converts them back to text

## Next Steps

Now I can **read** data from Jekyll into Pico-8. But what about the reverse? How do I get high scores **out** of the game and persist them?

That's where it gets interesting - and requires a GitHub Actions workflow. Coming in Article 3!

----------

_Part 2 of the "Pico-8 and Jekyll" series_

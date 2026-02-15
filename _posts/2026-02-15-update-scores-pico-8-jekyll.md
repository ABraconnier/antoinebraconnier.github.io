---
title: "Persisting Pico-8 high scores on a static Jekyll site"
description: How I built a complete high score system using Cloudflare Workers and GitHub Actions
layout: post
---

In [Part 1](https://www.antoinebraconnier.com/posts/2026-01-31-pico-8-embedded/), I showed how to embed Pico-8 games in Jekyll. In [Part 2](https://www.antoinebraconnier.com/posts/2026-02-01-pico-8-javascript/), I demonstrated reading high scores from Jekyll into Pico-8 using GPIO.

Now comes the hard part: **How do you save high scores from a game running on a static website?**

## The Challenge

Static sites like GitHub Pages don't have a backend. They're just HTML, CSS, and JavaScript served directly to users. When a player beats the high score:

1. The game needs to send the new score somewhere
2. That somewhere needs to update my Jekyll site's `_data/best_scores.yml` file
3. The site needs to rebuild with the new data

**But here's the problem:** You can't just commit to GitHub from JavaScript running in someone's browser. You need authentication tokens, and exposing those tokens would let anyone modify your repository.

## The Solution Architecture

Here's the complete flow I built:

```
Player beats high score in Pico-8
    ↓ (poke to GPIO)
JavaScript detects GPIO change
    ↓ (fetch request)
Cloudflare Worker (proxy with secrets)
    ↓ (authenticated API call)
GitHub Actions triggered
    ↓ (creates/updates PR)
Pull Request with new high score
    ↓ (I review and merge)
Jekyll rebuilds with new score
```

Of course, the score change is not instantaneous, and at the end of the line there's still me approving the PR. I could have just a github action commiting directly to my master branch, but I thought it would be safer that way. And it's good enough for my nephews!

Let me break down each step.

## Part 1: Sending Score from Pico-8

When a player beats the high score, my game writes to GPIO pins:

{% highlight lua %}
function submit_new_score(name, points)
  -- Convert 3-letter name to bytes and write to GPIO
  poke(0x5f80, "a")  -- First letter
  poke(0x5f81, "b")  -- Second letter
  poke(0x5f82, "c")  -- Third letter
  poke(0x5f83, 10)   -- Score amount
end
{% endhighlight %}

This is the **reverse** of Part 2 - instead of reading GPIO pins with `peek()`, I'm writing with `poke()`.

## Part 2: Detecting Changes in JavaScript

In my game page, I poll the GPIO array for changes:

{% highlight html %}
{% raw %}
<script>
  var pico8_gpio = new Array(128);

  // Initialize with current best score from Jekyll
  let bestScoreName = "{{ site.data.best_scores.chantalpanic.name }}";
  let bestScoreAmount = parseInt("{{ site.data.best_scores.chantalpanic.score }}");

  // Set initial GPIO values
  pico8_gpio[0] = bestScoreName.charCodeAt(0);
  pico8_gpio[1] = bestScoreName.charCodeAt(1);
  pico8_gpio[2] = bestScoreName.charCodeAt(2);
  pico8_gpio[3] = bestScoreAmount;

  // Check for changes every second
  setInterval(() => {
    if (pico8_gpio[3] != bestScoreAmount) {
      // Score changed! Extract new values
      bestScoreName = String.fromCharCode(
        pico8_gpio[0],
        pico8_gpio[1],
        pico8_gpio[2]
      );
      bestScoreAmount = pico8_gpio[3];

      // Submit to our backend
      submitScore(bestScoreAmount, bestScoreName);
    }
  }, 1000);

  async function submitScore(score, playerName) {
    try {
      // Here, you'd need to replace the URL with your own
      const response = await fetch('https://yourserverlessfunction.workers.dev', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ score, player: playerName })
      });

      const result = await response.json();

      if (result.success) {
        console.log('Score submitted successfully!');
        alert('Bravo! 🎉');
      } else {
        console.error('Error:', result.error);
      }
    } catch (error) {
      console.error('Failed to submit score:', error);
    }
  }
</script>
{% endraw %}
{% endhighlight %}

**Why polling?** Pico-8 doesn't have callbacks or events. The simplest solution is to check the GPIO array periodically.

## Part 3: The Cloudflare Worker Proxy

**Why do I need a proxy?** Because:

1. **Security**: GitHub API requires authentication tokens
2. **Can't expose tokens**: Putting tokens in client-side JavaScript means anyone can use them
3. **Rate limiting**: Prevent spam submissions
4. **Validation**: Check scores are legitimate

The proxy sits between your game and GitHub, holding secrets safely.

### Setting Up Cloudflare Workers

**Step 1: Install Wrangler**

```bash
npm install -g wrangler
wrangler login
```

**Step 2: Create your worker project**

```bash
mkdir score-updater
cd score-updater
wrangler init
```

Choose a worker only template (and yes to git).

**Step 3: Create the worker code**

Create `src/index.js`:

{% highlight javascript %}
export default {
  async fetch(request, env) {
    // Handle CORS preflight
    // When JavaScript from my Jekyll site (different domain) calls this worker,
    // the browser first sends an OPTIONS request asking "is this allowed?"
    // We respond with headers saying "yes, your domain can make POST requests here"
    if (request.method === 'OPTIONS') {
      return new Response(null, {
        headers: {
          'Access-Control-Allow-Origin': env.ALLOWED_ORIGIN,
          'Access-Control-Allow-Methods': 'POST, OPTIONS',
          'Access-Control-Allow-Headers': 'Content-Type',
        }
      });
    }

    if (request.method !== 'POST') {
      return new Response('Method not allowed', { status: 405 });
    }

    try {
      const { score, player } = await request.json();

      // Validate: score is a number between 0-999999
      if (typeof score !== 'number' || score < 0 || score > 999999) {
        return jsonResponse({ error: 'Invalid score' }, 400, env);
      }

      // Validate: player name is exactly 3 letters A-Z
      if (!/^[a-zA-Z]{3}$/.test(player)) {
        return jsonResponse({
          error: 'Player name must be 3 letters A-Z'
        }, 400, env);
      }

      // Rate limiting: max 1 submission per 30 seconds per IP. We'll see the SCORE_KV env var in the next step
      const clientIP = request.headers.get('CF-Connecting-IP');
      const rateLimitKey = `ratelimit:${clientIP}`;

      if (env.SCORE_KV) {
        const lastUpdate = await env.SCORE_KV.get(rateLimitKey);
        if (lastUpdate) {
          const timeSince = Date.now() - parseInt(lastUpdate);
          if (timeSince < 30000) {
            return jsonResponse({
              error: 'Please wait before submitting again',
              retryAfter: Math.ceil((30000 - timeSince) / 1000)
            }, 429, env);
          }
        }
      }

      // Trigger GitHub workflow via API
      const response = await fetch(
        `https://api.github.com/repos/${env.GITHUB_REPO}/dispatches`,
        {
          method: 'POST',
          headers: {
            'Accept': 'application/vnd.github.v3+json',
            'Authorization': `Bearer ${env.GITHUB_TOKEN}`,
            'Content-Type': 'application/json',
            'User-Agent': 'Pico8-Score-Updater'
          },
          body: JSON.stringify({
            event_type: 'update-score',
            client_payload: { score, player, timestamp: Date.now() }
          })
        }
      );

      // Update rate limit
      if (env.SCORE_KV && response.status === 204) {
        await env.SCORE_KV.put(rateLimitKey, Date.now().toString(), {
          expirationTtl: 60
        });
      }

      if (response.status === 204) {
        return jsonResponse({
          success: true,
          message: 'Score submitted!'
        }, 200, env);
      } else {
        console.error('GitHub API error:', response.status);
        return jsonResponse({ error: 'Failed to submit' }, 500, env);
      }

    } catch (error) {
      console.error('Worker error:', error);
      return jsonResponse({ error: 'Server error' }, 500, env);
    }
  }
};

// Reponse object is from the fetch API, the same as the one in a browser https://developer.mozilla.org/en-US/docs/Web/API/Response
function jsonResponse(data, status, env) {
  return new Response(JSON.stringify(data), {
    status,
    headers: {
      'Content-Type': 'application/json',
      'Access-Control-Allow-Origin': env.ALLOWED_ORIGIN
    }
  });
}
{% endhighlight %}

**Step 4: Create KV namespace for rate limiting**

For the rate limiting, we use Cloudflare KV (Key-Value store). KV is a simple persistent database: it stores data as key→value pairs. We use it to remember when each IP last submitted (prevent spam) So we need to create a KV namespace to store the value under the format : `'ratelimit:123.45.67.89': '1737732000000'`.

```bash
wrangler kv:namespace create "SCORE_KV"
```

Follow the instructions and add to `wrangler.jsonc`:

```jsonc
{
  "kv_namespaces": [
    {
      "binding": "SCORE_KV",
      "id": "your_kv_id_here"
    }
  ]
}
```

**Step 5: Set your secrets**

```bash
# GitHub personal access token
wrangler secret put GITHUB_TOKEN

# Repo (format: username/repo-name)
wrangler secret put GITHUB_REPO

# GitHub Pages URL
wrangler secret put ALLOWED_ORIGIN
```

**Step 6: Deploy**

```bash
wrangler deploy
```

You'll get a URL like: `https://chantal.workers.dev`

## Part 4: The GitHub Actions Workflow

Create `.github/workflows/update-high-score.yml`:

{% highlight yaml %}
name: update-high-score
on:
  repository_dispatch:
    types: [update-score]

permissions:
  contents: write
  pull-requests: write

jobs:
  update:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Get current high score
        id: current_score
        run: |
          git fetch origin score-updates 2>/dev/null || true

          # Check PR branch first (source of truth)
          if git rev-parse --verify origin/score-updates >/dev/null 2>&1; then
            CURRENT=$(git show origin/score-updates:_data/best_scores.yml | yq eval '.chantalpanic.score')
            echo "current=$CURRENT" >> $GITHUB_OUTPUT
          else
            # No PR yet, check master
            CURRENT=$(yq eval '.chantalpanic.score // 0' _data/best_scores.yml)
            echo "current=$CURRENT" >> $GITHUB_OUTPUT
          fi

      - name: Check if new score is higher
        id: check_score
        env:
          NEW_SCORE: ${{ github.event.client_payload.score }}
          CURRENT_SCORE: ${{ steps.current_score.outputs.current }}
        run: |
          if [ "$NEW_SCORE" -gt "$CURRENT_SCORE" ]; then
            echo "is_higher=true" >> $GITHUB_OUTPUT
            echo "✅ New score is higher"
          else
            echo "is_higher=false" >> $GITHUB_OUTPUT
            echo "❌ Score not higher, skipping"
          fi

      - name: Update score in YAML
        if: steps.check_score.outputs.is_higher == 'true'
        env:
          SCORE: ${{ github.event.client_payload.score }}
          PLAYER: ${{ github.event.client_payload.player }}
          TIMESTAMP: ${{ github.event.client_payload.timestamp }}
        run: |
          yq eval -n \
            '.chantalpanic.score = env(SCORE) |
             .chantalpanic.name = env(PLAYER) |
             .chantalpanic.timestamp = env(TIMESTAMP)' \
            > _data/best_scores.yml

      - name: Create or update PR
        if: steps.check_score.outputs.is_higher == 'true'
        env:
          SCORE: ${{ github.event.client_payload.score }}
          PLAYER: ${{ github.event.client_payload.player }}
          OLD_SCORE: ${{ steps.current_score.outputs.current }}
          GH_TOKEN: ${{ github.token }}
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"

          git checkout -B score-updates
          git add _data/best_scores.yml
          git commit -m "🎮 New high score: $SCORE by $PLAYER"
          git push -f origin score-updates

          # Check if PR exists
          PR_COUNT=$(gh pr list --head score-updates --base master --json number --jq length)

          if [ "$PR_COUNT" = "0" ]; then
            # Create new PR
            gh pr create \
              --title "🎮 High Score Update" \
              --body "**Score:** $SCORE | **Player:** $PLAYER | **Previous:** $OLD_SCORE" \
              --head score-updates \
              --base master
          else
            # Add comment to existing PR
            PR_NUMBER=$(gh pr list --head score-updates --base master --json number --jq '.[0].number')
            gh pr comment $PR_NUMBER \
              --body "🆕 **Score:** $SCORE | **Player:** $PLAYER | **Previous:** $OLD_SCORE"
          fi

      - name: Score not high enough
        if: steps.check_score.outputs.is_higher == 'false'
        run: |
          echo "No update needed"
{% endhighlight %}

**What this does:**

1. **Gets triggered** by the Cloudflare Worker via `repository_dispatch`
2. **Checks current score** from the PR branch (if exists) or master
3. **Only updates if higher** - prevents lowering scores
4. **Creates/updates a PR** - to review before merging!
5. **Force-pushes** - each new score replaces the previous PR content

### Enable PR Creation

To allow github actions to create and approve pull requests, you need to give it the right permissions.
So we need to go to repo Settings → Actions → General → Workflow permissions:
- ☑ Allow GitHub Actions to create and approve pull requests

## The Complete Flow Visualized

```
┌─────────────────────────────────────────────────────────────┐
│  Player beats high score: 1234 by "BOB"                    │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
         ┌───────────────────────┐
         │   Pico-8 Game         │
         │   poke(0x5f83, 1234)  │
         └───────────┬───────────┘
                     │
                     ▼
         ┌───────────────────────┐
         │   JavaScript          │
         │   Detects GPIO change │
         │   setInterval polling │
         └───────────┬───────────┘
                     │
                     ▼
         ┌───────────────────────────────────┐
         │   Cloudflare Worker               │
         │   ✓ Validates score & name        │
         │   ✓ Rate limits (30s per IP)      │
         │   ✓ Holds GitHub token safely     │
         └───────────┬───────────────────────┘
                     │
                     ▼
         ┌───────────────────────────────────┐
         │   GitHub API                      │
         │   POST /repos/.../dispatches      │
         │   Triggers workflow               │
         └───────────┬───────────────────────┘
                     │
                     ▼
         ┌───────────────────────────────────┐
         │   GitHub Actions Workflow         │
         │   1. Checks if score is higher    │
         │   2. Updates best_scores.yml      │
         │   3. Creates/updates PR           │
         └───────────┬───────────────────────┘
                     │
                     ▼
         ┌───────────────────────────────────┐
         │   Pull Request Created            │
         │                                   │
         └───────────┬───────────────────────┘
                     │
                     ▼
         ┌───────────────────────────────────┐
         │   Review & Merge                  │
         │   (Manual approval)               │
         └───────────┬───────────────────────┘
                     │
                     ▼
         ┌───────────────────────────────────┐
         │   Jekyll Rebuilds                 │
         │   New score live on site!         │
         └───────────────────────────────────┘
```

## What I Learned

**Static sites can be dynamic** - with the right architecture. The key is using serverless functions as a secure bridge between client and backend.

**Security is layered** - validation happens at multiple points:
- Worker validates format
- GitHub Actions validates score is higher
- Manual PR review is final check

**Rate limiting is essential** - without it, someone could spam your workflow and exhaust my GitHub Actions minutes.

**Being pragmatic is underrated** - a simple `setInterval()` is the most pragmatic solution, as well as a simple PR review mechanism!

## Potential Improvements

- **Webhooks**: Have GitHub notify my site when PR is merged (faster updates)
- **Multiple games**: Generalize the system to handle different games
- **Leaderboard**: Store top 10 scores instead of just #1
- **Replay protection**: Prevent submitting the same score twice

But for now, this works perfectly for my nephews' guinea pig game!

## Resources

- [Cloudflare Workers Docs](https://developers.cloudflare.com/workers/)
- [GitHub Actions Docs](https://docs.github.com/en/actions)
- [Pico-8 GPIO Reference](https://www.lexaloffle.com/dl/docs/pico-8_manual.html#GPIO)
- [My Complete Implementation](https://github.com/ABraconnier/antoinebraconnier.github.io)

---

*Part 3 of the "Pico-8 + Jekyll" series - The finale!*

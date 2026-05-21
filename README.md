# My Ritual Site

A Magic: The Gathering deck site built with [Ritual](https://github.com/sloshy/ritual).

## Getting Started

Add your decks to the `decks/` directory as Markdown files. For example:

```sh
ritual new-deck "My Commander Deck"
```

## Deploying

This repository is configured with a GitHub Action that automatically builds and
deploys your site to GitHub Pages on every push to `main`.

The action downloads Ritual, fetches the latest card data from Scryfall, builds
your site, and deploys it. Both the Ritual binary and Scryfall cache are
persisted between runs so subsequent builds are fast.

### Customizing the Ritual version

By default the action downloads the latest Ritual release. To pin a specific
version, create a GitHub Actions repository variable called `RITUAL_VERSION`
set to the release tag (e.g. `v1.0.0`).

## Setup

Make sure GitHub Pages is enabled in your repository settings:

1. Go to **Settings → Pages**
2. Under **Source**, select **GitHub Actions**

# MetroWebCacheIssue2024-08-09

This reproduces an issue with metro cache-key overlap, in a monorepo that has more than one expo project.

<img src=https://github.com/user-attachments/assets/25f92939-1269-4f01-9dfb-694bdc9f5225 width=550 />

## Issue Context

After upgrading to Expo SDK 51, and adpting expo web projects from webpack to metro web, we noticed that sometimes one project's build would have environment variables from a different project in the monorepo.

The issue was intermittent and only happened in pipelines with multiple web builds.

We determined that metro was resolving the cached app config, from a previous project's build, because they had the same cache-key.

<details>
  <summary>Additional Details</summary>

## Issue Context

After upgrading to Expo SDK 51, and adpting expo web projects from webpack to metro web, we noticed that sometimes one project's build would have environment variables from a different project in the monorepo.

The issue was intermittent and only happened in pipelines with multiple web builds.

We determined that metro was resolving the cached app config, from a previous project's build, because they had the same cache-key.

## Findings

NOTE: I am not an expert.

When metro is building it compiles and caches modules

> [cacheStores](https://metrobundler.dev/docs/configuration#cachestores) - When Metro needs to transform a module, it first computes a machine-independent cache key for that file, and uses it to try to read from each of the stores in order.

The cache keys for these modules seems to be computed by hashing the project's `metro.config.js` [[code](https://github.com/expo/expo/blob/43a79fb63c909e4770b22c0eae0b8b6a82b8a97a/packages/%40expo/metro-config/src/transform-worker/metro-transform-worker.ts#L737-L758)]

In a monorepo with multiple expo projects, all using identical `metro.config.js` files, this can result in the same cache key being used for multiple projects, which can lead to cache pollution.

For example, if a given pipeline included multiple expo web builds, then whichever project went first would be cached, and subsequent projects would resolve the first project's cached module assets.

In addition to build (expo export) this appears to also be happening when running the dev server.

</details>

<details>
  <summary>Workaround</summary>

### Workaround

If you edit the `metro.config.js` files so that each projects is differentiated, this appears to be sufficient to individualizing the cache keys, avoiding config.

For exampe this works, even though it'd be the same for each project, it differentiates when the `metro.config.js` file is compiled.

```js
const { getDefaultConfig } = require('expo/metro-config');

const config = getDefaultConfig(__dirname);

// differentiate metro cache between projects, see: https://metrobundler.dev/docs/configuration#cacheversion
config.cacheVersion = require('./project.json').name;

module.exports = config;
```

But I think even just adding a random key works too

```js
const { getDefaultConfig } = require('expo/metro-config');

const config = getDefaultConfig(__dirname);

config.projectNameCacheKey = 'my project name';

module.exports = config;
```

</details>

## Repro Steps

There are two expo web projects — each has a `.env` file with `PROJECT_NAME` where the value is specific to that project.

If the app config is cached and leaking between the projects then one of the will be reused.

### Setup

```
yarn install
```

### Build Web

```
yarn nx run hopping-heron-app:export --platform web --skip-nx-cache \
yarn nx run mysterious-macropod-app:export --platform web --skip-nx-cache
```

> [!NOTE]
> Might need to do this multiple times

### Check

The `yarn run test` script just greps both dists for the environment variable `PROJECT_NAME`

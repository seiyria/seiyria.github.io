---
title:  "A Better Deployment Process (Monorepos, CapRover, and Netlify)"
date:   2020-12-12 14:00:00
categories: monorepo, deployment, caprover, netlify
---

Deployment, one of the first pipelines to set up and one of the most important pipelines to have. I recently set up a really handy deployment process with Github Actions and wanted to share more, for anyone who finds themselves in a similar situation. Here's an overview of my situation:



- I have a monorepo, which makes deployment somewhat harder than I had hoped
- I wanted to deploy the backend to a private server with CapRover (this post will assume that CapRover is set up already)
- I wanted to deploy the frontend to Netlify, without building every time the backend updates
- I wanted the ability to deploy independently, together, manually, and automatically

For reference, I'll share [a link to my project](https://github.com/LandOfTheRair/LandOfTheRair) so it'll be easy to look up the finer points of my implementation.

A few problems I ran into upfront:

- CapRover makes deployment somewhat difficult if you have a monorepo, and I personally dislike vendor-specific files
- Netlify makes deployment somewhat difficult if you have a monorepo (as well), and I simply could not get their `build.ignore` settings to work, and the vendor-specific files too
- Docker

# Client Action

So, first, lets work on the manual process for Netlify w/ GitHub Actions, since it's so much easier. Thankfully, there's a Netlify action that makes this easy:

```
- name: Deploy to Netlify
  uses: nwtgck/actions-netlify@v1.1
  with:
    publish-dir: './client/dist'
    production-branch: master
    github-token: ${{ secrets.GITHUB_TOKEN }}
    deploy-message: "Deploy from GitHub Actions"
    enable-pull-request-comment: false
    enable-commit-comment: true
    overwrites-pull-request-comment: true
  env:
    NETLIFY_AUTH_TOKEN: ${{ secrets.NETLIFY_AUTH_TOKEN }}
    NETLIFY_SITE_ID: ${{ secrets.NETLIFY_SITE_ID }}
```

Of course, this assumes the app is built to `client/dist` first, but from there, it's straightforward - turn off Netlify auto deploys, since this will cover it.  [Here's my full action](https://github.com/LandOfTheRair/LandOfTheRair/blob/master/.github/workflows/manual-deploy-client.yml) for client side manual deployments.

# Server Action

The server action, for me, was much more difficult since I'm not well-versed with Docker. First, I had to come up with a Dockerfile (this is the easiest way to deploy to CapRover, IMO). I had some issues trying to find one that would work with my old version of cWS, turns out I had to use node 13.14 specifically. But that's neither here nor there, here's the Dockerfile I came up with ([current version here](https://github.com/LandOfTheRair/LandOfTheRair/blob/master/Dockerfile)):

```
FROM node:13.14.0-alpine
RUN apk add --no-cache libc6-compat
RUN ln -s /lib/libc.musl-x86_64.so.1 /lib/ld-linux-x86-64.so.2
RUN mkdir -p /usr/src/app
WORKDIR /usr/src/app
ADD https://www.google.com /time.now
COPY ./package.json /usr/src/app
COPY ./package-lock.json /usr/src/app
COPY ./tsconfig.json /usr/src/app
COPY ./server /usr/src/app/server
COPY ./shared /usr/src/app/shared
RUN npm install
RUN cd server && npm run setup && npm cache clean --force
RUN cd server/content && npm install --unsafe-perm
RUN cd server && npm run build
ENV NODE_ENV production
ENV PORT 80
EXPOSE 80
CMD cd server && npm start
```

I was running into some weird issues with needing a particular native function from `ld-linux-x86-64.so.2`, which `alpine` doesn't have. I searched around and found that I can symlink a different `.so` file and make this work, so that was one problem sorted out.

After that, I ran into some issues installing a sub-repo that gets cloned, which threw some weird error that ended up being an npm issue, fixed by adding `--unsafe-perm` to that npm install. Everything else is pretty straightforward for people who use docker, I imagine. 

One might notice I structure things a bit differently than most. I think in other Dockerfiles I've seen copying a single `package.json` file, installing, and going from there. Here, I had to copy `server/`, `shared/`, and the root `package.json` file to get the install process correct.

From there, the action is really straightforward, but note that it _must_ be run on linux. It looks like this:

```
- name: Deploy Server
  uses: AlexxNB/caprover-action@v1
  with:
    server: 'https://captain.server.rair.land'
    password: '${{ secrets.CAPROVER_PASSWORD }}'
    appname: 'game'
```

This action is yet a bit more convoluted because it does a build before it starts doing anything in the Dockerfile, then it does it again in the Dockerfile, but the time overhead isn't significant enough for me to want to fix it. [Here's my full action](https://github.com/LandOfTheRair/LandOfTheRair/blob/master/.github/workflows/manual-deploy-server.yml) for server deployments.

# Automatically Deploying

The two actions above handle the manual deployments, if needed. Seeing the full files shows how to set up manual deployments with GitHub Actions, but the tl;dr is `on: workflow_dispatch`.

Now, that's nice to have for testing, hotfixing, or whatever the case may be, but in general this is something worth automating. Thankfully, GitHub Actions is super flexible! It's really easy to compose the two manual tasks above into something that only deploys on tags ([the full action is here](https://github.com/LandOfTheRair/LandOfTheRair/blob/master/.github/workflows/release-tags.yml)):

```
name: Release New Tags

on:
  push:
    tags:
      - 'v*' # Any pushed tag

jobs:
  build:
    name: Create Release

    runs-on: ${{ matrix.os }}

    strategy:
      matrix:
        # os: [macos-latest, ubuntu-latest, windows-latest]
        os: [ubuntu-latest]
        node-version: [15.x]

    steps:
    - uses: actions/checkout@v2
    
    - name: Use Node.js ${{ matrix.node-version }}
      uses: actions/setup-node@v1
      with:
        node-version: ${{ matrix.node-version }}
    
    - run: npm install

    - run: npm run lint
      
    - run: npm run build

    - name: Deploy Server
      uses: AlexxNB/caprover-action@v1
      with:
        server: 'https://captain.server.rair.land'
        password: '${{ secrets.CAPROVER_PASSWORD }}'
        appname: 'game'

    - name: Deploy to Netlify
      uses: nwtgck/actions-netlify@v1.1
      with:
        publish-dir: './client/dist'
        production-branch: master
        github-token: ${{ secrets.GITHUB_TOKEN }}
        deploy-message: "Deploy from GitHub Actions"
        enable-pull-request-comment: false
        enable-commit-comment: true
        overwrites-pull-request-comment: true
      env:
        NETLIFY_AUTH_TOKEN: ${{ secrets.NETLIFY_AUTH_TOKEN }}
        NETLIFY_SITE_ID: ${{ secrets.NETLIFY_SITE_ID }}
```

This would automate client and server deployments on tags, but it could easily be changed to on push, or on any event GitHub Actions supports.

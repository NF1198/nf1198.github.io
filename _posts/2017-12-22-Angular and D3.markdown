---
layout: post
title:  "Using D3 with Angular"
date:   2017-12-22 13:40:00 -0600
categories: Angular D3 TypeScript
customjs:
 - https://ajax.googleapis.com/ajax/libs/jquery/3.1.0/jquery.min.js
---

Yes. You can use [D3](https://d3js.org/)(v4) with [Angular](https://angular.io/)(v5). Two usage models are possible:

* Bundle D3 with your app
* Load D3 from CDN

This post only describes how to load D3 in an Angular app, not how to use it.

note: this post assumes you're using the [angular-cli](https://cli.angular.io/),
and that you build your app using `ng build --prod`.

## Option 1: Bundling D3 with your app

This is the simplest and most direct way to use D3. It also 
follows the standard workflow for using an external library with Angular. D3 
components will get bundled into your app. Refer to the 
[d3 documentation](https://github.com/d3/d3/wiki#installing) for specifics.

First, install **d3** and **@types/d3**:

    npm install --save d3 @types/d3

Second, include and use D3 in your modules. If you * include D3, the production
build will be slightly larger.

    import * as d3 from 'd3';

Resulting artifact size: **377K**
    
    377K Dec 22 12:22 main.fa51e189cad1f5199229.bundle.js

Alternatively, selective imports *may* result in a smaller build.

    import { select } from 'd3';

Resulting artifact size: **254K**

    254K Dec 22 12:33 main.cc2e8b573c2053c38c33.bundle.js

## Option 2: Load D3 from CDN

This is the best option for web-hosted apps and is fully compatible with the
normal Angular workflow. No hacks necessary!

**First**, you won't need to install d3. If you've already installed it, simply 
remove the dependency from `package.json` and delete the `d3*` folders from 
`node_modules`. You will need the typings, however:

    npm install --save @types/d3

**Second**, you need to add a `<script>` tag to your `index.html` file to load
d3 from a CDN or other source. Use the CDN of choice and copy-paste the tag.

**Finally**, setup your Angular app to refer to the d3 UMD.

1. Add to `typings.d.ts`: (declares the d3 UMD to TypeScript)

    `declare var d3;`

2. Prepend modules that use d3: (enables intelli-sense in VSCode).

    `/// <reference types="d3" />`

You **do not need to `import` or `require` d3** in your modules.

The resulting production build will not include d3 and should not throw any errors.

Resulting artifact size: **149K**

    ng build --prod
    ...
    149K Dec 22 13:18 main.172b2841e074909059af.bundle.js

## Versions used (2017-12-22)

    npm --version
    5.6.0

    ng --version
    Angular CLI: 1.6.2
    Node: 8.6.0
    OS: linux x64
    Angular: 5.1.2

    "d3": "^4.12.0"
    "@types/d3": "^4.12.0"

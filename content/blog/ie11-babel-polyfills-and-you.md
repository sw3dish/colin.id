---
title: "IE11, Babel, Polyfills and You"
date: 2020-10-18T10:38:23-04:00
draft: false
tags: [
  "javascript",
  "tooling"
]
---

**_NOTE: This isn't the full story on polyfilling for IE11. I've written an update [here]() outlining the rest of what I found._**

*Hi! Emerging from COVID-19 isolation with something that recently cropped up at work.*

## IE11

As web developers, we all like to bash Internet Explorer.

That's fine -- **you** probably wouldn't use it.
**You** know IE11 has little ES6 support, is 7 years old, and there are better options.

But here's the thing -- IE still has [~5% market share](https://gs.statcounter.com/browser-market-share/desktop/united-states-of-america) on desktop in the US. I'm not saying you have to support it -- but 5% is a decent amount of %.
Especially if you can get ad revenue out of those 5%. IE's even more prevalent for [the company that I work for](https://industrydive.com) -- it's a business media company and their journalists write in a variety of industries.
Some of these industries are slow to upgrade their IT infrastructure, so IE makes up an even larger share of our users.

So, when we started writing React for a few features on our site, we were already cautious. We knew that React probably used
ES6 code and that IE11 would probably choke on it.

## Babel

We reached for Babel to transpile our ES6 down to ES5, which IE11 can definitely handle. We popped some code into our Webpack config file and went about our day. Easy!

Let's look at the code in our `webpack.config.js`:

```js
{
  module.exports {
    module: {
      rules: [
        {
          test: /.(js|jsx)$/,
          exclude: ['/node_modules/'],
          use:
            {
              loader: 'babel-loader',
              options: {
                presets: ['@babel/preset-env', '@babel/preset-react']
              },
            },
        },
      ]
    }
}
```

Looks like a perfectly normal rule for Webpack. We're running our `.js` and `.jsx` files through Babel using
their handy `preset-env` and `preset-react`. They're collections of plugins that tell Babel how we want to transpile.
We want `preset-react` to handle JSX and `preset-env` to transpile for different browsers.

Well, which browsers do we want to transpile for? `preset-env` lets you specify this using [Browserslist](https://github.com/browserslist/browserslist).

```
> 1%
Last 2 versions
not IE 10
```

We've decided to target the last 2 versions browsers that have more than 1% market share. (But not IE10.)

This is great! `preset-env` will transpile our ES6 code so that those browsers can run them. We don't have to do anything -- when we `webpack` our files to bundle modules, our code will be piped through `babel` so it's transpiled. So off we go, writing some React code here, some new ES6 there.

And all was right with the world -- until one day, we got the error in IE11:
`Object doesn't support property or method 'find'`. This is, correctly, telling us that IE11 does not, can not, and will not support `Array.prototype.find`.

But what gives? We're using `preset-env` and we're targeting IE11. We haven't had any problems with our other ES6 code -- arrow functions, imports, `const`, `let`, template strings.

## Polyfills

The answer is a piece of important `preset-env` config that we left out.

[`useBuiltIns`](https://babeljs.io/docs/en/babel-preset-env#usebuiltins) is an option that "configures how @babel/preset-env handles polyfills." By default, this is set to `false`.

Babel wasn't polyfilling any built-in functions!

But, we can fix this by properly configuring the `useBuiltIns` option to use either `usage` or `entry`.
You can read more in [the docs](https://babeljs.io/docs/en/babel-preset-env#usebuiltins), but to sum up -- `usage` adds polyfills when they are needed in each file and `entry` adds polyfills in a single entry file to your app.


We're using `usage` as we found it made the bundles smaller and bundled faster -- but your mileage may vary. We also want to use the newest `core-js` version (`core-js@3`) as `core-js` is what Babel will use for polyfilling.

That looks like (again in `webpack.config.js`):
```js
{
  module.exports {
    module: {
      rules: [
        {
          test: /.(js|jsx)$/,
          exclude: ['/node_modules/'],
          use:
            {
              loader: 'babel-loader',
              options: {
                presets: [
                  [
                    '@babel/preset-env',
                    {
                      useBuiltIns: 'usage',
                      corejs: 3,
                    }
                  ],
                  '@babel/preset-react'
                ]
              },
            },
        },
      ]
    }
}
```



## You (or in this case, me)

So what should you keep in mind if you support IE11?

1. `preset-env` isn't fire-and-forget. Make sure you read the config options, especially if you're supporting less than modern browsers.

2. Test your features regularly in **all** of the browsers you support. If we had tested this in a VM, we probably would've caught it before shipping.

3. I didn't cover this above, but keep your config in files related to the tool that needs it.
Part of the difficulty finding this missing option came from us having 3 different places that our `babel` config lived,
one of which was our `webpack` config. Which one applied to what code? We've since moved it all to a `.babelrc.json` file in our root, making sure that all of
the code that we're writing will be transpiled to the same targets.

But if you're supporting IE11, thanks!
Sometimes, users can't switch off of IE, even though they might want to.
Maybe they're not allowed to by their company.
Or, maybe, they don't know just don't know the difference between browsers.
Ultimately, the more browsers we support, the more users we can reach.
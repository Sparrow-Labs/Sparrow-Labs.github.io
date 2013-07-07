---
layout: post
title: About Node.js and precompiling assets under Rails
author: kiligarikond
authorname: Yovoslav Ivanov
---

# {{ page.title }}


A couple of days ago a shocking mystery revealed it's dark face as a fellow developer at one of our customers tried to precompile
his rails assets and got a strange error, that one of his _js.coffee_ files appeared to have an incompatible character encoding and
thus could not be parsed.

<!--- end preview -->

Now for this post to make sense to you, let's first talk a little bit about the setup. Our customer develops a rails application, which
has to be deployed on Windows. Since deployment possibilities under Windows are kind of limited we precompile the assets on our developer machines and later copy over everything via capistrano.
Having this deployment setup in mind let's have a look at the mystical character encoding error on our fellow dev's console:

```bash
$ rake assets:precompile
rake aborted!

(RegexpError) incompatible character encoding: /[\u0080-\uffff]/

  (in /Users/fellow_dev/Temp/bla_deployment/bla/vendor/bundle/jruby/1.9/bundler/gems/bla-de94bc4d0607/app/assets/javascripts/bla/bla.js.coffee)
```

The shock on both sides was, that on my machine everything worked fine and apparently we had the same setup. Same ruby (actually jruby), same repository, same RAILS\_ENV. It took me actually half an hour to trace down what was going wrong. When i ran the rake task with a _--trace_ option i recognized, that the error comes from ExecJS. A quick quote from the ExecJS's README actually showed the solution:

_ExecJS lets you run JavaScript code from Ruby. It automatically picks the best runtime available to evaluate your JavaScript program, then returns the result to you as a Ruby object._

So ExecJS automatically picks the best runtime to evaluate our JavaScript code. Reading further i saw, that it supports among other runtimes, _Node.js_ and _Apple JavaScriptCore_, the later is shipped with Mac OS X. And here trouble stroke on our colegue's machine.

At Sparrow-Labs UG we develop our main product's server side in _Node.js_, so ExecJS picked _Node.js_ as the best suiting runtime. On our fellow dev's machine it picked whatever was there and this was _Apple JavaScriptCore_. Now the interesting part is that the asset precompiling on his environment broke just a few weeks ago, after he updated Mac OS X.

# Conclusion

So whenever you have a production environment, that is not suitable for asset precompiling make sure you install _Node.js_ (or another suitable runtime) and the coffee-script compiler on your machine before trying to precompile the assets.
This is how i did it on my Mac Book Air:

```bash
$ brew install nodejs
$ npm install -g coffee-script
```
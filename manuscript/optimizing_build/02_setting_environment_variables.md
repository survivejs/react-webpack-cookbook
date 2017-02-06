# Setting Environment Variables

Sometimes a part of your code should execute only during development. Or you might have experimental features in your build that are not ready for production yet. This code should not end up in the production build.

As JavaScript minifiers can remove dead code (`if (false)`), we can build on top of this idea and write code that gets transformed into this form. Webpack's `DefinePlugin` enables replacing **free variables** so that we can convert `if (process.env.NODE_ENV === 'development')` kind of code to `if (true)` or `if (false)` depending on the environment.

Many packages rely on this behavior. React is perhaps the most known example of an early adopter of the technique. Using `DefinePlugin` can bring down the size of your React production build somewhat as a result and you may see a similar effect with other packages as well.

## The Basic Idea of `DefinePlugin`

To understand the idea of `DefinePlugin` better, consider the example below:

```javascript
var foo;

// Not free due to "foo" above, not ok to replace
if (foo === 'bar') {
  console.log('bar');
}

// Free since we don't refer to "bar", ok to replace
if (bar === 'bar') {
  console.log('bar');
}
```

If we replaced `bar` with a string like `'foobar'`, then we would end up with code like this:

```javascript
var foo;

// Not free due to "foo" above, not ok to replace
if (foo === 'bar') {
  console.log('bar');
}

// Free since we don't refer to "bar", ok to replace
if ('foobar' === 'bar') {
  console.log('bar');
}
```

Further analysis shows that `'foobar' === 'bar'` equals `false` so a minifier gives us:

```javascript
var foo;

// Not free due to "foo" above, not ok to replace
if (foo === 'bar') {
  console.log('bar');
}

// Free since we don't refer to "bar", ok to replace
if (false) {
  console.log('bar');
}
```

And based on this a minifier can eliminate the `if` statement as it has become dead code:

```javascript
var foo;

// Not free, not ok to replace
if (foo === 'bar') {
  console.log('bar');
}

// if (false) means the block can be dropped entirely
```

This is the core idea of `DefinePlugin`. We can toggle parts of code using this kind of mechanism. A good minifier is able to perform the analysis for us and enable/disable entire portions of it as we prefer.

T> [babel-plugin-transform-define](https://www.npmjs.com/package/babel-plugin-transform-define) can achieve the same result without webpack.

## Setting `process.env.NODE_ENV`

Given we are using React in our project and it happens to use the technique, we can try to enable `DefinePlugin` and see what it does to our production build.

As before, encapsulate this idea to a function. It is important to note that given the way webpack replaces the free variable, we should push it through `JSON.stringify`. We'll end up with a string like `'"demo"'` and then webpack will insert that into the slots it finds.

**webpack.parts.js**

```javascript
...

leanpub-start-insert
exports.setFreeVariable = function(key, value) {
  const env = {};
  env[key] = JSON.stringify(value);

  return {
    plugins: [
      new webpack.DefinePlugin(env),
    ],
  };
};
leanpub-end-insert
```

We can connect this with our configuration like this:

**webpack.config.js**

```javascript
...

module.exports = function(env) {
  if (env === 'production') {
    return merge([
      common,
      {
        ...
      },
leanpub-start-insert
      parts.setFreeVariable(
        'process.env.NODE_ENV',
        'production'
      ),
leanpub-end-insert
      ...
    ]);
  }

  ...
};
```

Execute `npm run build` again, and you should see improved results:

```bash
Hash: 9695368f0432ed1946ad
Version: webpack 2.2.1
Time: 3134ms
                                 Asset       Size  Chunks                    Chunk Names
                                app.js  640 bytes       1  [emitted]         app
  674f50d287a8c48dc19ba404d20fe713.eot     166 kB          [emitted]  [big]
  b06871f281fee6b241d60582ae9369b9.ttf     166 kB          [emitted]  [big]
af7ae505a9eed503f8b8e6982036873e.woff2    77.2 kB          [emitted]  [big]
 fee66e712a8a08eef5805a46892932ad.woff      98 kB          [emitted]  [big]
  9a0d8fb85dedfde24f1ab4cdb568ef2a.png    17.6 kB          [emitted]
                                  0.js  160 bytes       0  [emitted]
  912ec66d7572ff821749319396470bde.svg     444 kB          [emitted]  [big]
                             vendor.js      21 kB       2  [emitted]         vendor
                               app.css     3.5 kB       1  [emitted]         app
                              0.js.map  769 bytes       0  [emitted]
                            app.js.map    5.62 kB       1  [emitted]         app
                           app.css.map   84 bytes       1  [emitted]         app
                         vendor.js.map     261 kB       2  [emitted]         vendor
                            index.html  274 bytes          [emitted]
   [4] ./~/object-assign/index.js 2.11 kB {2} [built]
   [5] ./~/react/react.js 56 bytes {2} [built]
  [15] ./app/component.js 517 bytes {1} [built]
...
```

We went from 141 kB to 42 kB, and finally, to 21 kB. The final build is a little faster than the previous one as well.

Given the 21 kB can be served gzipped, it is somewhat reasonable. gzipping will drop around another 40% and it is well supported by browsers.

It is good to remember that we didn't include *react-dom* in this case and that would add around 100 kB to the final result. To get back to these figures, we would have to use a lighter alternative such as Preact or react-lite as discussed in the *Configuring React* chapter.

T> [babel-plugin-transform-inline-environment-variables](https://www.npmjs.com/package/babel-plugin-transform-inline-environment-variables) Babel plugin can be used to achieve the same effect. See [the official documentation](https://babeljs.io/docs/plugins/transform-inline-environment-variables/) for details.

T> `webpack.EnvironmentPlugin(['NODE_ENV'])` is a shortcut that allows you to refer to environment variables. It uses `DefinePlugin` internally and can be useful by itself in more limited cases. You can achieve the same effect by passing `process.env.NODE_ENV` to the custom function we made.

## Choosing Which Module to Use Based on Environment

The techniques discussed in this chapter can be useful for choosing entire modules depending on the environment. As seen above, `DefinePlugin` based splitting allows us to choose which branch of code to use and which to discard. This idea an be used to implement branching on module level. Consider the file structure below:

```bash
.
└── store
    ├── index.js
    ├── store.dev.js
    └── store.prod.js
```

The idea is that we will choose either `dev` or `prod` version of the store depending on the environment. It's that *index.js* which does the hard work like this:

```javascript
if(process.env.NODE_ENV === 'production') {
  module.exports = require('./store.prod');
} else {
  module.exports = require('./store.dev');
}
```

Webpack is able to pick the right code based on our `DefinePlugin` declaration and this code. It is good to note that we will have to use CommonJS module definition style here as ES6 module definition won't work by definition. ES6 definition doesn't allow dynamic behavior like this by design.

T> A related technique, **aliasing**, is discussed in the *Consuming Packages* chapter. You could alias to development or production specific file depending on the environment. The gotcha is that it will tie your setup to webpack in a tighter way than the solution above.

## Webpack Optimization Plugins

Webpack includes a collection of optimization related plugins, some of which we'll cover in greater detail in this book. In addition there are a few outside the core. I've listed the most important ones below:

* [compression-webpack-plugin](https://www.npmjs.com/package/compression-webpack-plugin) allows you to push the problem of generating compressed files to webpack. This can potentially save processing time on the server.
* `webpack.optimize.UglifyJsPlugin` allows you to minify output using different heuristics. Some of them might break code unless you are careful.
* `webpack.optimize.OccurrenceOrderPlugin` sorts module ids so that the most used modules get the shortest numeric ids. This can give slightly better size if you prefer to rely on number based ids over the book setup based on `HashedModuleIdsPlugin`.
* `webpack.optimize.AggressiveSplittingPlugin` allows you to split code into smaller bundles as discussed in the *Splitting Bundles* chapter. This can be particularly useful in HTTP/2 environment.
* `webpack.optimize.CommonsChunkPlugin` makes it possible to extract common dependencies into bundles of their own.
* `webpack.DefinePlugin` allows you to use feature flags in your code and eliminate the redundant code as discussed in this chapter.
* [lodash-webpack-plugin](https://www.npmjs.com/package/lodash-webpack-plugin) creates smaller Lodash builds by replacing feature sets with smaller alternatives leading to more compact builds.

## Conclusion

Even though simply setting `process.env.NODE_ENV` the right way can help a lot especially with React-related code, we can do better. Currently our build doesn't benefit on client level cache invalidation.

To achieve this, the build requires placeholders in which webpack can insert hashes that invalidate the files as we update the application.

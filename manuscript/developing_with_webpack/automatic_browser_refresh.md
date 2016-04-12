# Automatic Browser Refresh

Tools, such as [LiveReload](http://livereload.com/) or [Browsersync](http://www.browsersync.io/), allow us to refresh the browser as we develop our application. They can even avoid refresh for CSS changes. It is possible to set up the latter to work with Webpack by using [browser-sync-webpack-plugin](https://www.npmjs.com/package/browser-sync-webpack-plugin). In this case I'm going to show you how to use something more Webpack specific - namely *webpack-dev-server*.

*webpack-dev-server* is a development server running in-memory. It refreshes content automatically in the browser while you develop your application. It also supports an advanced feature known as Hot Module Replacement (HMR). In short, it provides a way to patch the browser state without a full refresh. This is particularly powerful with technology, such as React.

W> You should use *webpack-dev-server* strictly for development. If you want to host your application, consider other, standard solutions, such as Apache or Nginx.

## Getting Started with *webpack-dev-server*

To get started with *webpack-dev-server*, execute

```bash
npm i webpack-dev-server --save-dev
```

at the project root to get the server installed.

Just like in the previous chapter, we'll need to define an entry point to the `scripts` section of *package.json*. Given our *index.html* is below *./build*, we should let *webpack-dev-server* to serve the content from there. We'll move this to Webpack configuration later, but this will do for now:

**package.json**

```json
...
"scripts": {
leanpub-start-delete
  "build": "webpack"
leanpub-end-delete
leanpub-start-insert
  "build": "webpack",
  "start": "webpack-dev-server"
leanpub-end-insert
},
...
```

If you execute either *npm run start* or *npm start* now, you should see something like this at the terminal:

```bash
> webpack-dev-server

http://localhost:8080/webpack-dev-server/
webpack result is served from /
content is served from .../webpack-demo
Hash: 2dca5a3850ce5d2de54c
Version: webpack 1.12.14
```

The output means that the development server is running. If you open *http://localhost:8080/* at your browser, you should see something. If you try modifying the code, you should see output at your terminal. The problem is that the browser doesn't catch these changes without a hard refresh. That's something we need to resolve next.

![Hello world](images/hello_01.png)

T> If you fail to see anything at the browser, you may need to use a different port through *webpack-dev-server --port 3000* kind of invocation. One reason why the server might fail to run is simply because there's something else running in the port. You can verify this through a terminal command, such as `netstat -na | grep 8080`. If there's something running in the port 8080, it should display a message. The exact command may depend on your platform.

## Configuring Hot Module Replacement (HMR)

Hot Module Replacement gives us simple means to refresh the browser automatically as we make changes. The idea is that if we change our *app/component.js*, the browser will refresh itself. The same goes for possible CSS changes.

In order to make this work, we'll need to connect the generated bundle running in-memory to the development server. Webpack uses WebSocket based communication to achieve this.

To keep things simple, we'll let Webpack generate the client portion for us through the development server *inline* option. The option will include the client side scripts needed by HMR to the bundle that Webpack generates.

Beyond this we'll need to enable `HotModuleReplacementPlugin` to make the setup work. In addition I am going to enable HTML5 History API fallback as that is convenient default to have especially if you are dealing with advanced routing. Here's the setup:

**webpack.config.js**

```javascript
...
leanpub-start-insert
const webpack = require('webpack');
leanpub-end-insert

...

if(TARGET === 'start') {
leanpub-start-delete
  module.exports = merge(common, {});
leanpub-end-delete
leanpub-start-insert
  module.exports = merge(common, {
    devServer: {
      // Enable history API fallback so HTML5 History API based
      // routing works. This is a good default that will come
      // in handy in more complicated setups.
      historyApiFallback: true,
      hot: true,
      inline: true,
      progress: true,

      // Display only errors to reduce the amount of output.
      stats: 'errors-only',

      // Parse host and port from env to allow customization.
      //
      // If you use Vagrant or Cloud9, set
      // host: process.env.HOST || '0.0.0.0';
      //
      // 0.0.0.0 is available to all network devices
      // unlike default localhost
      host: process.env.HOST,
      port: process.env.PORT

      // If you want defaults, you can use a little trick like this
      // port: process.env.PORT || 3000
    },
    plugins: [
      new webpack.HotModuleReplacementPlugin()
    ]
  });
leanpub-end-insert
}

...
```

Execute `npm start` and surf to **localhost:8080**. Try modifying *app/component.js*. It should refresh the browser. Note that this is hard refresh in case you modify JavaScript code. CSS modifications work in a neater manner and can be applied without a refresh.

If you are using Windows and it doesn't refresh, see the following section for an alternative setup.

W> *webpack-dev-server* can be very particular about paths. If the given `include` paths don't match the system casing exactly, this can cause it to fail to work. Webpack [issue #675](https://github.com/webpack/webpack/issues/675) discusses this in more detail.

T> You should be able to access the application alternatively through **localhost:8080/webpack-dev-server/** instead of root. You can see all the files the development server is serving there.

## HMR on Windows and Ubuntu

The setup may be problematic on certain versions of Windows and Ubuntu. Instead of using `devServer` and `plugins` configuration, implement it like this:

**webpack.config.js**

```javascript
...

if(TARGET === 'start') {
  module.exports = merge(common, {});
}

...
```

**package.json**

```json
...
"scripts": {
  "build": "webpack",
leanpub-start-delete
  "start": "webpack-dev-server"
leanpub-end-delete
leanpub-start-insert
  "start": "webpack-dev-server --watch-poll --inline --hot"
leanpub-end-insert
},
...
```

Given this setup polls the filesystem, it is going to be more resource intensive. It's worth giving a go if the default doesn't work, though.

T> There are more details in *webpack-dev-server* issue [#155](https://github.com/webpack/webpack-dev-server/issues/155).

## Accessing the Development Server from Network

It is possible to customize host and port settings through the environment in our setup (i.e., `export PORT=3000` on Unix or `SET PORT=3000` on Windows). This can be useful if you want to access your server using some other device within the same network. The default settings are enough on most platforms.

To access your server, you'll need to figure out the ip of your machine. On Unix this can be achieved using `ifconfig`. On Windows `ipconfig` can be used. An npm package, such as [node-ip](https://www.npmjs.com/package/node-ip) may come in handy as well. Especially on Windows you may need to set your `HOST` to match your ip to make it accessible.

## Alternative Ways to Use *webpack-dev-server*

We could have passed *webpack-dev-server* options through terminal. I find it clearer to manage it within Webpack configuration as that helps to keep *package.json* nice and tidy.

Alternatively, we could have set up an Express server of our own and used *webpack-dev-server* as a [middleware](https://webpack.github.io/docs/webpack-dev-middleware.html). There's also a [Node.js API](https://webpack.github.io/docs/webpack-dev-server.html#api).

T> [dotenv](https://www.npmjs.com/package/dotenv) allows you to define environment variables through a *.env* file. This can be somewhat convenient during development!

W> Note that there are [slight differences](https://github.com/webpack/webpack-dev-server/issues/106) between the CLI and the Node.js API. This is the reason why some prefer to solely use the Node.js API.

## Conclusion

In this chapter you learned to set up Webpack to refresh your browser automatically. We can go a notch further and make this work beautifully with CSS files. We'll do that in the next chapter.

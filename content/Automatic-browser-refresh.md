When **webpack-dev-server** is running it will watch your files for changes. When that happens it rebundles your project and notifies browsers listening to refresh. To trigger this behavior you need to change your *index.html* file in the `build/` folder.

**build/index.html**

```html
<!DOCTYPE html>
<html>
  <head>
    <meta charset="UTF-8"/>
  </head>
  <body>
    <script src="http://localhost:8080/webpack-dev-server.js"></script>
    <script src="bundle.js"></script>
    <script src="http://localhost:8080/webpack-dev-server.js"></script>
  </body>
</html>
```

We added a script that refreshes the application when a change occurs. You will also need to add an entry point to your configuration:

**webpack.config.js**

```javascript
var path = require('path');

module.exports = {
    entry: [
      'webpack/hot/dev-server',
      'webpack-dev-server/client?http://localhost:8080',
      path.resolve(__dirname, 'app/main.js')
    ],
    output: {
        path: path.resolve(__dirname, 'build'),
        filename: 'bundle.js',
    },
};
```

Thats it! Now your application will automatically refresh on file changes.

## Default environment

In the example above we created our own *index.html* file to give more freedom and control. It is also possible to run the application from **http://localhost:8080/webpack-dev-server/bundle**. This will fire up a default *index.html* file that you do not control. It also fires this file up in an iFrame allowing for a status bar to indicate the status of the rebundling process.

> I discuss an alternative, `inline` based approach at the [Developing with Webpack](http://survivejs.com/webpack_react/developing_with_webpack/) chapter of *SurviveJS - Webpack and React*.

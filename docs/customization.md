# Neutrino Customization

No two JavaScript projects are ever the same, and as such there may be times when you will need to make modifications
to the way your Neutrino presets are building your project. Neutrino provides a mechanism to augment presets and
middleware in the context of a project without resorting to creating and publishing an entirely independent preset.

## Middleware formats

Before we delve into making customizations in `.neutrinorc.js`, it's important to note that this file can be in any
valid [middleware format](./middleware.md#formats) that Neutrino accepts. For project-based customization, it is
recommended to use the object format, and that will be the format we focus on for the remainder of this guide. Should
you need a lot of API customization, you may still opt to write your `.neutrinorc.js` file in the function format.

Your `.neutrinorc.js` file is a JavaScript module which will be required by Neutrino using Node.js. Any code written in
this file should be usable by the version of Node.js you have running on your system when running Neutrino. The
`.neutrinorc.js` file should export an object or function depending on which format you opt to use.

```js
module.exports = {
  /* make customizations */
};
```

```js
module.exports = (neutrino) => {
  /* make customizations */
};
```

**In a nutshell, the `.neutrinorc.js` file is wholly middleware**.

## Overriding Neutrino options

Neutrino has a number of useful options for customizing its behavior, and these can be overridden by using an
object at the `options` property:

### `options.root`

Set the base directory which Neutrino middleware and presets operate on. Typically this is the project directory where
the package.json would be located. If the option is not set, Neutrino defaults it to `process.cwd()`. If a relative
path is specified, it will be resolved relative to `process.cwd()`; absolute paths will be used as-is.

**It's recommended to always set this value, to ensure that tools such as ESLint work when run from a subdirectory
of the repository. If your `.neutrinorc.js` is in the root of the repository, use the value `__dirname` to achieve this.**

```js
module.exports = {
  options: {
    // Override to relative directory, resolves to process.cwd() + website
    root: 'website',
    // Override to absolute directory
    root: '/code/website'
  }
};
```

### `options.source`

Set the directory which contains the application source code. If the option is not set, Neutrino defaults it to `src`.
If a relative path is specified, it will be resolved relative to `options.root`; absolute paths will be used as-is.

```js
module.exports = {
  options: {
    // Override to relative directory, resolves to options.root + lib
    source: 'lib',
    // Override to absolute directory
    source: '/code/website/lib'
  }
};
```

### `options.output`

Set the directory which will be the output of built assets. If the option is not set, Neutrino defaults it to `build`.
If a relative path is specified, it will be resolved relative to `options.root`; absolute paths will be used as-is.

```js
module.exports = {
  options: {
    // Override to relative directory, resolves to options.root + dist
    output: 'dist',
    // Override to absolute directory
    output: '/code/website/dist'
  }
};
```

### `options.tests`

Set the directory that contains test files. If the option is not set, Neutrino defaults it to `test`.
If a relative path is specified, it will be resolved relative to `options.root`; absolute paths will be used as-is.

```js
module.exports = {
  options: {
    // Override to relative directory, resolves to options.root + testing
    tests: 'testing',
    // Override to absolute directory
    tests: '/code/website/testing'
  }
};
```

### `options.mains`

Set the main entry points for the application. If the option is not set, Neutrino defaults it to:

```js
{
  index: 'index'
}
```

Notice the entry point has no extension; the extension is resolved by webpack. If relative paths are specified,
they will be computed and resolved relative to `options.source`; absolute paths will be used as-is.

Multiple entry points and any page-specific configuration (if supported by the preset) can be specified like so:

```js
module.exports = {
  options: {
    mains: {
      // Relative path, so resolves to options.source + home.*
      index: 'home',

      // Absolute path, used as-is.
      login: '/code/website/src/login.js',

      // Long form that allows passing page-specific configuration
      // (such as html-webpack-plugin options in the case of @neutrinojs/web).
      admin: {
        entry: 'admin',
        // any page-specific options here (see preset docs)
        // ...
      }
    }
  }
};
```

## Mutating `neutrino.options`

While it is possible to mutate `neutrino.options` directly, this should be avoided.
Instead, it is _always recommended_ to pass an `options` object to ensure proper normalization.

```js
// Bad: Using function format, overriding `neutrino.options` properites.
// Paths will not be relative to `neutrino.options.root` as expected.
module.exports = neutrino => {
  Object.assign(neutrino.options, {
    source: 'lib',
    output: 'dist'
  });
}
```

```js
// Good: Using function format, setting `neutrino.options.*` properties directly.
module.exports = neutrino => {
  neutrino.options.source = 'lib';
  neutrino.options.output = 'dist';
}
```

```js
// Good: Use object format w/ `use` array
module.exports = {
  options: {
    source: 'lib',
    output: 'dist'
  },
  use: [/* ... */]
}
```

```js
// Good: Use object format w/ `use` function
module.exports = {
  options: {
    source: 'lib',
    output: 'dist'
  },
  use: neutrino => {
    neutrino.use(/* ... */);
  }
}
```

## Using middleware

By specifying a `use` array in your `.neutrinorc.js`, you can inform Neutrino to load additional middleware when it
runs, including any additional files you wish to include as middleware. Each item in this `use` array can be any
Neutrino-supported [middleware format](./middleware.md#formats).

In its simplest form, each item can be the string module name or path to middleware you wish Neutrino to require and
use for you:

```js
module.exports = {
  use: [
    '@neutrinojs/airbnb-base',
    '@neutrinojs/react',
    '@neutrinojs/jest',
    './override.js'
  ]
};
```

If your middleware module supports its own options, instead of referencing it by string, use an array pair of string
module name and options:

```js
module.exports = {
  use: [
    ['@neutrinojs/airbnb-base', {
      eslint: {
        rules: {
          semi: 'off'
        }
      }
    }],

    ['@neutrinojs/react', {
      html: { title: 'Epic React App' }
    }],

    '@neutrinojs/jest'
  ]
};
```

If you need to make more advanced configuration changes, you can even directly pass a function as middleware to `use`
and have access to the Neutrino API:

```js
module.exports = {
  use: [
    '@neutrinojs/airbnb-base',
    '@neutrinojs/react',
    '@neutrinojs/jest',
    (neutrino) => neutrino.config.module
      .rule('style')
      .use('css')
      .options({ modules: true })
  ]
};
```

## Environment-specific overrides

Sometimes you can only make certain configuration changes in certain Node.js environments, or you may choose to
selectively make changes based on the values of any arbitrary environment variable. This can be achieved by
conditionally applying middleware in `.neutrinorc.js`.

For example, if you wanted to include additional middleware when `NODE_ENV` is `production`:

```js
module.exports = {
  use: [
    process.env.NODE_ENV === 'production' ? '@neutrinojs/pwa' : false,
  ]
};
```

_Example: Turn on CSS modules when the environment variable `CSS_MODULES=enable`:_

```js
module.exports = {
  use: [
    (neutrino) => {
      // Turn on CSS modules when the environment variable CSS_MODULES=enable
      if (process.env.CSS_MODULES === 'enable') {
        neutrino.config.module
          .rule('style')
            .use('css')
              .options({ modules: true });
      }
    }
  ]
};
```

## Advanced configuration changes

Making deep or complex changes to Neutrino build configuration beyond what middleware options afford you can be done
using the function middleware format. If you wish, your entire `.neutrinorc.js` file can be a middleware function, but
typically this function can be inlined directly as an additional item in the `use` array.

If you're familiar with middleware from the Express/connect world, this works similarly. When using Express middleware,
you provide a function to Express which receives arguments to modify a request or response along its lifecycle. There
can be a number of middleware functions that Express can load, each one potentially modifying a request or response in
succession.

When you add a middleware function to `use`, this is typically used to override Neutrino's configuration, and you can
add as many functions as you wish in succession. Every preset or middleware that Neutrino has loaded follows this same
middleware successive pipeline.

The Neutrino API instance provided to your function has a `config` property that is an instance of
[webpack-chain](https://github.com/neutrinojs/webpack-chain). We won't go in-depth of all the configuration
possibilities here, but encourage you to check out the documentation for webpack-chain for instructions on your
particular use case. Just know that you can use webpack-chain to modify any part of the underlying webpack configuration
using its API.

This `neutrino.config` is an accumulation of all configuration up to this moment. All Neutrino middleware and presets
interact with and make changes through this config, which is all available to you. For example, if you are using the
presets `@neutrinojs/react` and `@neutrinojs/karma`, any config set can be extended, manipulated, or removed.

_Example: Neutrino's React preset adds `.jsx` as a module extension. Let's remove it._

```js
module.exports = {
  use: [
    '@neutrinojs/react',
    (neutrino) => neutrino.config.resolve.extensions.delete('.jsx')
  ]
};
```

_Example: Neutrino's Node.js preset has performance hints disabled. Let's re-enable them._

```js
module.exports = {
  use: [
    '@neutrinojs/node',
    (neutrino) => neutrino.config.performance.hints('error')
  ]
};
```

Remember, middleware can also have their own custom options for making some changes easier without having to resort to
interacting with the Neutrino API; see your respective middleware for details. See the [documentation on the
configuration API using webpack-chain](./webpack-chain.md) for all ways you can modify a config instance to solve
your use cases.

### Conditional configuration

Some plugins and rules are only available in certain environments. For example, the Web preset only exposes a `optimize-css`
plugin during production, leading to issues when trying to modify its settings, but throws an exception during
development.

_Example: Remove all arguments to the `optimize-css` plugin when using the Web preset._

```js
config.when(process.env.NODE_ENV === 'production', config => {
  config.plugin('optimize-css').tap(args => []);
});
```

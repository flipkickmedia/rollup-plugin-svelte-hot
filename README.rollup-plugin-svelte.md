# rollup-plugin-svelte [![CI](https://github.com/sveltejs/rollup-plugin-svelte/workflows/CI/badge.svg)](https://github.com/sveltejs/rollup-plugin-svelte/actions)

Compile Svelte components.


## Installation

```bash
npm install --save-dev svelte rollup-plugin-svelte
```

Note that we need to install Svelte as well as the plugin, as it's a 'peer dependency'.


## Usage

```js
// rollup.config.js
import svelte from 'rollup-plugin-svelte';
import resolve from '@rollup/plugin-node-resolve';

export default {
  input: 'src/main.js',
  output: {
    file: 'public/bundle.js',
    format: 'iife'
  },
  plugins: [
    svelte({
      // By default, all ".svelte" files are compiled
      extensions: ['.my-custom-extension'],

      // You can restrict which files are compiled
      // using `include` and `exclude`
      include: 'src/components/**/*.svelte',

      // Optionally, preprocess components with svelte.preprocess:
      // https://svelte.dev/docs#svelte_preprocess
      preprocess: {
        style: ({ content }) => {
          return transformStyles(content);
        }
      },

      // Emit CSS as "files" for other plugins to process. default is true
      emitCss: false,

      // Warnings are normally passed straight to Rollup. You can
      // optionally handle them here, for example to squelch
      // warnings with a particular code
      onwarn: (warning, handler) => {
        // e.g. don't warn on <marquee> elements, cos they're cool
        if (warning.code === 'a11y-distracting-elements') return;

        // let Rollup handle all other warnings normally
        handler(warning);
      },

      // You can pass any of the Svelte compiler options
      compilerOptions: {

        // By default, the client-side compiler is used. You
        // can also use the server-side rendering compiler
        generate: 'ssr',

        // ensure that extra attributes are added to head
        // elements for hydration (used with generate: 'ssr')
        hydratable: true,

        // You can optionally set 'customElement' to 'true' to compile
        // your components to custom elements (aka web elements)
        customElement: false
      },

      // Enable hot module replacement
      hot: true,
      // Or, with options (see https://github.com/rixo/svelte-hmr#options):
      hot: {
        preserveLocalState: true
      }
    }),
    // see NOTICE below
    resolve({ browser: true }),
    // ...
  ]
}
```

> **NOTICE:** You will need additional Rollup plugins. <br>Alone, this plugin translates Svelte components into CSS and JavaScript files. <br>You will need to include [`@rollup/plugin-node-resolve`](https://www.npmjs.com/package/@rollup/plugin-node-resolve) – and probably [`@rollup/plugin-commonjs`](https://www.npmjs.com/package/@rollup/plugin-commonjs) – in your Rollup config.


## Preprocessing and dependencies

If you are using the `preprocess` feature, then your callback responses may — in addition to the `code` and `map` values described in the Svelte compile docs — also optionally include a `dependencies` array. This should be the paths of additional files that the preprocessor result in some way depends upon. In Rollup 0.61+ in watch mode, any changes to these additional files will also trigger re-builds.


## `pkg.svelte`

If you're importing a component from your node_modules folder, and that component's package.json has a `"svelte"` property...

```js
{
  "name": "some-component",

  // this means 'some-component' resolves to 'some-component/src/SomeComponent.svelte'
  "svelte": "src/MyComponent.svelte"
}
```

...then this plugin will ensure that your app imports the *uncompiled* component source code. That will result in a smaller, faster app (because code is deduplicated, and shared functions get optimized quicker), and makes it less likely that you'll run into bugs caused by your app using a different version of Svelte to the component.

Conversely, if you're *publishing* a component to npm, you should ship the uncompiled source (together with the compiled distributable, for people who aren't using Svelte elsewhere in their app) and include the `"svelte"` property in your package.json.

If you are publishing a package containing multiple components, you can create an `index.js` file that re-exports all the components, like this:

```js
export { default as Component1 } from './Component1.svelte';
export { default as Component2 } from './Component2.svelte';
```

and so on. Then, in `package.json`, set the `svelte` property to point to this `index.js` file.


## Extracting CSS

By default (when `emitCss: true`) the CSS styles will be emitted into a virtual file, allowing another Rollup plugin – for example, [`rollup-plugin-css-only`](https://www.npmjs.com/package/rollup-plugin-css-only), [`rollup-plugin-postcss`](https://www.npmjs.com/package/rollup-plugin-postcss), etc. – to take responsibility for the new stylesheet. In fact, emitting CSS files _requires_ that you use a Rollup plugin to handle the CSS. Otherwise, your build(s) will fail! This is because this plugin will add an `import` statement to import the emitted CSS file. It's not valid JS to import a CSS file into a JS file, but it allows the CSS to be linked to its respective JS file and is a common pattern that other Rollup CSS plugins know how to handle.

If you set `emitCss: false` and your Svelte components contain `<style>` tags, the compiler will add JavaScript that injects those styles into the page when the component is rendered. That's not the default, because it adds weight to your JavaScript, prevents styles from being fetched in parallel with your code, and can even cause CSP violations.


## Hot module replacement

This plugin currently includes HMR support via community supported [svelte-hmr](https://github.com/rixo/svelte-hmr). This will be replaced by the official implementation once it is supported by Svelte's compiler itself.

What this option does is add `esm-hmr` compatible HMR handlers (see [there](https://github.com/rixo/svelte-hmr#whats-hmr-by-the-way) for an in-depth explanation) to the compiled components code.

By itself, this won't give you HMR out of the box with Rollup, because Rollup doesn't come with a dev server. See bellow for some HMR solutions with Rollup, or usage in HMR-enabled Vite.

Note that the `dev` compiler option is required for HMR. **If `compilerOptions.dev` is not set or falsy, the `hot` option will be disabled too.** (Which is probably what you want for a production build.)

### Options

For now (while using the community implementation), you can pass any of [svelte-hmr's options](https://github.com/rixo/svelte-hmr#options) to the `hot` option of this plugin.

For example:

```js
svelte({
  compilerOptions: {
    dev: true,
  },
  hot: {
    preserveLocalState: true,
  }
})
```

### HMR with Rollup

Rollup doesn't support HMR natively, because it doesn't even come with a dev server. If you want to enable HMR in a Rollup project, you'll need a HMR plugin like [rollup-plugin-hot](https://github.com/rixo/rollup-plugin-hot).

Or you can use [Nollup](https://github.com/PepsRyuu/nollup), a Rollup compatible dev server intended for faster rebuild during development (note that you'd also need a `esm-hmr` compatibility plugin with Nollup, like [this](https://github.com/rixo/rollup-plugin-hot-nollup)).

### HMR with Vite

This plugin can also be used directly with Vite 2, that supports a subset of Rollup plugin API, and has native HMR.

Example of the minimal config you need to enable Svelte HMR in Vite 2:

`vite.config.js`

```js
import svelte from 'rollup-plugin-svelte'

export default {
  plugins: [
    svelte({
      compilerOptions: {
        dev: true, // should be disabled for production build
      },
      hot: true,
      emitCss: false,
    }),
  ],
}
```


## License

MIT

# inkdrop-electron-link

inkdrop-electron-link is a fork of [electron-link](https://github.com/atom/electron-link) which is a node module that takes a JavaScript file (typically the entry point of an application) and a list of modules that need to be required lazily (see [Atom's build scripts](https://github.com/atom/atom/blob/d9ebd7e125d5f07def1a057a0a8278d4d9d7d23a/script/lib/generate-startup-snapshot.js#L19-L65) for an example). Then, starting from that file, it traverses the entire require graph and replaces all the forbidden `require` calls in each file with a function that will be called at runtime. The output is a single script containing the code for all the modules reachable from the entry point. This file can be then supplied to `mksnapshot` to generate a snapshot blob.

What's added:

- `navigator` global object in blueprint (See [the commitlog](https://github.com/inkdropapp/inkdrop-electron-link/commit/d92dc6c68f9d5fb35147eef5476f3737da502057))
- some additional properties for `HTMLElement` (See [the commitlog](https://github.com/inkdropapp/inkdrop-electron-link/commit/d92dc6c68f9d5fb35147eef5476f3737da502057))
- `transformSource` option for custom source transformations (See [the commitlog](https://github.com/inkdropapp/inkdrop-electron-link/commit/5ebff9507dce73039f46238c4d3889acf84db8c2))

It can also determine whether a module can be snapshotted or not. For instance, the following code can be snapshotted:

```js
const path = require("path");

module.exports = function () {
  return path.join("a", "b", "c");
};
```

And generates the following code:

```js
let path;
function get_path() {
  return (path || path = require("path"));
}

module.exports = function () {
  return get_path().join("a", "b", "c");
};
```

You can notice that the above code is valid because the forbidden module (i.e. `path`) is used inside a function that doesn't get called when requiring the script. On the other hand, when trying to process the following code, electron-link will throw an error because it is trying to access a forbidden module right when it gets required:

```js
const path = require("path");

module.exports = path.join("a", "b", "c");
```

Being a tool based on static analysis, however, electron-link is unable to detect all the cases where a piece of code can't be included in a snapshot. Therefore, we recommend running the generated JavaScript file in an empty V8 context (similar to the one provided by `mksnapshot`) to catch any invalid code that might have slipped through.

## Installation

```bash
npm install --save electron-link
```

## Usage

```js
const electronLink = require("electron-link");

const snapshotScript = await electronLink({
  baseDirPath: "/base/dir/path",
  mainPath: "/base/dir/path/main.js",
  cachePath: "/cache/path",
  shouldExcludeModule: (modulePath) => excludedModules.has(modulePath),
  transformSource: (source, filePath) => {
    // Transform source code if necessary
    return source;
  },
});

const snapshotScriptPath = "/path/to/snapshot/script.js";
fs.writeFileSync(snapshotScriptPath, snapshotScript);

// Verify if we will be able to use this in `mksnapshot`
vm.runInNewContext(snapshotScript, undefined, {
  filename: snapshotScriptPath,
  displayErrors: true,
});
```

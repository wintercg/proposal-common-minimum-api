# Helpers on `import.meta` for working with paths

## Background

A common user request from developers writing ESM code is easy access to file path information of the current module. Or put another way, they miss the `__dirname` and `__filename` local variables from CommonJS, that define the directory and full path of the current module. These values are distinct from `import.meta.url`, which is the URL of the current module. To illustrate:

- `import.meta.url` is a string like `file:///Users/Geoffrey/my-project/src/index.js`
- `__filename` is a string like `/Users/Geoffrey/my-project/src/index.js`
- `__dirname` is a string like `/Users/Geoffrey/my-project/src`

Since `__filename` and `__dirname` have been around [since Node.js 0.1.27](https://nodejs.org/api/modules.html#__dirname), released in 2010, they are very well known to Node.js developers and very broadly referenced by `npm` packages. It is common to find documentation suggesting passing `__dirname` into a function for serving the static assets of a folder, for example, or passing `__filename` into a function used for logging.

In Node.js, developers working with such path-based APIs need to jump through a few hoops. Node.js provides a helper `fileURLToPath` that developers can import from `node:url`, to enable code like `serveStaticAssets(dirname(fileURLToPath(import.meta.url)))`. A common complaint of Node.js developers is that this is too verbose, and a way to achieve the same with simpler code and without needing to import utility helpers would be preferable.

## Proposal

For runtimes that support filesystem-based modules, we propose adding two new optional properties to `import.meta`:

### `import.meta.filename`

A string containing the full absolute filesystem path to the current module, like CommonJS `__filename`. This is the same as `fileURLToPath(import.meta.url)` in Node.js. An example value would be `/Users/Geoffrey/my-project/src/index.js`.

The property will be absent for modules that are not loaded from the filesystem, such as modules whose `import.meta.url` begins with `http` or `data`.

### `import.meta.dirname`

A string containing the full absolute filesystem path of the folder containing the current module, like CommonJS `__dirname`. This is the same as `dirname(fileURLToPath(import.meta.url))` in Node.js. An example value would be `/Users/Geoffrey/my-project/src`.

The value should not have a trailing slash. This is consistent with CommonJS `__dirname`, so that it can be used as a drop-in replacement.

The value will be undefined for modules that are not loaded from the filesystem, such as modules whose `import.meta.url` begins with `http` or `data`.

## Notes

These properties would be optional, even if a runtime supports filesystem-based modules. This allows for runtimes where modules are _primarily_ run from HTTP / HTTPS URLs to avoid creating a portability hazard where these properties exist while developers are working locally but are undefined when the code is deployed.

This portability hazard is a primary objection to this proposal. We feel that on balance, the convenience of these properties outweighs the potential hazard; and that a different portability hazard exists without these helpers, in that developers will write code such as `fileURLToPath(import.meta.url)` that works locally but not over CDN similarly to how `import.meta.filename` would work locally but not over CDN.

## Alternatives Considered

### `import.meta.__filename` or `import.meta.__dirname`

As the primary goal of this proposal is convenience, we feel that shorter names are preferable; and the `__` prefix is inconsistent with other `import.meta` properties.

### `import.meta.path` or `import.meta.dir`

Bun has shipped `import.meta.path`, an equivalent to the `import.meta.filename` proposed here; and `import.meta.dir`, an equivalent to the `import.meta.dirname` proposed here.

On the one hand, Bun’s names are shorter and `import.meta.path` is arguably more correct of a name for a value that refers to a full absolute path like `/Users/Geoffrey/my-project/src/index.js` rather than just the filename `index.js`.

On the other hand, `__filename` and `__dirname` are _so_ well known by developers, being used by just about every Node.js developer who has written any CommonJS code over the past decade-plus, that preserving those names might aid in usability due to familiarity and consistency with documentation of existing ecosystem libraries.

We feel that on balance, the familiarity and legacy of `__filename` and `__dirname` outweighs the benefits of new names that are shorter and more technically correct.

### `import.meta.dirname` by itself, without `import.meta.filename`

There are many more use cases for `dirname` than for `filename`. It’s very common to want to reference files that are siblings to the current module; it’s less common to need to reference the current module itself. We also have `import.meta.url` as an URL-based reference to the current module, that can be converted to `import.meta.filename` easily enough.

However it feels incomplete to offer a helper only for the current module’s parent folder, and not also to the current module itself. This feels inconsistent with `import.meta.url`, and to the legacy of `__dirname` and `__filename`.

### Also `import.meta.file` or `import.meta.basename`

If we’re going to have `import.meta.dirname`, a helper for the file path of the containing folder, why not also a helper for the current module’s filename such as Bun’s `import.meta.file` (returning a string like `index.js`)? This would be equivalent to `basename(fileURLToPath(import.meta.url))`.

Along the lines of the previous section, it’s not common enough to refer to the current file _as a file path_ to warrant a helper on `import.meta`. The current module’s filename can easily be derived via helper utilities.

## Prior Art

- [Node.js `__dirname` and `__filename` documentation](https://nodejs.org/api/modules.html#__dirname)
- [Bun `import.meta.path` and `import.meta.dir`](https://bun.sh/docs/api/import-meta)
- [Node.js PR to add `import.meta.dirname` and `import.meta.filename`](https://github.com/nodejs/node/pull/48740)
- [Earlier Node.js PR to add `import.meta.__dirname` and `import.meta.__filename`](https://github.com/nodejs/node/pull/39147)

## References

- [WinterCG issue](https://github.com/wintercg/proposal-common-minimum-api/issues/50)

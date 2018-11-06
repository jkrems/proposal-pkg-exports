# Module Resolution & Format Lookup

## Motivating Examples

* A package (`react-dom`) has a dedicated entrypoint `react-dom/server for code that isn't compatible with a browser environment.
* A package (`angular`) exposes multiple independent APIs, modeled via import paths like `angular/common/http`.
* A package (`lodash`) allows to import individual functions, e.g. `lodash/map`.
* A package is exclusively exposing an ESM interface.
* A package is exclusively exposing a CJS interface.
* A package is exposing both an ESM and a CJS interface.
* A package wishes to publish `.js` files containing ESM code.
* A project wants to mix both ESM and CJS code, with CJS running as part of the ESM module graph.

### High Level Considerations

* The baseline behavior of relative imports should match a browser's with a simple file server.
  This implies that `./x` will only ever import exactly the sibling file "x" without appending paths or extensions.
* The primary compatibility boundary are bare specifiers. Relative and absolute imports can follow simpler rules.

### `package.json` Interface

We propose a field in `package.json` to specify an ESM location when importing bare specifiers.
The key is TBD, the examples use "esm-import-map" as a placeholder.
Neither the name nor the fact that it exists top-level is final.

The `package.json` "esm-import-map" interface will only be respected for bare specifiers.

When using this property as a signifier of the package being an ESM package,
this `package.json` check would also be done in package boundary lookups to determine
if the package is ESM or legacy.

#### Single Mapping

For packages that only want to support `import 'pkg-name'`, the key can be set to a specifier.
The specifier will be resolved relative to the URL of `package.json`.
It may only be an absolute URL or a relative path ("./[...]", "../[...]").
Bare specifiers are not allowed as values for the mapping.
The `import:` URL scheme is also explicitly disallowed in the mapping.

```js
{
  "name": "@user/name",
  "esm-import-map": "./dist/index.mjs"
}
```

#### Multiple Mappings

"Deep imports" are explicit and expressed as an object.
When importing the package name directly, the default key will be used.
The semantics of the mapped value are the same as for the single mapping.

```js
{
  "name": "@user/name",
  "esm-import-map": {
    "default": "./dist/index.mjs",
    "foo": "./some/filename-here.mjs"
  }
}
```

For a consumer, this will mean the following, assuming the `package.json` file is located at `file:///path/to/pkg/package.json` and `@user/name` is mapped to that package:

```js
// Loads file:///path/to/pkg/dist/index.mjs
import x from '@user/name';
// Does the same as:
import x from 'file:///path/to/pkg/dist/index.mjs';

// Loads file:///path/to/pkg/some/filename-here.mjs
import x from '@user/name/foo';
// Does the same as:
import x from 'file:///path/to/pkg/some/filename-here.mjs';

// Fails - no such mapping (trailing slash not allowed).
import x from '@user/name/';

// Fails - directories cannot be imported, package.json only considered for bare imports
import x from 'file:///path/to/pkg';
// Fails - directories cannot be imported, package.json only considered for bare imports
import x from 'file:///path/to/pkg/';
// Fails - directories cannot be imported (there is no index.* magic)
import x from 'file:///path/to/pkg/dist/';

// Fails - no such mapping.
import x from '@user/name/dist/index.mjs';
```

### Extension #1: Format Hint for `.js`

For packages that support both ESM and CJS,
the import map can be preferred on `import` while `require` would hit the existing main entrypoint.
The fact that we have a strong signal for the preference of the module author,
we can use the same signal to interpret a `.js` file as ESM if it is used in the import mapping.

So the following would work:

```
{
  // dist/index.js never hits the ESM content type logic and is interpreted as CJS by require.extensions:
  "main": "dist/index.js",
  // src/index.js is mapped to the ESM content type because it is loaded via the import map:
  "esm-import-map": "src/index.js"
}
```

This raises the question how something might appear in an import map but be CJS, e.g. because a library hasn't fully migrated.
Also how subsequent imports from `src/index.js` would be interpreted.

### Extension #2: Content Type Override for CJS

One possible way of making a purely `.js`-based package easier to write,
is to flip defaults.
The content type lookup used when importing a `.js` file from disk would treat it as a module.
A custom lookup can be used to allow importing `.js` files and interpret them as CJS.

Example (strawman):

```js
// In package.json or via a node command line flag:
{
  "esm-content-types": {
    // node --esm-content-type=./old-src/**/*.js:application/vnd.node.js
    "./old-src/**/*.js": "application/vnd.node.js"
  }
}
```

No matter how a file is loaded, these overrides will be applied when loading the resource from a URL matching this filter.
Code that does not want through this mapping can use `createRequireFunction`.

## Open Questions

### Should there be a way to allow deep imports?

Possible solutions include:

* It's always allowed and tried if an explicit mapping isn't found.
* It's configured via a special key in the import map (e.g. `"/": "./lib/"`).
* It's configured via a dedicated flag (`"esm-allow-deep-import": true`).

### Should explict `/default` be allowed?

```js
import x from 'some-pkg';
// Does this work and do the same thing?
import x from 'some-pkg/default';
```

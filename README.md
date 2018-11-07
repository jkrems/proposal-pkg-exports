# Bare Module Specifier Resolution in node.js

## Motivating Examples

* A package (`react-dom`) has a dedicated entrypoint `react-dom/server` for code that isn't compatible with a browser environment.
* A package (`angular`) exposes multiple independent APIs, modeled via import paths like `angular/common/http`.
* A package (`lodash`) allows to import individual functions, e.g. `lodash/map`.
* A package is exclusively exposing an ESM interface.
* A package is exclusively exposing a CJS interface.
* A package is exposing both an ESM and a CJS interface.
* A project wants to mix both ESM and CJS code, with CJS running as part of the ESM module graph.
* A package wants to expose multiple entrypoints as its public API without leaking internal directory structure.

## High Level Considerations

* The baseline behavior of relative imports should match a browser's with a simple file server.
  This implies that `./x` will only ever import exactly the sibling file "x" without appending paths or extensions.
  `"x"` is never resolved to `x.mjs` or `x/index.mjs` (or the `.js` equivalents).
* The primary compatibility boundary are bare specifiers. Relative and absolute imports can follow simpler rules.
* Resolution should not depend on file extensions, leaving open the potential for supporting ESM in `.js` files.
* The directory structure of a module should be treated as private implementation detail.

## `package.json` Interface

We propose a field in `package.json` to specify an ESM entrypoint location when importing bare specifiers.

> **The key is TBD, the examples use `"exports"` as a placeholder.**
> **Neither the name nor the fact that it exists top-level is final.**

The `package.json` `"exports"` interface will only be respected for bare specifiers, e.g. `import _ from 'lodash'` where the specifier `'lodash'` doesn’t start with a `.` or `/`.

The existence of this `"exports"` key in `package.json` signifies that the module should be imported as ESM by Node. The module may *also* have a CommonJS export, the `"main"` field, for consumers that use `require` such as older versions of Node.

Looking forward to future work around format disambiguation, such as the [`"mimes"` field proposal](https://github.com/nodejs/modules/pull/160), the check for `"exports"` in `package.json` as a signifier of ESM mode would also be done in package boundary lookups to determine if the package is ESM or legacy.

### Example

Here’s a complete `package.json` example, for a hypothetical module named `@momentjs/moment`:

```js
{
  "name": "@momentjs/moment",
  "version": "0.0.0",
  "main": "./dist/index.js",
  "exports": {
    "": "./src/moment.mjs",
    "/": "./src/util/",
    "/timezones/": "./data/timezones/",
    "/timezones/utc": "./data/timezones/utc/index.mjs"
  }
}
```

Within the `"exports"` object, the keys are concatenated on the end of the name field, e.g. `import utc from '@momentjs/moment/timezones/utc'` is formed from `'@momentjs/moment'` + `'/timezones/utc'`.

The main entrypoint is therefore the empty string, `"": "./src/moment.mjs"`. For modules that desire to export *only* a single entrypoint, e.g. `import request from 'request'`, the `"exports"` key itself can be set to the entrypoint:

```js
{
  "name": "request",
  "version": "0.0.0",
  "exports": "./request.mjs"
}
```

Keys that end in slashes can map to folder roots, following the [pattern in the import maps proposal](https://github.com/domenic/import-maps#packages-via-trailing-slashes): `"/timezones/": "./data/timezones/"` would allow `import pdt from "@momentjs/moment/timezones/pdt.mjs"` to import `./data/timezones/pdt.mjs`.

- Using `"/"` maps to the root, so `"/": "./src/util/"` would allow `import tick from "@momentjs/moment/tick.mjs"` to import `/src/util/tick.mjs`.

- Mapping a key of `"/"` to a value of `"./"` exposes all files in the package, where `"/": "./"` would allow `import privateHelpers from "@momentjs/moment/private-helpers.mjs"` to import `./private-helpers.mjs`.

- When mapping to a folder root, both the left and right sides must end in slashes: `"./": "./dist/"`, not `".": "./dist"`.

- Unlike in CommonJS, there is no automatic searching for `index.js` or `index.mjs`.

The value of an export, e.g. `"./src/moment.mjs"`, must begin with `.` to signify a relative path (e.g. "./src" is okay, but `"/src"` or `"src"` are not). This is to reserve potential future use for `"exports"` to export things referenced via specifiers that aren’t relatively-resolved files, such as other packages or other protocols.

There is the potential for collisions in the exports, such as `"/timezones/"` and `"/timezones/utc"` in the example above (e.g. if there’s a file named `utc` in the `./data/timezones` folder).
Rough outline of a possible resolution algorithm:

1. Find the package matching the base specifier, e.g. `@momentjs/moment` or `request`.
1. Load its exports map.
1. If there is an exact match for the requested specifier, return the resolution.
1. Otherwise, find the longest matching path prefix. If there is a path prefix, return the resolution by applying the prefix.
1. Return an error - no mapping found.

In the future, the algorithm might be adjusted to align with work done in the [import maps proposal](https://github.com/domenic/import-maps).

### Usage

For a consumer, the above `@momentjs/moment` and `request` packages can be used as follows, assuming the user’s project is in `/app` with `/app/package.json` and `/app/node_modules`:

```js
import request from 'request';
// Loads file:///app/node_modules/request/request.mjs

import request from './node_modules/request/request.mjs';
// Loads file:///app/node_modules/request/request.mjs

import request from 'file:///app/node_modules/request/request.mjs';
// Loads file:///app/node_modules/request/request.mjs

import utc from '@momentjs/moment/timezones/utc';
// Loads file:///app/node_modules/@momentjs/moment/timezones/utc/index.mjs
```

The following don’t work - please note that **error messages and codes are TBD**:

```js
import request from 'request/';
// Error: trailing slash not mapped

import request from 'request/request.mjs';
// Error: no such mapping

import moment from '@momentjs/moment/';
// Error: trailing slash not allowed (cannot import folders, only files)

import request from 'file:///app/node_modules/request';
// Error: folders cannot be imported, package.json only considered for bare imports

import request from 'file:///app/node_modules/request/';
// Error: folders cannot be imported, package.json only considered for bare imports

import utc from '@momentjs/moment/timezones/utc/'; // Note trailing slash
// Error: folders cannot be imported (there is no index.* magic)
```

### Prior Art

* [`package.json#browser`](https://github.com/defunctzombie/package-browser-field-spec)
* [Import Maps](https://github.com/domenic/import-maps)
* [`package.json#mimes`](https://github.com/nodejs/modules/pull/160)

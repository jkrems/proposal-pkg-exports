# Bare Module Specifier Resolution in node.js

**Contributors:** Guy Bedford, Geoffrey Booth, Jan Krems, Saleh Abdel Motaal

## Motivating Examples

* A package (`react-dom`) has a dedicated entrypoint `react-dom/server` for code that isn't compatible with a browser environment.
* A package (`angular`) exposes multiple independent APIs, modeled via import paths like `angular/common/http`.
* A package (`lodash`) allows to import individual functions, e.g. `lodash/map`.
* A package is exclusively exposing an ESM interface.
* A package is exclusively exposing a CJS interface.
* A package is exposing both an ESM and a CJS interface.
* A project wants to mix both ESM and CJS code, with CJS running as part of the ESM module graph.
* A package wants to expose multiple entrypoints as its public API without leaking internal directory structure.
* A package wants to reference an internally aliased subpath, without exposing it publicly.

## High Level Considerations

* The baseline behavior of relative imports should match a browser's with a simple file server.
  This implies that `./x` will only ever import exactly the sibling file "x" without appending paths or extensions.
  `"x"` is never resolved to `x.mjs` or `x/index.mjs` (or the `.js` equivalents).
* The primary compatibility boundary are bare specifiers. Relative and absolute imports can follow simpler rules.
* Resolution should not depend on file extensions, allowing ESM syntax in `.js` files.
* The directory structure of a module should be treated as private implementation detail.

## `package.json` Interfaces

We propose two fields in `package.json` to specify entrypoints and internal aliasing of bare specifiers - [`"exports"`](#1-exports-field) and [`"imports"`](#2-imports-field).

> **For both fields the final names of `"exports"` and `"imports"` are still TBD, and these names should be considered placeholders.**

These interfaces will only be respected for bare specifiers, e.g. `import _ from 'lodash'` where the specifier `'lodash'` doesn’t start with a `.` or `/`.

Package _exports_ and _imports_ can be supported fully independently, and in both CommonJS and ES modules.

### 1. Exports Field

#### Example

Here’s a complete `package.json` example, for a hypothetical module named `@momentjs/moment`:

```js
{
  "name": "@momentjs/moment",
  "version": "0.0.0",
  "type": "module",
  "main": "./dist/index.js",
  "exports": {
    "./": "./src/util/",
    "./timezones/": "./data/timezones/",
    "./timezones/utc": "./data/timezones/utc/index.mjs"
  }
}
```

Within the `"exports"` object, the key string after the `'.'` is concatenated on the end of the name field, e.g. `import utc from '@momentjs/moment/timezones/utc'` is formed from `'@momentjs/moment'` + `'/timezones/utc'`. Note that this is string manipulation, not a file path: `"./timezones/utc"` is allowed, but just `"timezones/utc"` is not.

Keys that end in slashes can map to folder roots, following the [pattern in the browser import maps proposal](https://github.com/WICG/import-maps#packages-via-trailing-slashes): `"./timezones/": "./data/timezones/"` would allow `import pdt from "@momentjs/moment/timezones/pdt.mjs"` to import `./data/timezones/pdt.mjs`.

- Using `"./"` maps the root, so `"./": "./src/util/"` would allow `import tick from "@momentjs/moment/tick.mjs"` to import `./src/util/tick.mjs`.

- Mapping a key of `"./"` to a value of `"./"` exposes all files in the package, where `"./": "./"` would allow `import privateHelpers from "@momentjs/moment/private-helpers.mjs"` to import `./private-helpers.mjs`.

- When mapping to a folder root, both the left and right sides must end in slashes: `"./": "./dist/"`, not `".": "./dist"`.

- Unlike in CommonJS, there is no automatic searching for `index.js` or `index.mjs` or for file extensions. This matches the [behavior of the import maps proposal](https://github.com/WICG/import-maps#packages-via-trailing-slashes).

The value of an export, e.g. `"./src/moment.mjs"`, must begin with `.` to signify a relative path (e.g. "./src" is okay, but `"/src"` or `"src"` are not). This is to reserve potential future use for `"exports"` to export things referenced via specifiers that aren’t relatively-resolved files, such as other packages or other protocols.

There is the potential for collisions in the exports, such as `"./timezones/"` and `"./timezones/utc"` in the example above (e.g. if there’s a file named `utc` in the `./data/timezones` folder).
Rough outline of a possible resolution algorithm:

1. Find the package matching the base specifier, e.g. `@momentjs/moment` or `request`.
1. Load its exports map.
1. If there is an exact match for the requested specifier, return the resolution.
1. Otherwise, find the longest matching path prefix. If there is a path prefix, return the resolution by applying the prefix.
1. Return an error - no mapping found.

In the future, the algorithm might be adjusted to align with work done in the [import maps proposal](https://github.com/domenic/import-maps).

For packages that only have a main and no exports, `"exports": false` can be used as a shorthand for `"exports": {}` providing an encapsulated package.

#### Usage

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

### 2. Imports Field

Imports provide the ability to remap bare specifiers within packages before they hit the node_modules resolution process.

The current proposal prefixes all imports with `#` to provide a clear signal that it's a _symbolic specifier_ and also to prevent packages that use imports from working in any environment (runtime, bundler) that isn't aware of imports.

> **Whether this restriction is maintained in the final proposal, or what exact symbol is used for `#` is still TBD.**

#### Example

For the same example package as provided for `"exports"`, consider if we wanted to make the `timezones` implementation something that is only referenced internally by code within `@momentjs/moment`, instead of exposing it to external importers.

```js
{
  "name": "@momentjs/moment",
  "version": "0.0.0",
  "type": "module",
  "main": "./dist/index.js",
  "imports": {
    "#timezones/": "./data/timezones/",
    "#timezones/utc": "./data/timezones/utc/index.mjs",
    "#external-feature": "external-pkg/feature",
    "#moment/": "./"
  }
}
```

As with package exports, mappings are mapped relative to the package base, and keys that end in slashes can map to folder roots.

The resolution algorithms remain the same except `"imports"` provide the added feature that they can also map into third-party packages that would be looked up in node_modules, including to subpaths that would be in turn resolved through `"exports"`. There is no risk of circular resolution here, since `"exports"` themselves only ever resolve to direct internal paths and can't in turn map to aliases.

The `"imports"` that apply within a given file are determined based on looking up the package boundary of that file.

#### Usage

For the author of `@momentjs/moment`, they can use these aliases with the following code in any file in `@momentjs/moment/*.js`, provided it matches the package boundary of the `package.json` file:

```js
import utc from '#timezones/utc';
// Loads file:///app/node_modules/@momentjs/moment/data/timezones/utc/index.mjs

import utc from './data/timezones/utc/index.mjs';
// Loads file:///app/node_modules/@momentjs/moment/data/timezones/utc/index.mjs

import utc from '#timezones/utc/index.mjs';
// Loads file:///app/node_modules/@momentjs/moment/data/timezones/utc/index.mjs

import utc from '#moment/data/timezones/utc/index.mjs';
// Loads file:///app/node_modules/@momentjs/moment/data/timezones/utc/index.mjs
```

The following don’t work - please note that **error messages and codes are TBD**:

```js
import utc from '#timezones/utc/';
// Error: trailing slash not mapped

import unknown from '#unknown';
// Error: no such import alias

import timezones from '#timezones/';
// Error: trailing slash not allowed (cannot import folders, only files)

import utc from '#moment';
// Error: no mapping provided (folder mappings require subpaths)
```

### Prior Art

* [`package.json#browser`](https://github.com/defunctzombie/package-browser-field-spec)
* [Import Maps](https://github.com/domenic/import-maps)
* [`package.json#mimes`](https://github.com/nodejs/modules/pull/160)
* [node.js ESM resolver spec](https://github.com/nodejs/ecmascript-modules/pull/12)

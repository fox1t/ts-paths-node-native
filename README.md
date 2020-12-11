# ts-paths-node-native

This repository shows how to use TypeScript paths with Node.js without any additional bundling/transformation step.

## Usage

```bash
$ git clone git@github.com:fox1t/ts-paths-node-native.git
$ npm i
$ npm run build
$ npm start
```

### Note

If you use a Node.js version that doesn't support the subpath-patterns feature, you will get this error:

```bash
internal/modules/cjs/loader.js:969
  throw err;
  ^

Error: Cannot find module '#subpath/index'
```

## Background

Before version v12.20.0 and v14.13.0, in order to use [TS path-mapping](https://www.typescriptlang.org/docs/handbook/module-resolution.html#path-mapping) in Node.js, a bundling/transformation step was mandatory. In fact, TypeScript itself doesn't "convert" the mapped paths to the real paths. That's it if you write:

```ts
import { foo } from "#subpath/foo";
```

it will be compiled to

```js
const foo_1 = require("#subpath/foo");
```

Possible ways of making it work were to use babel to transform the code during the build or add a dependency at runtime using the `-r` flag. Neither of two was an ideal one:

- Using another tool in conjunction with `tsc` adds complexity to the configuration: in this case, `tsc` was used just to check types and the compilation was usually [handled by babel](https://www.typescriptlang.org/docs/handbook/babel-with-typescript.html)
- Adding a runtime dependency has two significant drawbacks:
  - slower startup time of the process;
  - there are still no "mature" projects that work in every scenario.

## Solution

Version 12.20.0 and 14.13.0 Node.js supports native path remapping during require/import thanks to two features:

- [Subpath imports](https://nodejs.org/api/packages.html#packages_subpath_imports)
- [Subpath patterns](https://nodejs.org/api/packages.html#packages_subpath_patterns)

Adding these lines to the package.json file

```json
"imports": {
    "#subpath/*": {
      "require": "./dist/subpath/*.js"
    }
  },
```

makes Node.js understand what to do when it finds `const foo_1 = require("#subpath/foo");` in the code.
On the tsconfig.json side, things remain the same as before.

```json
{
  "compilerOptions": {
    "outDir": "dist",
    "baseUrl": "./src",
    "paths": {
      "#subpath/*": ["subpath/*"]
    }
  }
}
```

## Limitations

### Always use `#` to specify an import subpath

**Entries in the imports field must always start with `#`** to ensure they are disambiguated from package specifiers. So no more `@`.

### Use `index` file when importing folders
Since Node.js' `subpath imports` just add `.js` extension to the path you provide, you need to use the `index` file when importing a folder.
That's it, 
```ts
import * as config from "#subpath/module/index";
```

instead of
```ts
import * as config from "#subpath/module";
```

In fact, the latest one will be resloved as 
```js
`const module = require("./dist/module.js")`
```
instead of 
```js
`const module = require("./dist/module/index.js")`
```
and Node.js will not find the file to import at runtime.

### No JSON files import

Today, importing .json files in TypeScript using the flag `resolveJsonModule` isn't supported since Node.js matches files without extension. So writing

```ts
import * as config from "#subpath/config.json";
```

Will resolve to this path.

```bash
./src/subpath/config.json.js
```

**Use the full path instead of the alias.**

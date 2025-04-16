# typescript-json-schema

[![npm version](https://img.shields.io/npm/v/typescript-json-schema.svg)](https://www.npmjs.com/package/typescript-json-schema) ![Test](https://github.com/YousefED/typescript-json-schema/workflows/Test/badge.svg)

* == library /
  * lightweight
  * | maintenance mode
  * -- compatible with -- recent Typescript versions
  * -- use -- 
    * Typescript compiler internally
    * type hierarchy
* allows
  * from your Typescript sources -- generate -- json-schemas  
* ALTERNATIVES
  * [ts-json-schema-generator](https://github.com/vega/ts-json-schema-generator)

## Features

- your Typescript program -- is compiled to get -- complete type information
- required properties, extends, annotation keywords, property initializers -- are translated as -- defaults
  - _Examples:_
    - [api](api.md)
    - [test examples](test/programs)

## how to use?

### -- via -- CL

- `npm install typescript-json-schema -g`
- `typescript-json-schema [options] <path-to-typescript-files-or-tsconfig> <type>`
  - from a typescript type -- generate a -- schema 
  - `<path-to-typescript-files-or-tsconfig>`
    - recommendations
      - use `tsconfig.json`
    - if you specify DIRECTLY .ts files -> use some built-in compiler presets
  - `<type>`
    - ALLOWED
      - 1! FULLY qualified type
      - `"*"` == ALL types
  - options
    - --refs                Create shared ref definitions.                               [boolean] [default: true]
    - `--aliasRefs`
      - Create SHARED ref definitions / type aliases
      - `boolean`
        - by default, `false`
    - --topRef              Create a top-level ref definition.                           [boolean] [default: false]
    - --titles              Creates titles in the output schema.                         [boolean] [default: false]
    - --defaultProps        Create default properties definitions.                       [boolean] [default: false]
    - `--noExtraProps`
      - disable objects' ADDITIONAL properties 
      - `boolean`
        - by default, `false`
    - `--propOrder`
      - create property order definitions
      - `boolean`
        - by default, `false`
    - `--required`
      - create required [] / NON-OPTIONAL properties
      - `boolean`
        - by default, `false`
    - --strictNullChecks    Make values non-nullable by default.                         [boolean] [default: false]
    - --esModuleInterop     Use esModuleInterop when loading typescript modules.         [boolean] [default: false]
    - --skipLibCheck        Use skipLibCheck when loading typescript modules.            [boolean] [default: false]
    - --useTypeOfKeyword    Use `typeOf` keyword (https://goo.gl/DC6sni) for functions.  [boolean] [default: false]
    - `--out, -o`
      - output file
        - by default, stdout
    - --validationKeywords  Provide additional validation keywords to include            [array]   [default: []]
    - `--include [array]`
      - restrict types / generate
      - by default, `[]`
      - use cases
        - large projects
    - --ignoreErrors        Generate even if the program has errors.                     [boolean] [default: false]
    - --excludePrivate      Exclude private members from the schema                      [boolean] [default: false]
    - --uniqueNames         Use unique names for type symbols.                           [boolean] [default: false]
    - --rejectDateType      Rejects Date fields in type definitions.                     [boolean] [default: false]
    - --id                  Set schema id.                                               [string]  [default: ""]
    - --defaultNumberType   Default number type.                                         [choices: "number", "integer"] [default: "number"]
    - --tsNodeRegister      Use ts-node/register (needed for require typescript files).  [boolean] [default: false]
    - --constAsEnum         Use enums with a single value when declaring constants.      [boolean] [default: false]
    - --experimentalDecorators  Use experimentalDecorators when loading typescript modules [boolean] [default: true]
  - _Example:_ `typescript-json-schema "project/directory/**/*.ts" TYPE`

### -- via -- Programmatic use

```ts
import { resolve } from "path";

import * as TJS from "typescript-json-schema";

// optionally pass argument to schema generator
const settings: TJS.PartialArgs = {
    required: true,
};

// optionally pass ts compiler options
const compilerOptions: TJS.CompilerOptions = {
    strictNullChecks: true,
};

// optionally pass a base path
const basePath = "./my-dir";

const program = TJS.getProgramFromFiles(
  [resolve("my-file.ts")],
  compilerOptions,
  basePath
);

// We can either get the schema for one file and one type...
const schema = TJS.generateSchema(program, "MyType", settings);

// ... or a generator that lets us incrementally get more schemas

const generator = TJS.buildGenerator(program, settings);

// generator can be also reused to speed up generating the schema if usecase allows:
const schemaWithReusedGenerator = TJS.generateSchema(program, "MyType", settings, [], generator);

// all symbols
const symbols = generator.getUserSymbols();

// Get symbols for different types from generator.
generator.getSchemaForSymbol("MyType");
generator.getSchemaForSymbol("AnotherType");
```

```ts
// In larger projects type names may not be unique,
// while unique names may be enabled.
const settings: TJS.PartialArgs = {
    uniqueNames: true,
};

const generator = TJS.buildGenerator(program, settings);

// A list of all types of a given name can then be retrieved.
const symbolList = generator.getSymbols("MyType");

// Choose the appropriate type, and continue with the symbol's unique name.
generator.getSchemaForSymbol(symbolList[1].name);

// Also it is possible to get a list of all symbols.
const fullSymbolList = generator.getSymbols();
```

* `getSymbols('<SymbolName>')` & `getSymbols()`
  * return `[SymbolRef]` /
    ```ts
    type SymbolRef = {
        name: string;
        typeName: string;
        fullyQualifiedName: string;
        symbol: ts.Symbol;
    };
    ```

* `getUserSymbols` & `getMainFileSymbols`
  * return `[string]`

### -- via --Annotations

* TODO:
The schema generator converts annotations to JSON schema properties.

For example

```ts
export interface Shape {
    /**
     * The size of the shape.
     *
     * @minimum 0
     * @TJS-type integer
     */
    size: number;
}
```

will be translated to

```json
{
    "$ref": "#/definitions/Shape",
    "$schema": "http://json-schema.org/draft-07/schema#",
    "definitions": {
        "Shape": {
            "properties": {
                "size": {
                    "description": "The size of the shape.",
                    "minimum": 0,
                    "type": "integer"
                }
            },
            "type": "object"
        }
    }
}
```

Note that we needed to use `@TJS-type` instead of just `@type` because of an [issue with the typescript compiler](https://github.com/Microsoft/TypeScript/issues/13498).

You can also override the type of array items, either listing each field in its own annotation or one
annotation with the full JSON of the spec (for special cases). This replaces the item types that would
have been inferred from the TypeScript type of the array elements.

Example:

```ts
export interface ShapesData {
    /**
     * Specify individual fields in items.
     *
     * @items.type integer
     * @items.minimum 0
     */
    sizes: number[];

    /**
     * Or specify a JSON spec:
     *
     * @items {"type":"string","format":"email"}
     */
    emails: string[];
}
```

Translation:

```json
{
    "$ref": "#/definitions/ShapesData",
    "$schema": "http://json-schema.org/draft-07/schema#",
    "definitions": {
        "Shape": {
            "properties": {
                "sizes": {
                    "description": "Specify individual fields in items.",
                    "items": {
                        "minimum": 0,
                        "type": "integer"
                    },
                    "type": "array"
                },
                "emails": {
                    "description": "Or specify a JSON spec:",
                    "items": {
                        "format": "email",
                        "type": "string"
                    },
                    "type": "array"
                }
            },
            "type": "object"
        }
    }
}
```

This same syntax can be used for `contains` and `additionalProperties`.

### -- via -- `integer` type alias

If you create a type alias `integer` for `number` it will be mapped to the `integer` type in the generated JSON schema.

Example:

```typescript
type integer = number;
interface MyObject {
    n: integer;
}
```

Note: this feature doesn't work for generic types & array types, it mainly works in very simple cases.

### -- via -- `require` a variable from a file

(for requiring typescript files is needed to set argument `tsNodeRegister` to true)

When you want to import for example an object or an array into your property defined in annotation, you can use `require`.

Example:

```ts
export interface InnerData {
    age: number;
    name: string;
    free: boolean;
}

export interface UserData {
    /**
     * Specify required object
     *
     * @examples require("./example.ts").example
     */
    data: InnerData;
}
```

file `example.ts`

```ts
export const example: InnerData[] = [{
  age: 30,
  name: "Ben",
  free: false
}]
```

Translation:

```json
{
    "$schema": "http://json-schema.org/draft-07/schema#",
    "properties": {
        "data": {
            "description": "Specify required object",
            "examples": [
                {
                    "age": 30,
                    "name": "Ben",
                    "free": false
                }
            ],
            "type": "object",
            "properties": {
                "age": { "type": "number" },
                "name": { "type": "string" },
                "free": { "type": "boolean" }
            },
            "required": ["age", "free", "name"]
        }
    },
    "required": ["data"],
    "type": "object"
}
```

Also you can use `require(".").example`, which will try to find exported variable with name 'example' in current file. 
Or you can use `require("./someFile.ts")`, which will try to use default exported variable from 'someFile.ts'.

Note: For `examples` a required variable must be an array.

## Examples

* | root  
  * `npm install`
  * `npm build`
* | SOME [test's program](test/programs)
  * _Example:_ [namespace](test/programs/namespace)
  * `node ../../../bin/typescript-json-schema main.ts Type`
  * 👀check the stdout == schema.json 👀

## Background

* -- inspired on -- [Typson](https://github.com/lbovet/typson/) 
* if you are looking for a library / BETTER support for type aliases -> see [vega/ts-json-schema-generator](https://github.com/vega/ts-json-schema-generator)
  * Reason: 🧠uses the AST🧠  

## How to debug?

* `npm run debug -- test/programs/type-alias-single/main.ts --aliasRefs true MyString`
* connect -- via the -- debugger protocol

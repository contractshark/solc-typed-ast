# solc-typed-ast

A TypeScript package providing a normalized typed Solidity AST along with the utilities necessary to generate the AST (from Solc) and traverse/manipulate it.

```bash
npm i -g https://github.com/contractshark/solc-typed-ast/releases/download/v5.0.6/solc-typed-ast-5.0.5.tgz
```

## Features

-   Various [solc-js](https://www.npmjs.com/package/solc) compiler versions support, starting from **0.4.13**.
-   Various compiler selection strategies, including compiler auto-detection via `pragma solidity` directives in Solidity sources with possible fallback options.
-   Interaction with AST nodes, using type definitions.
-   Compensation of differences between legacy and compact Solidity AST.
-   Referenced AST nodes linking.
-   Easy tree traversal with siblings, parent/children relations, recursive walks and node selection via custom predicates.
-   XPath AST traversal (based on [jSel package](https://www.npmjs.com/package/jsel)).
-   Generating the source code from the AST.

## Installation

Package could be installed globally via following command:

```bash
npm install -g solc-typed-ast
```

Also it can be installed as the dependency:

```bash
npm install --save solc-typed-ast
```

## Usage

### Easy compiling

The package introduces easy and universal compiler invocation. Starting with **Solidity 0.4.13**, source compilation could be done universally:

```typescript
import { CompileFailedError, CompileResult, compileSol } from "solc-typed-ast";

let result: CompileResult;

try {
    result = compileSol("sample.sol", "auto", []);
} catch (e) {
    if (e instanceof CompileFailedError) {
        console.error("Compile errors encountered:");

        for (const failure of e.failures) {
            console.error(`SolcJS ${failure.compilerVersion}:`);

            for (const error of failure.errors) {
                console.error(error);
            }
        }
    } else {
        console.error(e.message);
    }
}

console.log(result);
```

The second argument with the `"auto"` value specifies a compiler selection strategy. If `"auto"` is specified and source code contains valid `pragma solidity` directive, then compiler version will be automatically picked from it. If compile process will not succedd, the execution will fall back to _"compiler guessing"_: trying to compile source with a few different versions of the new and old Solidity compilers. The other option would be to specify a concrete supported compiler version string, like `"0.7.0"` for example. There is also a support for various compiler selection strategies, including used-defined custom ones (`CompilerVersionSelectionStrategy` interface implementations).

**NOTE:** We want to notify that **the package preinstalls 50+ SolcJS compilers** (since 0.4.13), so be ready that it will occupy fair amount of free space. At some point we may consider to move compilers away from this package. It will mostly depend on users feedback.

### Typed universal AST

After the source is compiled and original compiler has provided the raw AST, the `ASTReader` could be used to read the typed universal AST:

```typescript
import { ASTReader } from "solc-typed-ast";

const reader = new ASTReader();
const sourceUnits = reader.read(result.data);

console.log("Used compiler version: " + result.compilerVersion);
console.log(sourceUnits[0].print());
```

The typed universal AST has following benefits:

-   It is _universal_ between legacy and compact AST. Solc changed AST format between releases, so dependent packages were either forced to migrate from one raw AST version to another or to build adaptive logic (such as implemented in this package).
-   It has TypeScript types and proper AST node classes hierarchy.
-   It has built-in tree traversal routines, like `walk()`, `getChildrenBySelector()`, `getParents()`, `getClosestParentBySelector()` and so on.

### Converting an AST back to source

One of the goals of each AST is to provide a way for programmatic modification. To do that, you could modify properties of the typed universal AST nodes. Then use the `ASTWriter` to write modified AST back to source code:

```typescript
import {
    ASTWriter,
    DefaultASTWriterMapping,
    LatestCompilerVersion,
    PrettyFormatter
} from "solc-typed-ast";

const formatter = new PrettyFormatter(4, 0);
const writer = new ASTWriter(
    DefaultASTWriterMapping,
    formatter,
    result.compilerVersion ? result.compilerVersion : LatestCompilerVersion
);

for (const sourceUnit of sourceUnits) {
    console.log("// " + sourceUnit.absolutePath);
    console.log(writer.write(sourceUnit));
}
```

### CLI tool

Package bundles a `sol-ast-compile` CLI tool to provide help with development process. It is able to compile the Solidity source and output short AST structure with following:

```bash
sol-ast-compile sample.sol --tree
```

Use `--help` to see all available features.

## Project overview

The project have following directory structure:

```
├── coverage                    # Test coverage report, produced by "npm test" command.
├── dist                        # Generated JavaScript sources for package distribution (produced by "npm run build" and published by "npm publish" commands).
├── docs                        # Project documentation and API reference, produced by "npm run docs:render" or "npm run docs:refresh" commands.
├── src                         # Original TypeScript sources.
│   ├── ast                     # AST-related definitions and logic:
│   │   ├── implementation      #   - Implemented universal AST nodes:
│   │   │   ├── declaration     #       - declarations or definitions;
│   │   │   ├── expression      #       - expressions;
│   │   │   ├── meta            #       - directives, units, specifiers and other information nodes;
│   │   │   ├── statement       #       - statements;
│   │   │   └── type            #       - type-related nodes.
│   │   ├── legacy              #   - Solc legacy raw AST processors, that are producing arguments for constrcuting universal AST nodes.
│   │   ├── modern              #   - Solc modern (or compact) raw AST processors, that are producing arguments for constrcuting universal AST nodes.
│   │   ├── postprocessing      #   - AST postprocessors to apply additional logic (fixes and discovery) during tree finalization process.
│   │   └── writing             #   - Components to convert universal AST nodes back to Solidity source code.
│   ├── bin                     # Executable files, that are shipped with the package and deployed via "npm install" or "npm link" commands.
│   ├── compile                 # Compile-related definitions and logic.
│   ├── misc                    # Miscellaneous functionality and utility modules.
│   └── types                   # Solc AST typeString parser and AST.
└── test                        # Tests:
    ├── integration             #   - Integration test suites.
    ├── samples                 #   - Solidity and compiler ourput JSON samples for the tests.
    └── unit                    #   - Unit test suites.
```

A key points for better understanding:

-   The `ASTNode` is generic implementation of universal AST node and a base class for all concrete node implementations, _except Yul nodes_.
-   The `ASTReader` takes raw Solc AST (legacy or modern) and produces universal AST.
-   The `ASTContext` provides node-to-node dynamic reference resolution map. The example of such reference would be a `vReferencedDeclaration` property of the `Identifier` node.
-   The `LegacyNodeProcessor` class, `ModernNodeProcessor` class and their descendant classes are raw-to-universal AST conversion bridge.
-   The compiling-related logic (located in `src/compile/*`) should not have imports from or any bindings to the universal AST (`src/ast/*`). Compile-related utils should be standalone as much as possible.

## Development installation

Install prerequisites:

-   NodeJS version **12.x** or newer.
-   NPM version **6.9.0** or newer.

Clone repository:

```bash
git clone https://github.com/ConsenSys/solc-typed-ast.git
```

Install and link:

```bash
npm install
npm link
```

## Project documentation and API reference

The project documentation is contained in the `docs/` directory. It could be built via following command:

```bash
npm run docs:refresh
```

It is also published here: https://consensys.github.io/solc-typed-ast/

### The list of AST node types

The list of known AST node types can be found [here](https://github.com/ConsenSys/solc-typed-ast/blob/master/NODE_LIST.md).

# Bytenode

A minimalist bytecode compiler for Node.js.

This tool truly compiles your JavaScript code into `V8` bytecode, so that you can protect your source code. It can be used with Node.js >= 5.7.x, as well as Electron and NW.js (check `examples/` directory).

---

## Install

```console
npm install --save bytenode
```

Or globally:

```console
sudo npm install -g bytenode
```

---

## Known Issues and Limitations

* In Node 10.x, Bytenode does not work in debug mode. See [#29](https://github.com/OsamaAbbas/bytenode/issues/29).

* Any code depends on `Function.prototype.toString` function will break, because Bytenode remove the source code from `.jsc` files and put a dummy code instead. See [#34](https://github.com/OsamaAbbas/bytenode/issues/34).

* In recent versions of Node, the `--no-flush-bytecode` must be set. Bytenode sets it internally, but if you encounter any issues, try to run Node with that flag: ` $ node --no-flush-bytecode server.js`. See [#41](https://github.com/OsamaAbbas/bytenode/issues/41).

---

## Bytenode CLI

```
  Usage: bytenode [option] [ FILE... | - ] [arguments]

  Options:
    -h, --help                        show help information.
    -v, --version                     show bytenode version.

    -c, --compile [ FILE... | - ]     compile stdin, a file, or a list of files
        --no-module                   compile without producing commonjs module

  Examples:

  $ bytenode -c script.js             compile `script.js` to `script.jsc`.
  $ bytenode -c server.js app.js
  $ bytenode -c src/*.js              compile all `.js` files in `src/` directory.

  $ bytenode script.jsc [arguments]   run `script.jsc` with arguments.
  $ bytenode                          open Node REPL with bytenode pre-loaded.
```

Examples:

* Compile `express-server.js` to `express-server.jsc`.
```console
user@machine:~$ bytenode --compile express-server.js
```

* Run your compiled file `express-server.jsc`.
```console
user@machine:~$ bytenode express-server.jsc
Server listening on port 3000
```

* Compile all `.js` files in `./app` directory.
```console
user@machine:~$ bytenode --compile ./app/*.js
```

* Compile all `.js` files in your project.
```console
user@machine:~$ bytenode --compile ./**/*.js
```
Note: you may need to enable `globstar` option in bash (you should add it to `~/.bashrc`):
`shopt -s globstar`

* Starting from v1.0.0, bytenode can compile from `stdin`.
```console
$ echo 'console.log("Hello");' | bytenode --compile - > hello.jsc
```

---

## Bytenode API

You have to run node with this flag `node --no-lazy` in order to compile all functions in your code eagerly.
Or, if you have no control on how your code will be run, you can use `v8.setFlagsFromString('--no-lazy')` function as well.

```javascript
const bytenode = require('bytenode');
```

---

#### bytenode.compileCode(javascriptCode) → {Buffer}

Generates v8 bytecode buffer.

- Parameters:

| Name           | Type   | Description                                          |
| ----           | ----   | -----------                                          |
| javascriptCode | string | JavaScript source that will be compiled to bytecode. |

- Returns:

{Buffer} The generated bytecode.

- Example:

```javascript
let helloWorldBytecode = bytenode.compileCode(`console.log('Hello World!');`);
```
This `helloWorldBytecode` bytecode can be saved to a file. However, if you want to use your code as a module (i.e. if your file has some `exports`), you have to compile it using `bytenode.compileFile({compileAsModule: true})`, or wrap your code manually, using `Module.wrap()` function.

---

#### bytenode.runBytecode(bytecodeBuffer) → {any}

Runs v8 bytecode buffer and returns the result.

- Parameters:

| Name           | Type   | Description                                                    |
| ----           | ----   | -----------                                                    |
| bytecodeBuffer | Buffer | The buffer object that was created using compileCode function. |

- Returns:

{any} The result of the very last statement executed in the script.

- Example:

```javascript
bytenode.runBytecode(helloWorldBytecode);
// prints: Hello World!
```

---

#### bytenode.compileFile(args, output) → {string}

Compiles JavaScript file to .jsc file.

- Parameters:

| Name                 | Type             | Description                                                                                              |
| ----                 | ----             | -----------                                                                                              |
| args                 | object \| string |                                                                                                          |
| args.filename        | string           | The JavaScript source file that will be compiled.                                                        |
| args.compileAsModule | boolean          | If true, the output will be a commonjs module. Default: true.                                            |
| args.output          | string           | The output filename. Defaults to the same path and name of the original file, but with `.jsc` extension. |
| output               | string           | The output filename. (Deprecated: use args.output instead)                                               |

- Returns:

{string}: The compiled filename.

- Examples:

```javascript
let compiledFilename = bytenode.compileFile({
  filename: '/path/to/your/file.js',
  output: '/path/to/compiled/file.jsc' // if omitted, it defaults to '/path/to/your/file.jsc'
});
```
Previous code will produce a commonjs module that can be required using `require` function.

```javascript
let compiledFilename = bytenode.compileFile({
  filename: '/path/to/your/file.js',
  output: '/path/to/compiled/file.jsc',
  compileAsModule: false
});
```
Previous code will produce a direct `.jsc` file, that can be run using `bytenode.runBytecodeFile()` function. It can NOT be required as a module. Please note that `compileAsModule` MUST be `false` in order to turn it off. Any other values (including: `null`, `""`, etc) will be treated as `true`. (It had to be done this way in order to keep the old code valid.)

---

#### bytenode.runBytecodeFile(filename) → {any}

Runs .jsc file and returns the result.

- Parameters:

| Name     | Type   |
| ----     | ----   |
| filename | string |

- Returns:

{any} The result of the very last statement executed in the script.

- Example:

```javascript
// test.js
console.log('Hello World!');
```

```javascript
bytenode.runBytecodeFile('/path/to/test.jsc');
// prints: Hello World!
```

---

#### require(filename) → {any}

- Parameters:

| Name     | Type   |
| ----     | ----   |
| filename | string |

- Returns:

{any} exported module content

- Example:

```javascript
let myModule = require('/path/to/your/file.jsc');
```
Just like regular `.js` modules. You can also omit the extension `.jsc`.

`.jsc` file must have been compiled using `bytenode.compileFile()`, or have been wrapped inside `Module.wrap()` function. Otherwise it won't work as a module and it can NOT be required.

Please note `.jsc` files must run with the same Node.js version that was used to compile it (using same architecture of course). Also, `.jsc` files are CPU-agnostic. However, you should run your tests before and after deployment, because V8 sanity checks include some checks related to CPU supported features, so this may cause errors in some rare cases.

---

## Acknowledgements

I had the idea of this tool many years ago. However, I finally decided to implement it after seeing this [issue](https://github.com/nodejs/node/issues/11842) by @hashseed. Also, some parts was inspired by [v8-compile-cache](https://github.com/zertosh/v8-compile-cache) by @zertosh.
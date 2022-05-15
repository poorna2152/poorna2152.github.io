---
title: "WebAssembly backend for Ballerina"
date: 2022-05-15T11:30:03+05:30
tags: ["wasm", "ballerina"]
author: "Me"
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: true
disableHLJS: true # to disable highlightjs
disableShare: false
disableHLJS: false
hideSummary: false
searchHidden: true
ShowReadingTime: true
ShowBreadCrumbs: true
ShowPostNavLinks: true
ShowRssButtonInSectionTermList: true
cover:
    image: "/blog/img/cover-wasm.png" # image path/url
    alt: "Cover Photo" # alt text
    relative: false # when using page bundles set this to true
    hidden: false # only hide on current single page
editPost:
    URL: "https://github.com/poorna2152/blog/tree/main/content"
    Text: "Suggest Changes" # edit text
    appendFilePath: true # to append file path to Edit link
---

Up to recently Javascript has been the only language which we can use to write programs for web browsers. Javascript is not suitable to run resource intensive applications such as AR-VR, video editing and 3D games. WebAssembly (wasm) was initially proposed as a compilation target for higher level languages which enabled them to run on web browsers with the additional performance gains for running computational intensive applications. Tensorflow, Unity, GoogleEarth and many more applications already use wasm to run on web browsers. With the invention of Web Assembly System Interface (WASI) the use cases of wasm have moved beyond the web browser. It is now fast becoming a language which is used for edge computing. Cloudflare workers, Netflify, Fastly provides options to deploy the code as wasm modules. Currently there are wasm compilers for 40+ languages such as C, C++, Java, Rust etc.

My internship project is to develop a backend for nBallerina compiler for generating wasm. Ballerina is an open source programming language designed specifically for writing distributed applications. nBallerina is the native compiler of the Ballerina language which is developed incrementally in subsets. Each subset adds a set of features to the language. Having a wasm backend for the Ballerina language provides the capability for it to run in a web browser and in standalone Javascript runtimes such as Node.

### Plan for WASM Backend.

WebAssembly has two formats. The binary format and the text format. Text format (wat) is a human readable format. 
Our plan is to emit the wasm text format and compile it to binary format using the Binaryen tool. Binaryen is a compiler and toolchain infrastructure library for WebAssembly which provides a C-API which can be used to generate wasm and it also provides tools which can be used to convert the wasm text format to binary format.
Since nBallerina compiler is yet to have a Foreign Function Interface we could not directly call the Binaryen C-API from the Ballerina code to produce wasm. Thus what we decided was to create a Ballerina module replicating the functionality of the Binaryen C-API. The functions in this module are then called to emit the wasm Text Format. 
nBallerina compiler frontend emits the Ballerina Intermediate Representation(BIR) which is then consumed by the backend to generate LLVM. This BIR is also taken as the input for the new wasm  backend. The generated wat file is converted to binary format using the Binaryen Optimizer tool which optimizes the code as well as outputs the binary format. Then the generated wasm file is run using a Javascript file.

![Flow](/blog/img/flow.png)

The Ballerina Intermediate representation is an unstructured control flow graph, but wasm is a structured control flow graph. Initially in the subset01 we were using the Relooper algorithm to convert the unstructured BIR to structured format. But later we added structured information to the BIR using a concept of Regions thus allowing us to remove the use of the Relooper algorithm.

Another simple approach to generate wasm would have been to use the LLVM emitted from the existing nBallerina backend and convert this to wasm. This is a straightforward conversion but we are not doing this because if we are to directly convert LLVM to wasm we might not be able to benefit from the higher level constructs that wasm provides for us such as Garbage collection (You'll see why later in the article).


### Implementation of subset01.
Subset01 of the language consist of,
-	Types: `Boolean`, `Integers` 
-	Constructs: if-else and while, break, continue
-	function println

wasm has integer types `i32` and `i64`. Ballerina `booleans` are mapped to an `i32` and `integers` are mapped to `i64s` in the wasm.
In subset01 `println` function is called with an `i64` value. `Println` function was implemented using the `console.log` function in Javascript.

Let's consider the following simple program.

```go
    import ballerina/io;

    public function main() {
        int i = 0;
        int sum = 0;
        while i < 15 {
            i = i + 1;
            sum = sum + 1;
        }
        io:println(sum);
    }
```

For this program the blocks in BIR representation and the corresponding WAT instructions are as follows.

![BIR](/blog/img/bir.png)

The `(%number)` format in the above represents a register number in the ballerina. The registers are mapped to local variables in wasm function. The complete code looks like this.

``` Lisp
(module 
  (import "console" "log" (func $println (param i64)))
  (export "main" (func $main)) 
  (func $main 
    (local $0 i64) 
    (local $1 i64) 
    (local.set $0 
      (i64.const 0)) 
    (local.set $1 
      (i64.const 0)) 
    (loop $block$1$continue 
      (if 
        (i64.lt_s 
          (local.get $0) 
          (i64.const 15))
        (block
          (local.set $0
            (i64.add
              (local.get $0)
              (i64.const 1)))
          (local.set $1 
            (i64.add 
              (local.get $1) 
              (local.get $0))) 
          (br $block$1$continue)))) 
    (call $println
      (local.get $1))))
```

### Implementation of subset02.

Subset02 of the language contains the type `any`. A `boolean`, `integer` or `nil` value can be assigned to a variable of type `any` and a value assigned to an `any` can be casted back to its original type. The next step in implementation was to figure out the wasm type that we can use to represent `any`.

```go
    int x = 1;
    any y = x;
    int z = <int>y;
    boolean b = true;
    y = b;
    boolean c = <boolean>y;
```

The variable `y` which is an `any` is casted to an `int` and assigned to variable `z`. Thus there should be a way to identify the runtime type of the value assigned to a variable of type `any`, so we can ensure that the castings are valid.
My first attempt in approaching this problem was to look at what nBallerina already does and replicate its behavior in the wasm side.

In nBallerina a variable of type `any` is represented with the LLVM type `i64`. The first 56 bits out of 64 were used to represent the actual value; the remaining 8 bits were used as a tag. This tag contains information about the runtime type of the variable stored. 

Since `integers` are also assigned to `any`, what if we want to assign an integer whose binary representation uses more than 56 bits to a variable of type `any`. In this case nBallerina stores the actual value in the memory and then gets the memory address and stores this memory address in the first 56 bits. There is also a bit in the tag to represent whether the value is an immediate or a memory stored one.

![tagging](/blog/img/tag.png)

Wasm has type `i64` and it also has a memory to which we can store and retrieve values. So we should be able to replicate this in the wasm side right? Unfortunately no.
 
When we are storing values in memory we need to consider garbage collection. wasm doesn't automatically garbage collect the values stored in memory. So this approach is not feasible.

However wasm already has a Garbage Collection(GC) proposal. Even though this is a proposal there is an implemented MVP. This proposal introduces a set of reference types which are garbage collected. Namely the `i31ref`, `struct` and `array`.
`Struct` is a data type similar to a C struct where we can define a set of fields and their types. 

Numerical values which can be represented using 31 bits can be stored in an `i31`. `i31ref` is capable of storing references in the form of offsets in wasm linear memory, as well as numbers. Storing numbers in an `i31ref`, instead of wrapping in a `struct`, allows faster retrieval because it lives in the stack. So the plan was to use `i31` to store `booleans`, `struct` to store `ints` and `null` to store `null`. So the wasm type of a variable of type `any` would be a reference type (type `eqref` in wasm).

![tagging an int](/blog/img/tagint.png)

When a variable is to be assigned to a variable of type `any` it is converted to the respective reference type. As per the above diagram  the `int x` is converted to a reference type `struct` using the `int_to_tagged()` function. We can give a type name, field type and field names when defining a struct. In this case it is a struct of type `$BoxedInt` with a field i64 to store the `int` value. 

In subset 02 `println` function  was called with a variable of type `any`. When `println` is called with a value of simple basic type it is first converted to `any` type. Then in Javascript side `console.log` checks for the type of the reference it got and retrieves the value accordingly. The functions for converting to `any` type, retrieving the original value back from `any` type and retrieving the runtime type of `any` variable are implemented in wasm and exported to Javascript so it can use them as necessary.

![println](/blog/img/println.png)

This project is still going and currently I have finished subset03 of the language.
The progress I have made so far has been with the help of my mentor Manuranga Perera, nBallerina team lead James Clark and the nBallerina team.

### References
1.	Repository: https://github.com/poorna2152/nballerina
2.  Current Branch: https://github.com/poorna2152/nballerina/tree/wback_subset04
3.	nBallerina runtime document: https://github.com/poorna2152/nballerina/blob/main/docs/rtvalue.md
4.	Wasm use cases: https://madewithwebassembly.com/
5.	Wasm compilers: https://github.com/appcypher/awesome-wasm-langs
6.	https://blog.container-solutions.com/webassembly-in-the-cloud



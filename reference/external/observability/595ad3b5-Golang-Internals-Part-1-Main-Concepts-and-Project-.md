---
title: Golang Internals, Part 1: Main Concepts and Project Structure | Altoros
source: https://blog.altoros.com/golang-part-1-main-concepts-and-project-structure.html
kind: external
domain: observability
author: Siarhei Matsiukevich
original_date: 2015-03-02
fetched_at: 2026-05-16
tags: [external, observability]
---

> [!info] 外部文章 · 自动导入
> 来源：[blog.altoros.com](https://blog.altoros.com/golang-part-1-main-concepts-and-project-structure.html)
> 作者：Siarhei Matsiukevich
> 原始日期：2015-03-02
> 抓取日期：2026-05-16

# Golang Internals, Part 1: Main Concepts and Project Structure | Altoros

# Golang Internals, Part 1: Main Concepts and Project Structure

**Part 1**| Part 2 | Part 3 | Part 4 | Part 5 | Part 6

This series of blog posts is intended for those who are already familiar with the basics of Go and would like to get a deeper insight into its internals. This tutorial is dedicated to the structure of the Go source code and some internal details of the Go compiler. After reading this, you should be able to answer the following questions:

- What is the structure of the Go source code?
- How does the Go compiler work?
- What is the basic structure of a node tree in Go?


### Getting started

When you start learning a new programming language, you can usually find a lot of “hello-world” tutorials, beginner guides, and books with details on the main language concepts, syntax, and even the standard library. However, getting information on such things as the layout of major data structures that the language runtime allocates or what assembly code is generated when you call a built-in function is not that easy. Obviously, the answers lie inside the source code, but, from our own experience, you can spend hours wandering through it without making much progress. The goal of this article is to demonstrate how you can decipher Go sources on your own.

Before we can begin, we certainly need our own copy of the Go source files. There is nothing special in getting them. Just execute the command below.

1 | git clone <a href="https://github.com/golang/go" target="_blank" rel="noopener noreferrer">https://github.com/golang/go</a> |

Please note that the code in the main branch is being constantly changed, so we use the `release-branch.go1.4`

branch in this blog post.


### Understanding a project structure

If you look at the `/src`

folder of the Go repository, you can see a lot of folders. Most of them contain source files of the standard Go library. The standard naming conventions are always applied here, so each package is inside a folder with a name that directly corresponds to the package name. Apart from the standard library, there is a lot of other stuff. In our opinion, the most important and useful folders are listed in the table below.

| /src/cmd/ | Contains different command line tools. |
| /src/cmd/go/ | Contains source files of a Go tool that downloads and builds Go source files and installs packages. While doing this, it collects all source files and makes calls to the Go linker and Go compiler command line tools. |
| /src/cmd/dist/ | Contains a tool responsible for building all other command line tools and all the packages from the standard library. You may want to analyze its source code to understand what libraries are used in every particular tool or package. |
| /src/cmd/gc/ | This is the architecture-independent part of the Go compiler. |
| /src/cmd/ld/ | The architecture-independent part of the Go linker. Architecture-dependent parts are located in the folder with the “l” postfix that uses the same naming conventions as the compiler. |
| /src/cmd/5a/, 6a, 8a, and 9a | Here you can find Go assembler compilers for different architectures. The Go assembler is a form of assembly language that does not map precisely to the assembler of the underlying machine. Instead, there is a distinct compiler for each architecture that translates the Go assembler to the machine’s assembler. You can find more details in the official documentation. |
| /src/lib9/, /src/libbio, /src/liblink | Different libraries that are used inside the compiler, linker, and runtime package. |
| /src/runtime/ | The most important Go package that is indirectly included into all programs. It contains the entire runtime functionality, such as memory management, garbage collection, goroutines creation, etc. |


### Inside the Go compiler

As we mentioned above, the architecture-independent part of the Go compiler is located in the /src/cmd/gc/ folder. The entry point is located in the lex.c file. Apart from some common stuff, such as parsing command line arguments, the compiler does the following:

- Initializes some common data structures.
- Iterates through all of the provided Go files and calls the yyparse method for each file. This causes actual parsing to occur. The Go compiler uses Bison as the parser generator. The grammar for the language is fully described in the go.y file (I will provide more details on it later). As a result, this step generates a complete parse tree where each node represents an element of the compiled program.
- Recursively iterates through the generated tree several times and applies some modifications, e.g., defines type information for the nodes that should be implicitly typed, rewrites some language elements—such as typecasting—into calls to some functions in the runtime package and does some other work.
- Performs the actual compilation after the parse tree is complete. Nodes are translated into assembler code.
- Creates the object file that contains generated assembly code with some additional data structures, such as the symbols table, which is generated and written to the disk.


### Diving into the Go grammar

Now, lets take a closer look at the second step. The go.y file that contains the language grammar is a good starting point for investigating the Go compiler and the key to understanding the language syntax. The main part of this file consists of declarations, similar to the following:

1 2 3 4 5 6 | xfndcl: LFUNC fndcl fnbody fndcl: sym '(' oarg_type_list_ocomma ')' fnres | '(' oarg_type_list_ocomma ')' sym '(' oarg_type_list_ocomma ')' fnres |

In this declaration, the `xfndcl`

and `fundcl`

nodes are defined. The `fundcl`

node can be in one of the two forms. The first form corresponds to the following language construct:

1 | somefunction(x int, y int) int |

and the second one to this language construct:

1 | (t *SomeType) somefunction(x int, y int) int. |

The `xfndcl`

node consists of the `func`

keyword that is stored in the `LFUNC`

constant, followed by the `fndcl`

and `fnbody`

nodes.

An important feature of Bison (or Yacc) grammar is that it allows for placing arbitrary C code next to each node definition. The code is executed every time a match for this node definition is found in the source code. Here, you can refer to the result node as `$$`

and to the child nodes as `$1`

, `$2`

, etc.

It is easier to understand this through an example. Note that the following code is a shortcut version of the actual code.

1 2 3 4 5 6 7 8 9 10 11 12 13 | fndcl: sym '(' oarg_type_list_ocomma ')' fnres { t = nod(OTFUNC, N, N); t->list = $3; t->rlist = $5; $$ = nod(ODCLFUNC, N, N); $$->nname = newname($1); $$->nname->ntype = t; declare($$->nname, PFUNC); } | '(' oarg_type_list_ocomma ')' sym '(' oarg_type_list_ocomma ')' fnres |

First, a new node is created, which contains type information for the function declaration. The `$3`

argument list and the `$5`

result list are referenced from this node. Then, the `$$`

result node is created. It stores the function name and the type node. As you can see, there can be no direct correspondence between definitions in the `go.y`

file and the node structure.


### Understanding nodes

Now it is time to take a look at what a node actually is. First of all, a node is a struct (you can find a definition here). This struct contains a large number of properties, since it needs to support different kinds of nodes and different nodes have different attributes. Below is a description of several fields that I think are important to understand.

| op | Node operation. Each node has this field. It distinguishes different kinds of nodes from each other. In our previous example, those were `OTFUNC` (operation type function) and `ODCLFUNC` (operation declaration function). |
| type | This is a reference to another struct with type information for nodes that have type information (there are no types for some nodes, e.g., control flow statements, such as `if` , `switch` , or `for` ). |
| val | This field contains the actual values for nodes that represent literals. |

Now that you understand the basic structure of the node tree, you can put your knowledge into practice. In the next post, we will investigate what exactly the Go compiler generates, using a simple Go application as an example.


### Further reading

- Golang Internals, Part 2: Diving Into the Go Compiler
- Golang Internals, Part 3: The Linker, Object Files, and Relocations
- Golang Internals, Part 4: Object Files and Function Metadata
- Golang Internals, Part 5: the Runtime Bootstrap Process
- Golang Internals, Part 6: Bootstrapping and Memory Allocator Initialization

**Siarhei Matsiukevich**and edited by Sophia Turol and Alex Khizhniak.
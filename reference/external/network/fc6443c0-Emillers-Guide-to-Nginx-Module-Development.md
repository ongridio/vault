---
title: Emiller’s Guide to Nginx Module Development
source: https://www.evanmiller.org/nginx-modules-guide.html
kind: external
domain: network
author: Evan Miller
original_date: 2007-04-28
fetched_at: 2026-05-16
bookmark_title: Emiller’s Guide to Nginx Module Development – Evan Miller
tags: [external, network]
---

> [!info] 外部文章 · 自动导入
> 来源：[www.evanmiller.org](https://www.evanmiller.org/nginx-modules-guide.html)
> 作者：Evan Miller
> 原始日期：2007-04-28
> 抓取日期：2026-05-16

# Emiller’s Guide to Nginx Module Development

# Emiller’s Guide To Nginx Module Development

By Evan Miller

First published: April 28, 2007 (Last edit: August 11, 2017 – changes)

*What’s that?*

Lucius Fox:

*The Tumbler? Oh… you wouldn’t be interested in that.*

To fully appreciate Nginx, the web server, it helps to understand Batman, the comic book character.

Batman is fast. Nginx is fast. Batman fights crime. Nginx fights wasted CPU cycles and memory leaks. Batman performs well under pressure. Nginx, for its part, excels under heavy server loads.

But Batman would be almost nothing without the **Batman utility belt**.

**Figure 1**: The Batman utility belt, gripping Christian Bale’s love handles.

At any given time, Batman’s utility belt might contain a lock pick, several batarangs, bat-cuffs, a bat-tracer, bat-darts, night vision goggles, thermite grenades, smoke pellets, a flashlight, a kryptonite ring, an acetylene torch, or an Apple iPhone. When Batman needs to tranquilize, blind, deafen, stun, track, stop, smoke out, or text-message the enemy, you better believe he’s reaching down for his bat-belt. The belt is so crucial to Batman’s operations that if Batman had to choose between wearing pants and wearing the utility belt, he would definitely choose the belt. In fact, he *did* choose the utility belt, and that’s why Batman wears rubber tights instead of pants (Fig. 1).

Instead of a utility belt, Nginx has a **module chain**. When Nginx needs to gzip or chunk-encode a response, it whips out a module to do the work. When Nginx blocks access to a resource based on IP address or HTTP auth credentials, a module does the deflecting. When Nginx communicates with Memcache or FastCGI servers, a module is the walkie-talkie.

Batman’s utility belt holds a lot of doo-hickeys, but occasionally Batman needs a new tool. Maybe there’s a new enemy against whom bat-cuffs and batarangs are ineffectual. Or Batman needs a new ability, like being able to breathe underwater. That’s when Batman rings up **Lucius Fox** to engineer the appropriate bat-gadget.

**Figure 2**: Bruce Wayne (née Batman) consults with his engineer, Lucius Fox

The purpose of this guide is to teach you the details of Nginx’s module chain, so that you may be like Lucius Fox. When you’re done with the guide, you’ll be able to design and produce high-quality modules that enable Nginx to do things it couldn’t do before. Nginx’s module system has a lot of nuance and nitty-gritty, so you’ll probably want to refer back to this document often. I have tried to make the concepts as clear as possible, but I’ll be blunt, writing Nginx modules can still be hard work.

But whoever said making bat-tools would be easy?

## Table of Contents

- Prerequisites
- High-Level Overview of Nginx’s Module Delegation
- Components of an Nginx Module
- Module Configuration Struct(s)
- Module Directives
- The Module Context
- The Module Definition
- Module Installation
- Handlers
- Anatomy of a Handler (Non-proxying)
- Anatomy of an Upstream (a.k.a. Proxy) Handler
- Handler Installation
- Filters
- Load-Balancers
- The enabling directive
- The registration function
- The upstream initialization function
- The peer initialization function
- The load-balancing function
- The peer release function
- Writing and Compiling a New Nginx Module
- Advanced Topics
- Code References

## 0. Prerequisites

You should be comfortable with C. Not just "C-syntax"; you should know your way around a struct and not be scared off by pointers and function references, and be cognizant of the preprocessor. If you need to brush up, nothing beats K&R.

Basic understanding of HTTP is useful. You’ll be working on a web server, after all.

You should also be familiar with Nginx’s configuration file. If you’re not, here’s the gist of it: there are four *contexts* (called *main*, *server*, *upstream*, and *location*) which can contain directives with one or more arguments. Directives in the main context apply to everything; directives in the server context apply to a particular host/port; directives in the upstream context refer to a set of backend servers; and directives in a location context apply only to matching web locations (e.g., "/", "/images", etc.) A location context inherits from the surrounding server context, and a server context inherits from the main context. The upstream context neither inherits nor imparts its properties; it has its own special directives that don’t really apply elsewhere. I’ll refer to these four contexts quite a bit, so… don’t forget them.

Let’s get started.

## 1. High-Level Overview of Nginx’s Module Delegation

Nginx modules have three roles we’ll cover:

*handlers*process a request and produce output*filters*manipulate the output produced by a handler*load-balancers*choose a backend server to send a request to, when more than one backend server is eligible

Modules do all of the "real work" that you might associate with a web server: whenever Nginx serves a file or proxies a request to another server, there’s a handler module doing the work; when Nginx gzips the output or executes a server-side include, it’s using filter modules. The "core" of Nginx simply takes care of all the network and application protocols and sets up the sequence of modules that are eligible to process a request. The de-centralized architecture makes it possible for *you* to make a nice self-contained unit that does something you want.

Note: Unlike modules in Apache, Nginx modules are *not* dynamically linked. (In other words, they’re compiled right into the Nginx binary.)

How does a module get invoked? Typically, at server startup, each handler gets a chance to attach itself to particular locations defined in the configuration; if more than one handler attaches to a particular location, only one will "win" (but a good config writer won’t let a conflict happen). Handlers can return in three ways: all is good, there was an error, or it can decline to process the request and defer to the default handler (typically something that serves static files).

If the handler happens to be a reverse proxy to some set of backend servers, there is room for another type of module: the load-balancer. A load-balancer takes a request and a set of backend servers and decides which server will get the request. Nginx ships with two load-balancing modules: round-robin, which deals out requests like cards at the start of a poker game, and the "IP hash" method, which ensures that a particular client will hit the same backend server across multiple requests.

If the handler does not produce an error, the filters are called. Multiple filters can hook into each location, so that (for example) a response can be compressed and then chunked. The order of their execution is determined at compile-time. Filters have the classic "CHAIN OF RESPONSIBILITY" design pattern: one filter is called, does its work, and then calls the next filter, until the final filter is called, and Nginx finishes up the response.

The really cool part about the filter chain is that each filter doesn’t wait for the previous filter to finish; it can process the previous filter’s output as it’s being produced, sort of like the Unix pipeline. Filters operate on *buffers*, which are usually the size of a page (4K), although you can change this in your nginx.conf. This means, for example, a module can start compressing the response from a backend server and stream it to the client before the module has received the entire response from the backend. Nice!

So to wrap up the conceptual overview, the typical processing cycle goes:

Client sends HTTP request → Nginx chooses the appropriate handler based on the location config → (if applicable) load-balancer picks a backend server → Handler does its thing and passes each output buffer to the first filter → First filter passes the output to the second filter → second to third → third to fourth → etc. → Final response sent to client

I say "typically" because Nginx’s module invocation is *extremely* customizable. It places a big burden on module writers to define exactly how and when the module should run (I happen to think too big a burden). Invocation is actually performed through a series of callbacks, and there are a lot of them. Namely, you can provide a function to be executed:

- Just before the server reads the config file
- For every configuration directive for the location and server for which it appears;
- When Nginx initializes the main configuration
- When Nginx initializes the server (i.e., host/port) configuration
- When Nginx merges the server configuration with the main configuration
- When Nginx initializes the location configuration
- When Nginx merges the location configuration with its parent server configuration
- When Nginx’s master process starts
- When a new worker process starts
- When a worker process exits
- When the master exits
- Handling a request
- Filtering response headers
- Filtering the response body
- Picking a backend server
- Initiating a request to a backend server
*Re*-initiating a request to a backend server- Processing the response from a backend server
- Finishing an interaction with a backend server

Holy mackerel! It’s a bit overwhelming. You’ve got a lot of power at your disposal, but you can still do something useful using only a couple of these hooks and a couple of corresponding functions. Time to dive into some modules.

## 2. Components of an Nginx Module

As I said, you have a *lot* of flexibility when it comes to making an Nginx module. This section will describe the parts that are almost always present. It’s intended as a guide for understanding a module, and a reference for when you think you’re ready to start writing a module.

### 2.1. Module Configuration Struct(s)

Modules can define up to three configuration structs, one for the main, server, and location contexts. Most modules just need a location configuration. The naming convention for these is `ngx_http_<module name>_(main|srv|loc)_conf_t`

. Here’s an example, taken from the dav module:

```
typedef struct {
ngx_uint_t methods;
ngx_flag_t create_full_put_path;
ngx_uint_t access;
} ngx_http_dav_loc_conf_t;
```


Notice that Nginx has special data types (`ngx_uint_t`

and `ngx_flag_t`

); these are just aliases for the primitive data types you know and love (cf. core/ngx_config.h if you’re curious).

The elements in the configuration structs are populated by module directives.

### 2.2. Module Directives

A module’s directives appear in a static array of `ngx_command_t`

s. Here’s an example of how they’re declared, taken from a small module I wrote:

```
static ngx_command_t ngx_http_circle_gif_commands[] = {
{ ngx_string("circle_gif"),
NGX_HTTP_LOC_CONF|NGX_CONF_NOARGS,
ngx_http_circle_gif,
NGX_HTTP_LOC_CONF_OFFSET,
0,
NULL },
{ ngx_string("circle_gif_min_radius"),
NGX_HTTP_MAIN_CONF|NGX_HTTP_SRV_CONF|NGX_HTTP_LOC_CONF|NGX_CONF_TAKE1,
ngx_conf_set_num_slot,
NGX_HTTP_LOC_CONF_OFFSET,
offsetof(ngx_http_circle_gif_loc_conf_t, min_radius),
NULL },
...
ngx_null_command
};
```


And here is the declaration of `ngx_command_t`

(the struct we’re declaring), found in core/ngx_conf_file.h:

```
struct ngx_command_t {
ngx_str_t name;
ngx_uint_t type;
char *(*set)(ngx_conf_t *cf, ngx_command_t *cmd, void *conf);
ngx_uint_t conf;
ngx_uint_t offset;
void *post;
};
```


It seems like a bit much, but each element has a purpose.

The `name`

is the directive string, no spaces. The data type is an `ngx_str_t`

, which is usually instantiated with just (e.g.) `ngx_str("proxy_pass")`

. Note: an `ngx_str_t`

is a struct with a `data`

element, which is a string, and a `len`

element, which is the length of that string. Nginx uses this data structure most places you’d expect a string.

`type`

is a set of flags that indicate where the directive is legal and how many arguments the directive takes. Applicable flags, which are bitwise-OR’d, are:

`NGX_HTTP_MAIN_CONF`

: directive is valid in the main config`NGX_HTTP_SRV_CONF`

: directive is valid in the server (host) config`NGX_HTTP_LOC_CONF`

: directive is valid in a location config`NGX_HTTP_UPS_CONF`

: directive is valid in an upstream config

`NGX_CONF_NOARGS`

: directive can take 0 arguments`NGX_CONF_TAKE1`

: directive can take exactly 1 argument`NGX_CONF_TAKE2`

: directive can take exactly 2 arguments- …
`NGX_CONF_TAKE7`

: directive can take exactly 7 arguments

`NGX_CONF_FLAG`

: directive takes a boolean ("on" or "off")`NGX_CONF_1MORE`

: directive must be passed at least one argument`NGX_CONF_2MORE`

: directive must be passed at least two arguments

There are a few other options, too, see core/ngx_conf_file.h.

The `set`

struct element is a pointer to a function for setting up part of the module’s configuration; typically this function will translate the arguments passed to this directive and save an appropriate value in its configuration struct. This setup function will take three arguments:

- a pointer to an
`ngx_conf_t`

struct, which contains the arguments passed to the directive - a pointer to the current
`ngx_command_t`

struct - a pointer to the module’s custom configuration struct

This setup function will be called when the directive is encountered. Nginx provides a number of functions for setting particular types of values in the custom configuration struct. These functions include:

`ngx_conf_set_flag_slot`

: translates "on" or "off" to 1 or 0`ngx_conf_set_str_slot`

: saves a string as an`ngx_str_t`

`ngx_conf_set_num_slot`

: parses a number and saves it to an`int`

`ngx_conf_set_size_slot`

: parses a data size ("8k", "1m", etc.) and saves it to a`size_t`


There are several others, and they’re quite handy (see core/ngx_conf_file.h). Modules can also put a reference to their own function here, if the built-ins aren’t quite good enough.

How do these built-in functions know where to save the data? That’s where the next two elements of `ngx_command_t`

come in, `conf`

and `offset`

. `conf`

tells Nginx whether this value will get saved to the module’s main configuration, server configuration, or location configuration (with `NGX_HTTP_MAIN_CONF_OFFSET`

, `NGX_HTTP_SRV_CONF_OFFSET`

, or `NGX_HTTP_LOC_CONF_OFFSET`

). `offset`

then specifies which part of this configuration struct to write to.

*Finally*, `post`

is just a pointer to other crap the module might need while it’s reading the configuration. It’s often `NULL`

.

The commands array is terminated with `ngx_null_command`

as the last element.

### 2.3. The Module Context

This is a static `ngx_http_module_t`

struct, which just has a bunch of function references for creating the three configurations and merging them together. Its name is `ngx_http_<module name>_module_ctx`

. In order, the function references are:

- preconfiguration
- postconfiguration
- creating the main conf (i.e., do a malloc and set defaults)
- initializing the main conf (i.e., override the defaults with what’s in nginx.conf)
- creating the server conf
- merging it with the main conf
- creating the location conf
- merging it with the server conf

These take different arguments depending on what they’re doing. Here’s the struct definition, taken from http/ngx_http_config.h, so you can see the different function signatures of the callbacks:

```
typedef struct {
ngx_int_t (*preconfiguration)(ngx_conf_t *cf);
ngx_int_t (*postconfiguration)(ngx_conf_t *cf);
void *(*create_main_conf)(ngx_conf_t *cf);
char *(*init_main_conf)(ngx_conf_t *cf, void *conf);
void *(*create_srv_conf)(ngx_conf_t *cf);
char *(*merge_srv_conf)(ngx_conf_t *cf, void *prev, void *conf);
void *(*create_loc_conf)(ngx_conf_t *cf);
char *(*merge_loc_conf)(ngx_conf_t *cf, void *prev, void *conf);
} ngx_http_module_t;
```


You can set functions you don’t need to `NULL`

, and Nginx will figure it out.

Most handlers just use the last two: a function to allocate memory for location-specific configuration (called `ngx_http_<module name>_create_loc_conf`

), and a function to set defaults and merge this configuration with any inherited configuration (called `ngx_http_<module name >_merge_loc_conf`

). The merge function is also responsible for producing an error if the configuration is invalid; these errors halt server startup.

Here’s an example module context struct:

```
static ngx_http_module_t ngx_http_circle_gif_module_ctx = {
NULL, /* preconfiguration */
NULL, /* postconfiguration */
NULL, /* create main configuration */
NULL, /* init main configuration */
NULL, /* create server configuration */
NULL, /* merge server configuration */
ngx_http_circle_gif_create_loc_conf, /* create location configuration */
ngx_http_circle_gif_merge_loc_conf /* merge location configuration */
};
```


Time to dig in deep a little bit. These configuration callbacks look quite similar across all modules and use the same parts of the Nginx API, so they’re worth knowing about.

#### 2.3.1. create_loc_conf

Here’s what a bare-bones create_loc_conf function looks like, taken from the circle_gif module I wrote (see the the source). It takes a directive struct (`ngx_conf_t`

) and returns a newly created module configuration struct (in this case `ngx_http_circle_gif_loc_conf_t`

).

```
static void *
ngx_http_circle_gif_create_loc_conf(ngx_conf_t *cf)
{
ngx_http_circle_gif_loc_conf_t *conf;
conf = ngx_pcalloc(cf->pool, sizeof(ngx_http_circle_gif_loc_conf_t));
if (conf == NULL) {
return NGX_CONF_ERROR;
}
conf->min_radius = NGX_CONF_UNSET_UINT;
conf->max_radius = NGX_CONF_UNSET_UINT;
return conf;
}
```


First thing to notice is Nginx’s memory allocation; it takes care of the `free`

’ing as long as the module uses `ngx_palloc`

(a `malloc`

wrapper) or `ngx_pcalloc`

(a `calloc`

wrapper).

The possible UNSET constants are `NGX_CONF_UNSET_UINT`

, `NGX_CONF_UNSET_PTR`

, `NGX_CONF_UNSET_SIZE`

, `NGX_CONF_UNSET_MSEC`

, and the catch-all `NGX_CONF_UNSET`

. UNSET tell the merging function that the value should be overridden.

#### 2.3.2. merge_loc_conf

Here’s the merging function used in the circle_gif module:

```
static char *
ngx_http_circle_gif_merge_loc_conf(ngx_conf_t *cf, void *parent, void *child)
{
ngx_http_circle_gif_loc_conf_t *prev = parent;
ngx_http_circle_gif_loc_conf_t *conf = child;
ngx_conf_merge_uint_value(conf->min_radius, prev->min_radius, 10);
ngx_conf_merge_uint_value(conf->max_radius, prev->max_radius, 20);
if (conf->min_radius < 1) {
ngx_conf_log_error(NGX_LOG_EMERG, cf, 0,
"min_radius must be equal or more than 1");
return NGX_CONF_ERROR;
}
if (conf->max_radius < conf->min_radius) {
ngx_conf_log_error(NGX_LOG_EMERG, cf, 0,
"max_radius must be equal or more than min_radius");
return NGX_CONF_ERROR;
}
return NGX_CONF_OK;
}
```


Notice first that Nginx provides nice merging functions for different data types (`ngx_conf_merge_<data type>_value`

); the arguments are

*this*location’s value- the value to inherit if #1 is not set
- the default if neither #1 nor #2 is set

The result is then stored in the first argument. Available merge functions include `ngx_conf_merge_size_value`

, `ngx_conf_merge_msec_value`

, and others. See core/ngx_conf_file.h for a full list.

Trivia question: How do these functions write to the first argument, since the first argument is passed in by value?

Answer: these functions are defined by the preprocessor (so they expand to a few "if" statements and assignments before reaching the compiler).

Notice also how errors are produced; the function writes something to the log file, and returns `NGX_CONF_ERROR`

. That return code halts server startup. (Since the message is logged at level `NGX_LOG_EMERG`

, the message will also go to standard out; FYI, core/ngx_log.h has a list of log levels.)

### 2.4. The Module Definition

Next we add one more layer of indirection, the `ngx_module_t`

struct. The variable is called `ngx_http_<module name>_module`

. This is where references to the context and directives go, as well as the remaining callbacks (exit thread, exit process, etc.). The module definition is sometimes used as a key to look up data associated with a particular module. The module definition usually looks like this:

```
ngx_module_t ngx_http_<module name>_module = {
NGX_MODULE_V1,
&ngx_http_<module name>_module_ctx, /* module context */
ngx_http_<module name>_commands, /* module directives */
NGX_HTTP_MODULE, /* module type */
NULL, /* init master */
NULL, /* init module */
NULL, /* init process */
NULL, /* init thread */
NULL, /* exit thread */
NULL, /* exit process */
NULL, /* exit master */
NGX_MODULE_V1_PADDING
};
```


…substituting <module name> appropriately. Modules can add callbacks for process/thread creation and death, but most modules keep things simple. (For the arguments passed to each callback, see core/ngx_http_config.h.)

### 2.5. Module Installation

The proper way to install a module depends on whether the module is a handler, filter, or load-balancer; so the details are reserved for those respective sections.

## 3. Handlers

Now we’ll put some trivial modules under the microscope to see how they work.

### 3.1. Anatomy of a Handler (Non-proxying)

Handlers typically do four things: get the location configuration, generate an appropriate response, send the header, and send the body. A handler has one argument, the request struct. A request struct has a lot of useful information about the client request, such as the request method, URI, and headers. We’ll go over these steps one by one.

#### 3.1.1. Getting the location configuration

This part’s easy. All you need to do is call `ngx_http_get_module_loc_conf`

and pass in the current request struct and the module definition. Here’s the relevant part of my circle gif handler:

```
static ngx_int_t
ngx_http_circle_gif_handler(ngx_http_request_t *r)
{
ngx_http_circle_gif_loc_conf_t *circle_gif_config;
circle_gif_config = ngx_http_get_module_loc_conf(r, ngx_http_circle_gif_module);
...
```


Now I’ve got access to all the variables that I set up in my merge function.

#### 3.1.2. Generating a response

This is the interesting part where modules actually do work.

The request struct will be helpful here, particularly these elements:

```
typedef struct {
...
/* the memory pool, used in the ngx_palloc functions */
ngx_pool_t *pool;
ngx_str_t uri;
ngx_str_t args;
ngx_http_headers_in_t headers_in;
...
} ngx_http_request_t;
```


`uri`

is the path of the request, e.g. "/query.cgi".

`args`

is the part of the request after the question mark (e.g. "name=john").

`headers_in`

has a lot of useful stuff, such as cookies and browser information, but many modules don’t need anything from it. See http/ngx_http_request.h if you’re interested.

This should be enough information to produce some useful output. The full `ngx_http_request_t`

struct can be found in http/ngx_http_request.h.

#### 3.1.3. Sending the header

The response headers live in a struct called `headers_out`

referenced by the request struct. The handler sets the ones it wants and then calls `ngx_http_send_header(r)`

. Some useful parts of `headers_out`

include:

```
typedef struct {
...
ngx_uint_t status;
size_t content_type_len;
ngx_str_t content_type;
ngx_table_elt_t *content_encoding;
off_t content_length_n;
time_t date_time;
time_t last_modified_time;
..
} ngx_http_headers_out_t;
```


(The rest can be found in http/ngx_http_request.h.)

So for example, if a module were to set the Content-Type to "image/gif", Content-Length to 100, and return a 200 OK response, this code would do the trick:

```
r->headers_out.status = NGX_HTTP_OK;
r->headers_out.content_length_n = 100;
r->headers_out.content_type.len = sizeof("image/gif") - 1;
r->headers_out.content_type.data = (u_char *) "image/gif";
ngx_http_send_header(r);
```


Most legal HTTP headers are available (somewhere) for your setting pleasure. However, some headers are a bit trickier to set than the ones you see above; for example, `content_encoding`

has type `(ngx_table_elt_t*)`

, so the module must allocate memory for it. This is done with a function called `ngx_list_push`

, which takes in an `ngx_list_t`

(similar to an array) and returns a reference to a newly created member of the list (of type `ngx_table_elt_t`

). The following code sets the Content-Encoding to "deflate" and sends the header:

```
r->headers_out.content_encoding = ngx_list_push(&r->headers_out.headers);
if (r->headers_out.content_encoding == NULL) {
return NGX_ERROR;
}
r->headers_out.content_encoding->hash = 1;
r->headers_out.content_encoding->key.len = sizeof("Content-Encoding") - 1;
r->headers_out.content_encoding->key.data = (u_char *) "Content-Encoding";
r->headers_out.content_encoding->value.len = sizeof("deflate") - 1;
r->headers_out.content_encoding->value.data = (u_char *) "deflate";
ngx_http_send_header(r);
```


This mechanism is usually used when a header can have multiple values simultaneously; it (theoretically) makes it easier for filter modules to add and delete certain values while preserving

[... 内容超长，已截断；完整原文见 source URL ...]

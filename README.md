### Prune.js

Prune.js is a JavaScript code optimizer, compressor, concatenator, source map generator with special focus on building AMD and CommonJS module trees into compact concatenated single-file packages.

Prune.js is a materialization of a particular belief: **build tools must read our minds and just do the right thing.**

Prune.js is especially useful for turning a diverse tree of AMD modules (and CommonJS modules, and AMD plugin resources) into a single compressed file. 
It auto-discovers the AMD modules tree and AMD loader configuration object (place where you set `paths` and `baseUrl` values). It turns various files, including those referred to through 'text!', 'js!', 'css!', (soon 'cjs!') AMD plugins into in-line, named AMD modules. Prune.js can deal with (detect and inline) dynamically-declared AMD resources.

Prune.js is a bit like `r.js` in purpose but it is very different in philosophy. It is designed for highest possible ease of use and sniffs out from your source all the information it needs for a successful build. Build configuration file is **not** needed. Prune.js is not tied to and does not replace any particular AMD loader. Prune.js is more general-purpose. Instead it works within the confines of the common AMD API. Prune.js is like as if you had an intelligent `automake` for JavaScript.

Oh, yes. Prune.js certainly works on Windows, as well, as on other Node.js-supporting operating systems.

### Roots

At the core of Prune.js there are:
* ESPrima - a JavaScript parser, AST tree generator
* ESMangle - an AST tree optimizer, mangler
* ESCodeGen - an AST-to-JavaScript and AST-to-SourceMap generator
* Robert Tarjan's algorythm for topological sorting of dependency trees containing circular dependencies
* A tiny bit of real magic

All of that is served as ONE javascript file ready to be ran against Node.js. (No need to pull modules. Everything is included.)

### Why

Building large JavaScript applications is a pain. There are all-inclusive and constricted-by-design kits like SproutCore where you get sane but inflexible build logic with the kit. There are free-form collections of kits where you need to hand-stitch the various JavaScript, template, css files into a build script. There is also r.js, which ties you to require.js and its ever-growing collection of obscure settings combinations and (as personally whitenessed with 1.x versions) rather fragile loading logic. 

Prune.js is an attempt to unchain an average web developer from a particular toolkit, yet still provide top-notch, elastic, auto-forming, supporting application build infrastructure.


### Build walk-through

Prune.js is supposed to be pointed at the "root" folder and the “main” module of a particular cohesive tree of source files. The "main" JavaScript file does not have to be an AMD or CJS module. Prune.js automatically detects the “require” and “define” calls used inside and traverses the dependency chain.

While support for pure CommonJS-style dependency tree is planned but not yet present, full support for AMD modules pattern is here.

If Prune.js can match an AMD reference within `require` and `define` calls to actual files and it knows how to inline (fold the contents of) this type of AMD resource, it will. Prune.js can convert static references to non-module resources like text ('text!' plugin), CSS  ('css!' plugin), plain JavaScript ('js!' plugin) (and soon CommonJS module loading ('cjs!' plugin)) into wrapped, automatically-named AMD modules and concatenates all of that together. What you get as output file is a long, properly-sorted chain of named AMD modules.

Absolute AMD references and other AMD references not resolved to local files (or in-lined named defines) are left unchanged. This means your custom AMD plugins should continue to work.

Prune.js will scan the “page root” folder and will auto-extract the AMD loader configuration object. You need to mark the configuration object with one specific property if you want this auto-detection to happen.

Because all concatenation manipulations are done in AST tree form, all code preserves references to its original filename, line and column. Source Maps generated by Prune.js, thus, contain line, column references that match exactly to the original source JavaScript files.

### How

After downloading prune.js file, put it anywhere you want. Prune.js is designed to be ran against Node.js like so:

	node ~/bin/prune.js

Prune.js needs to know the following:

1. What is the "root" of the web page? That means if your index.html is located in `~/projects/myapp/checkout` that's the "root" folder.

2. What's "main" JavaScript file. If no file is specified, it defaults to `./main.js` where the relative reference is against the "root" of the page. You can always override the default main module file name by providing it as first command-line argument to Prune.js

3. Where to put the assembly file. By default it's same as “main” file, but with “min” added to the name. If “main” file is `./main.js` then auto-picked assembly name will be `./main.min.js` You can always override the output name with -t (--target) command-line switch.

4. What Prune.js needs to do. Prune.js expects a collection of commands that it auto-sorts and runs in order. Default commands are “minify” and “amd” These mean Pruje.js will auto-detect AMD-based references in the main module, will traverse the resource tree, fold the resources as named modules into single AST tree, will optimize it, minify it and will spit it out as single, compact JavaScript file. ( Use `-c` option for specifying other commands. )

Here is how you communicate 1, 2, 3 and 4 to Prune.js:

	# Make the "root" folder your Current Working Dir
	cd ~/projects/myapp/checkout
	# run Prune.js, specifying the source, target
	node ~/bin/prune.js
	# if your main module is not './main.js', run while specifying the source
	node ~/bin/prune.js js/app/main.js

As mentioned earlier, Prune.js is capable of finding the AMD loader configuration object where you set things like `paths` and `baseUrl` for your AMD application. To make this happen you must mark the configuration object with one, specifically-named property - `AMDLoaderConfiguration` 

It is not defined now what that property value should be. What's important is that the string `AMDLoaderConfiguration` exists within the JavaScript that defines the AMD loader configuration object. Although it does not matter where that object is declared, Prune.js first looks inside the main module for such AMD loader configuration object. Then it looks in “index” files like `index.html` and `default.aspx` Then the rest of the files in “root” folder are scanned.

AMD loader config object auto-detection Example

Somewhere inside your index.html file you have a script tag that contains something like this:

	var myloaderconfig = {
		paths: {
			'app': 'js/app'
		}
		, baseUrl: './'
		, AMDLoaderConfiguration: {}
	}

After not finding references to `AMDLoaderConfiguration` inside the main module, Prune.js will scan root folder's typical “index” files and will find your `index.html` It will look at all script tag contents and will extract any first object declaration that has a property named `AMDLoaderConfiguration`.

All AMD references in main module and beyond will be filtered through `paths` and the root of the AMD namespace will be adjusted to stated `baseUrl`

Prune.js will handle (detect and inline) dynamically-used resources if you provide hints to Prune.js what possible modules this dynamic resource may be.

	// our optimizer actually finds unused code paths
	// like these and removes them before minification.
	// But, before that, Prune.js's AMD tree resolution tree
	// will see these modules and will inline them.
	if (false) {
		var we_may_need_this
		we_may_need_this = require('deep/deeper/dynamic')
		we_may_need_this = require('alt/tree/dynamic')
	}

	var prefix1 = 'deep/deeper/'
	var prefix2 = 'alt/tree/'
	var resource = 'dynamic'

	// this require will work as expected at run-time, but
	// will still benefit from inlined modules
	require(
		[( flag ? prefix1 : prefix2 ) + resource ]
		, function(dynamic){}
	)


### Limitations

Prune.js does not (yet) handle resolution of **relative** AMD resource IDs properly in absolutely all cases.

To get Prune.js to usable state sooner one particular important corner was cut:
Asynchronous-patterned require calls that use "local" require (usually one which was obtained from define by specifying the "require" dependency explicitly) do not resolve relative resources against this module's ID. instead, relative resource's module id is resolved against global AMD namespace root (baseUrl) - exactly the way "global" asynchronous require calls do. (I know, i know, lots of text, but what does it mean?! Does it affect me? Read on...)

AMD specification allows a "local" `require` to be declared inside of a define statement. That "local" require is supposed to resolve relative resources against that define's module ID, not against "global" AMD namespace root. 

Example file with path [root of AMD namespace/]app/main.js:

	// global require
	require(
		['./templates/maintemplate']
		, function(template){ /* do something */}
	)

	define(
		['./templates/maintemplate','require']
		, function(template, require /* <- !!!!!! makes require "local" */){

			// blocking local require that IS supported.
			var tmp = require('./templates/maintemplate')

			// async local require that is NOT supported properly
			require(
				['./templates/maintemplate']
				, function(template){ /* do something */}
			)
		}
	)

In the above example:

- async global require should resolve `./templates/maintemplate` to `templates/maintemplate`

- define call should resolve `../templates/maintemplate` to `app/templates/maintemplate`

- blocking (CommonJS-style) local require call should resolve `../templates/maintemplate` to `app/templates/maintemplate`

- async local require call should resolve `../templates/maintemplate` to `app/templates/maintemplate`


In the above example, Prune.js will match the outcomes per specification in all except for one case:
- async local require call will inappropriately resolve `../templates/maintemplate` to `templates/maintemplate` instead of `app/templates/maintemplate`

In other words:

Any require() call that looks like application of AMD-style asynchronous patternt (utilizing a callback) will be assumed to be "global" for purposes of resolution of relative resources. 

Relative resources inside define calls and CommonJS-style require calls are resolved properly.

Any require call that looks like applicaiton of CommonJS pattern (no callback, required arguments are NOT inside Array) will behave like "local" - resolve relative references against the module ID, not AMD namespace root. 

This Prune.js's anomaly is temporary. Implementing scoped require handling demands a gargantuan pile of extra code. Yet, this anomaly is not deemed vital or insurmountable as, based on personal experience, majority of cases where require calls are asyncronous are made in global context and the specific times require is used "locally" is usually done so in CommonJS style.

As long as you limit your use of **relative** AMD resources to the following patterns, Prune.js will work just fine for you:

	// use global require
	require(
		['./templates/maintemplate']
		, function(template){ /* do something */}
	)

	// Do not use blocking global CommontJS-style require
	// for resolution of **relative** resources:
	// var tmp = require('./templates/maintemplate')

	// Relative resources work as expected in define() calls.
	define(
		['./templates/maintemplate','require']
		, function(template, require /* <- !!!!!! makes require "local" */){

			// blocking local require that IS supported.
			var tmp = require('./templates/maintemplate')

			// do NOT use
			// async local require for resolution of **relative** resources
			// as that is NOT supported properly
			// require(
			// 	['./templates/maintemplate']
			// 	, function(template){ /* do something */}
			// )
		}
	)


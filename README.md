# Vash, the 60 billion double-dollar template-maker

Vash is an implementation of the [Razor](http://www.asp.net/webmatrix/tutorials/2-introduction-to-asp-net-web-programming-using-the-razor-syntax) templating syntax in JavaScript. A cheat sheet can be found on [Phil Haack's Blog](http://haacked.com/archive/2011/01/06/razor-syntax-quick-reference.aspx). I call this a "template-maker" because it's not a framework, and it's not a templating engine. Vash does one thing, and one thing only: turn a string template into a compiled JS function!

[![Build Status](https://secure.travis-ci.org/kirbysayshi/Vash.png?branch=master)](https://travis-ci.org/kirbysayshi/Vash)

Vash now has a ["playground"](http://codepen.io/kirbysayshi/full/IjrFw) of sorts at [CodePen.io](http://codepen.io). It uses the current version of `vash.js` from the `build` folder. Fork it to test your own template ideas!

Here's a quick example:

	// compile template
	var itemTpl = vash.compile( '<li data-description="@desc.name">This is item number: @number.</li>' );

	// generate using some data
	var out = itemTpl({ desc: { name: 'templating functions seem to breed' }, number: 2 });

	// dump it
	console.log( out )

	// outputs:
	<li data-description="templating functions seem to breed">This is item number: 2.</li>

One more:

	// assume this is in the DOM somewhere:
	<ul>
	@model.forEach(function(m){
		<li>@m.lastname, @m.firstname</li>
	})
	</ul>

	// generate
	var out = itemTpl([
		{ firstname: 'Meryl', lastname: 'Stryfe' },
		{ firstname: 'Milly', lastname: 'Thompson' }
	]);

	// outputs
	<ul>
		<li>Stryfe, Meryl</li>
		<li>Thompson, Milly</li>
	</ul>

There are many more examples in the unit tests, located in `test/vash.test.js`. Stand-alone examples are coming soon!

## Neat Stuff

* __Works in Browser or in Node__: Comes with built in Express support, and also works clientside in all browsers >= IE6 (!) and up (with the use of [ES5 Shim for the array iteration methods](https://github.com/kriskowal/es5-shim)).
* __HTML Escaping__: As of version 0.4x, Vash automatically HTML encodes values generated by an explicit or implicit expression (e.g. interpolation).
* __Speed__: Rendering with `vash.config.useWith = false` (see test/SPEED.txt, test/vash.speed.js for now) is "fast enough" `:)`.
* __No dependencies__: Vash itself has no external dependencies, aside from the ES5 array iteration methods (map, reduce, filter, etc), and [Vows](http://vowsjs.org/) for testing.
* __Complete__: Vash supports approximately 98% (this is a highly scientific estimate) of the actual Razor syntax, all the things you actually use. See below for what it doesn't support.

# BUILD

	support/tasks build
	support/tasks test

OR:

	support/tasks test

The	`test` task builds before running tests.

Vash has a few builds for various purposes, found in the [build](vash/tree/master/build) folder. There are three primary use cases for Vash:

1. Templates are compiled at runtime in the browser.
2. Templates are compiled at build time server-side, and then compressed template functions are served to the browser.
3. Templates are compiled and rendered in node, server-side, using something like [Express](http://expressjs.com/).

Given these use cases, there are a few different builds optimized:

* vash.js / vash.min.js: The full version of Vash, including all runtime helpers. Use this for scenario 1 and 3.
* vash-runtime-all.js / vash-runtime-all.min: Vash's runtime, which currently includes HTML escaping and layout/view helpers. Use this for scenario 2.
* vash-runtime.js / vash-runtime.min.js: the minimum amount of code required for a precompiled template to be able to execute client-side (currently HTML escaping). Use this for scenario 2, if you're trying to save code size and only need pure template-rendering functionality.

# USAGE

### vash.compile(templateString, [options])

Vash has one public method, `vash.compile()`. It accepts a string template, and returns a function that, when executed, returns a generated string template. The compiled function accepts one parameter, `model`, which is the object that should be used to populate the template.

`options` is an optional parameter that can be used to override the global `vash.config` for a single call to `compile`.

## OPTIONS

### vash.config.useWith = true/false(default)

If `vash.config.useWith` is set to `true`, then Vash will wrap a `with` block around the contents of the compiled function. Set to `false`, changes how you must write your templates, and is best explained with an example:

	// vash.config.useWith == true
	<li>@description</li>

vs

	// vash.config.useWith == false
	<li>@model.description</li>

Rendering is the same regardless:

	compiledTpl( { description: 'I am a banana!' } );
	// outputs:
	// <li>I'm a banana!</li>

The default is `false`.

Tech note: using a `with` block comes at a severe performance penalty (at least 25x slower!). Using `with` is mostly to support the cleanest syntax as possible.

### vash.config.htmlEscape = true(default)/false

As of version 0.4x, Vash now automatically HTML encodes values generated by an explicit or implicit expression. For example:

	// template
	<li>@model.description</li>

	// content
	For the > good!

	// vash.config.htmlEscape = true (default)
	// outputs
	<li>For the &gt; good!</li>

	// vash.config.htmlEscape = false
	// outputs
	<li>For the > good!</li>

To prevent this from happening on a single expression:

	// template
	<li>@html.raw(model.description)</li>

	// content
	For the > good!

	// outputs
	<li>For the > good!</li>

`html.raw` marks output as having been pre-encoded by wrapping it in a lightweight object that carries a `toHtmlString` method. Any object carrying
a `toHtmlString` method is automatically converted to an encoded representation using this method when output into the rendered template by Vash.

### vash.config.modelName = "model"

If `vash.config.useWith` is set to `false`, then this property is used to determine what the name of the default internal variable will be. Example:

	// vash.config.useWith == false
	<li>@model.description</li>

vs

	// vash.config.useWith == false
	// vash.config.modelName == 'whatwhat'
	<li>@whatwhat.description</li>

Again, rendering is the same regardless:

	compiledTpl( { description: 'I am a banana!' } );
	// outputs:
	// <li>I'm a banana!</li>

### vash.config.helpersName = "html"

Determines the name of the internal variable through which registered helper methods can be reached. Example:

	<li>@html.raw(model.description)</li>

vs

	// vash.config.helpersName == "help";
	<li>@help.raw(model.description)</li>

Again, rendering is the same regardless:

	compiledTpl( { description : '<strong>Raw</strong> content!' } );
	// outputs:
	// <li><strong>Raw</strong> content!</li>

### vash.config.debug = true(default)/false

Setting `vash.config.debug` to `true` will compile templates with extensive debugging information, so if an error is thrown while rendering a template (not compiling), exact location (line, character) information can be given.

This option is slightly misleading, because while `vash.config.debug` should primarily be used while developing, there are fairly negligible performance implications of running it in production.

A template with `debug` set to `true` (default):

	function anonymous(model, html) {
		try {
			html = html || vash.helpers;
			html.__vo = html.__vo || [];
			var __vo = html.__vo;
			html.model = model;
			var __vl = html.__vl = 0,__vc = html.__vc = 0;
			html.__vl = __vl = 1, html.__vc = __vc = 1;
			__vo.push(html.escape(model.someVal).toHtmlString());
			html.__vl = __vl = 1, html.__vc = __vc = 7;
		} catch(e){ html.reportError(e, __vl, __vc, "@model.someVal") }
		delete html.__vo;
		delete html.__vl
		delete html.__vc
		return __vo.join('');
	}

And that same template with `debug` set to `false`:

	function anonymous(model, html) {
		html = html || vash.helpers;
		html.__vo = html.__vo || [];
		var __vo = html.__vo;
		html.model = model;
		__vo.push(html.escape(model.someVal).toHtmlString());
		delete html.__vo;
		return __vo.join('');
	}

The primary differences are the removal of the `try/catch` block and line/character information (`__vl` and `__vc`).

### vash.config.debug[Compiler/Parser] = true/false(default)

Setting either of these to true causes Vash to output extensive debugging information (parse tree, tokens, decompiled template function) to the console, useful mostly for Vash development.

### vash.config.favorText = true/false(default)

When Vash encounters text that directly follows and opening brace of a block, it assumes that unless it encounters an HTML tag, the text is JS code. For example:

	@it.forEach(function(a){
		var b = a; // vash assumes this line is code
	})

When `vash.config.favorText` is set to `true`, Vash will instead assume that most things are non-code (markup/text) unless it's very explicit.

	@it.forEach(function(a){
		var b = a; // vash.config.favorText assumes this line is content
	})

This option is __EXPERIMENTAL__, and should be treated as such. It allows Vash to be used in a context like [Markdown](http://daringfireball.net/projects/markdown/syntax), where HTML tags, which typically help Vash tell the difference between code and content, are rare.

# CLI

As of v0.5.1, Vash now has a command-line utility. Install Vash globally:

	npm install -g vash

To access it. Once installed, you can do things like:

	// stdin, stdout
	echo '<div>@model.name</div>' | vash

	// file in, write to file in directory
	vash -f mytemplatefile.html -o tpls/

	// pass options to the vash compiler
	echo '<div>@model.name</div>' | vash -j '{ "modelName": "it", "debug": true }'

	// assign the compiled function to a namespace
	echo '<div>@model.name</div>' | vash -t JST -p mytpl
	// outputs: JST['mytpl']=anonymous(model){ ... }

And more!

# Express Support

A basic example can be found at [vash-express-example](https://github.com/kirbysayshi/vash-express-example).

# Vash as a View Engine

As of v0.5.1, Vash now offers runtime view engine support in the style of Jade's `block/append/prepend` and `extends/include`. This means that you can do:

	// in index.vash
	@html.extend('layout', function(model){

		@html.block('content', function(model){

			<h1 class="name">@model.location.name</h1>
			@html.include( 'hours', model.location.hours )
			<p class="menu"><a href="@model.location.menuLink">Menu</a></p>
			<p class="address">
				<a class='gmap' href="@model.location.gmapLink">
					<span class="street">@model.location.street</span><br />
					<span class="city">@model.location.city</span>,
					<span class="state">@model.location.state</span>
					<span class="zip">@model.location.zip</span>
				</a>
			</p>
		})

		@html.append('footer', function(){
			<p>Your Copyright Goes Here</p>
		})
	})

	// layout.vash
	<!DOCTYPE html>
	<html lang="en">
		<head>
			<meta charset="utf-8">
			<title>@model.title</title>
			<link rel="stylesheet" href="site.css" type="text/css" media="screen" charset="utf-8">
		</head>
		<body>
			<section>
				@html.block('content')
			</section>

			<footer>
				@html.block('footer')
			</footer>
		</body>
	</html>

This has been relatively untested in the browser. While it _should_ in theory work (assuming all templates are available synchronously), it has not been thoroughly tested.

There are a few limitations currently. This all occurs at runtime, which is why callbacks are used instead of straight blocks. In the future this will hopefully happen at compile time, but that adds a lot of extraneous functionality deep into the core of Vash (things like loading files from the network/file system). Because of the scoping involved, all callbacks must have `model` defined as the first parameter. In the above examples, `model` is used because that is Vash's default. If `vash.config.modelName` were set to `it`, a block would be defined using `it`:

	@html.block('content', function(it){
		<h1 class="name">@it.location.name</h1>
	})

# Errata

Since this is JavaScript and not C#, there are a few superfluous aspects of Razor that are not implemented, and most likely never will be. Here is a list of unimplemented Razor features:

### `@foreach`

This is not a JavaScript keyword, and while some code generation could take place, it's more complex than I'd like to get. However, the following is a perfect substitute:

	<ul>
	@model.forEach(function(m){
		<li>@m.lastname, @m.firstname</li>
	})
	</ul>

### `@helper`

`helper` implies that there are view lookups, auto-compiling, centralized view locations, etc, and this is bigger than Vash. However, templates can of course be called from within other templates, provided they are scoped properly, and you can do this too:

	@function specialLink(url, text){
		<a href="@url" class="special-link">@text</a>
	}

	<p>This is my @specialLink(model.url, 'SPECIAL LINK!')</p>

### `@using`

JS doesn't work this way...

### Passing templates to inline function calls

Vash cannot do this:

	@someHelperMethod('a parameter', @<a href="@url">@text</a>)

# Current Test Results

	support/task test
	······································································································
	✓ OK » 102 honored (0.182s)

# Why Vash?

The original name of this syntax is Razor, implying that it is as stripped down as possible (see [Occam's Razor](http://en.wikipedia.org/wiki/Occam's_razor)), and so a friend and I started riffing on it. Below is the stream of connected thoughts:

 	> razor...
	> precision, surgical, steel
	> tanto
	> wakizashi
		> WK.tpl()
		> japanese emoticons?
		> _$8 is a valid JS identifier
		> mootools has $$
			> double dollars
			> the 60 billion double dollar man
				> Vash the Stampede!
				> vash.tpl()
					> Very Awesome Scripted HTML
					> ...maybe just vash

# TODO

* `vash.install` to allow a file full of templates to "export" them, instead of using separate files
* Rework build system to either use a Makefile or something like Grunt
* Rewrite documentation to be more comprehensive, with syntax examples

# License

MIT

	Copyright (C) 2012 by Andrew Petersen

	Permission is hereby granted, free of charge, to any person obtaining a copy
	of this software and associated documentation files (the "Software"), to deal
	in the Software without restriction, including without limitation the rights
	to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
	copies of the Software, and to permit persons to whom the Software is
	furnished to do so, subject to the following conditions:

	The above copyright notice and this permission notice shall be included in
	all copies or substantial portions of the Software.

	THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
	IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
	FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
	AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
	LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
	OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN
	THE SOFTWARE.

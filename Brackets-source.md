#Learning from the source
###An overview of Brackets' code architecture



[Brackets](https://github.com/adobe/brackets), the new code editor for the web initiative, is a fairly ambitious project on many levels, not the least of which is that the tool itself is written using javascript, CSS and HTML, using Brackets itself when possible.

At a time when discussions around best practices for web app development have never been so lively, Brackets, which is open source, is a great opportunity for developers to witness how a complex, real life project based on web standards might be handled. And while the code editor itself is still at its infancy (Sprint 10 at the time of this writing), there are already many things we can learn from studying its current source code.

But first, a little disclaimer. While I do work for Adobe (which leads this initiative), I do not belong to the team responsible for Brackets, nor did I contribute to the source (though I did write a bunch of extensions). As a consequence, consider myself as an external observer, and as such my observations are subjective and some interpretations might even be plain wrong.



## General organization

Brackets is hosted on [github](https://github.com/), within two different repositories: [one](https://github.com/adobe/brackets) for the main source code responsible for the core web application. [The other one](https://github.com/adobe/brackets-app/) is the source of the native application shell, which makes it possible for the editor to run as a standalone application for Window and Mac OS (and before you ask: yes, Linux is on the roadmap).

The reason for this separation of concerns is quite obvious: Brackets has the potential to be run in many places outside native desktop apps. This ubiquity is the main strength of web standards, and it's what makes the project so exciting if you ask me.

So, while I spent most of my time in the first, main project, I wanted to point out that, as of today, Brackets uses the [Chromium Embedded Framework](http://code.google.com/p/chromiumembedded/) for its native application shell. This framework embeds a version of [Webkit](http://www.webkit.org/) and carries a payload of about 30 megs on each platform. I'm often asked whether or not [Cordova](http://incubator.apache.org/cordova/) (aka [Phonegap](http://phonegap.com/) will one day be able to create desktop apps, and while we still don't have any roadmap for this, I'm glad to see that Brackets has the potential to easily jump to another native shell technology, at least in theory.

In terms of file organization, the core Brackets repository has a pretty self explanatory folder structure, and is rather well commented, so I think there is no need to detail each and every one of them sequentially here. Instead, I'd rather highlight the various aspects that stroke me as a developer.



## Main code conventions

A quick look at any javascript file in the repository will answer a now classical question regarding modern web apps. Yes, Brackets javascript code is written using modules. This should be no surprise. Modules are a great way to provide a clear, limited scope, and a proper encapsulation of your javascript code by taking advantage of javascript closures. It's also a great way to provide a basic level of privacy since only the members exposed via the returned object are publicly available.

As you may know, that there are several format proposals for modules, but only two are really used today: [CommonJS Modules](http://www.commonjs.org/specs/modules/1.0/), and [AMD](https://github.com/amdjs/amdjs-api/wiki/AMD) (for *Asynchronous Module Definition*). The Brackets team has chosen a hybrid approach: it uses the CommonJS format inside AMD-compatible wrappers. This was probably done to limit dependencies to either one of the formats.

```
define(function (require, exports, module) {

	var otherModule = require("path/to/required/Module");

	var _privateStuff="I'm private";
	var publicStuff="I need to be accessed from outside";

	function doStuff(){
		console.log(publicStuff);
	}

	// Exposing public members through the exports object
	exports.doStuff=doStuff;
	exports.publicStuff= publicStuff;
	
});
```


Note that the module parameter is not used: it's only there for commonJS compliance. Also, modules are loaded via the popular script / module loader [requireJS](http://requirejs.org/). And for more informations on Modules and Module Loaders, I highly recommand reading Addy Osmani's [Writing Modular JavaScript](http://addyosmani.com/writing-modular-js/). By the way, an important thing to keep in mind is that a module, and its corresponding file, may define zero, one, or many classes. So if you're looking for a class definition, don't necessary look for the corresponding file name.

Another interesting bit you might encounter while wandering around the repository is the use of deferred / promise mechanism. This pattern is used to handle asynchronous APIs which are needed if only because of the very nature of the relationship with the native shell. If you're not familiar with this API, I'd recommend reading [the related entry in the *jQuery* docs](http://api.jquery.com/category/deferred-object/).

By the way, in case you're wondering, Brackets does not use any third party micro-architecture framework. You'll find some common concepts such as models, views and controller, but not formalized around a particular mechanism.

Finally, there is a page on the wiki describing [code conventions used for Brackets](https://github.com/adobe/brackets/wiki/Brackets-Coding-Conventions). While nothing I saw should surprise any javascript developer, I still think it's important for such an open project to agree and communicate about those practices, even if they're quite common.




## Code edition and document management 

Most of what makes Brackets a tool for actually editing code comes from the use of a great third party project: [CodeMirror]((http://codemirror.net/). Everything regarding laying out a text document, to representing code and editing it in a proper way, comes from CodeMirror, including syntax highlighting and focus management. To be more accurate, Brackets uses a [dedicated fork]((https://github.com/adobe/CodeMirror2) of [CodeMirror2](https://github.com/marijnh/CodeMirror2), but my understanding is CodeMirror committers have begun accepting pull request from the fork.

Document and project management, however, is handled separately. This is the logic responsible for things like defining a project, a working set, selecting a file and keeping track of open files inside that project. While this may seem trivial when put this way, this is actually a very central part of any software which job is to edit document. 

Several classes are involved in this process, such as `Document`, `Editor` and their respective managers, and understanding their relationship is essential to understand Brackets architecture.

While a *Document* represents a file and its content, an *Editor* exposes methods to edit this document. *Documents* and *Editors* have a one to many relationship (one *Document* may be edited by several *Editors*). You can see the `Document` as a model, and the `Editor` as a middle-man between a view and the `Document` object it represents (some might call that a controller, even if that's not technically correct).

The `DocumentManager` and the `EditorManager` classes create, delete, and give a central access for all documents and editors, respectively. For instance, to access the current document, you can ask the *DocumentManager*:

```var currentDoc = DocumentManager.getCurrentDocument();```

But since *Editors* keep track of the document they edit, and the *EditorManager* manages all editors, you could also go this way:

```var currentDoc = EditorManager.getFocusedEditor().document;```

Note that the current *Editor*, and its corresponding *Document* do not necessary correspond to the currently opened tab in the UI, because Brackets has a notion of inline editors which can come on top of the main, full size editor representing the currently opened document.

Editors give complete access to write the content and selection of a document, abstracting the CodeMirror implementation. The underlying code mirror API can still be accessed directly through ```_codeMirror```, like when you use ```_codeMirror.replaceSelection(newString);```. You sometimes need to do that simply because the Editor API only covers a limited part of the CoreMirror features. In any case, if you want to get into Brackets, get prepared to spend some time in the CodeMirror API documentation.

But, as a model, the `Document` API also exposes text editing operations, such as `setText()` or `replaceRange()`, which are delegated to its master editor (which itself delegates it to CodeMirror). So, to edit a *Document*, you can simply use its own API, regardless of editors.

```currentDoc.setText("Hello world");```

Selection management, however, is related to a corresponding view so it's only available through the `Editor` object, using methods like `getSelectedText()` and `setSelection(start, end)`. Here's how you could set the selection of the document which currently has focus:

```EditorManager.getFocusedEditor().setSelection({line:0, ch:0}, {line:0, ch:5});```

Finally, a word of warning: if you need to keep a reference of a `Document` object, you have to call ```Document.addRef()``` for reference counting. This is a workaround to circumvent the lack of weak references in javascript.



## Live development and inline Editors

One of things that make Brackets unique today is its so-called *Live Development* feature: Brackets is "connected" to the browser is such a way that modifications you do while you type are immediately shown in the browser. Needless to say, this is potentially a killer feature in terms of productivity.

At the heart of the implementation of Live Development is the `Inspector.js` file, which is in charge of the communication with Chrome's remote debugger API using JSON over WebSockets. Other elements involved in this process include Agents - which keep track of changes and propagate them- and Documents which, again, represent CSS, JS and HTML files on the system.

The *Inline Editor* feature, which makes it possible to overlay some content over code, is implemented through the `InlineWidget` class (the very basis of inline editors, which is just a panel floating over your code). The `InlineTextEditor` class is slightly more sophisticated: it inherits from `InlineWidget` and adds a simple text editor.

The class you'd use to create an inline editor which contains a small selectable panel on the right, such as Brackets' default CSS inline editor, is the `MultiRangeInlineEditor`, which itself inherits from `InlineTextEditor`.

To use such an *Inline Editor*, you have to register a provider function with the `EditorManager`, like this:
```
EditorManager.registerInlineEditProvider(providerFunction);
```

This function has to take two parameters: the host editor, and a position object. Position objects are used by *CodeMirror*, and contain two properties, the line number and the character position. Inside this function, you'll simply instantiate your `MultiRangeInlineEditor`, and pass it the corresponding `Editor` instance.

```
function providerFunction(hostEditor, pos){
	(…)
    var myInlineEditor = new MultiRangeInlineEditor(data);
    myInlineEditor.load(hostEditor);
}
```



## Views and user input management

I will not dive that much into styling, widgets, layout, and other UI details here, but I wanted to point out that, besides *jQuery*, Brackets makes use of the über popular Twitter Bootstrap framework for its elegant UI. As a consequence, it also uses the [LESS CSS pre-processor](http://lesscss.org/).

Surprisingly, Brackets does not seem to use any templating system. Most of the UI is "hard coded" in the main brackets HTML file. My understanding is that this is supposed to change in the future.

More important to understand is how Brackets handle user inputs. To do so, it makes use of the *Command* pattern. This pattern describes how to encapsulate actions to be performed by the software (e.g. *Open File*) as objects (the Command). It is very convenient to map user gestures and inputs to those such objects, for many reasons. 

First, it allows to decouple user interactions (whether it comes from menus, keyboard, etc) from its consequence in the editor, which gives a much more maintainable code.

Second, since actions are treated like objects, you can perform all sorts of operations on them. For instance, they can be pushed into an array. This is a popular pattern for implementing the basics of things such as history management and tool scripting, so you can expect those features in the future. And of course, it makes the code much easier to maintain, and to test.

In terms of implementation, registering a Command is a simple as that:
```
CommandManager.register(menuName, commandID, callback);
```
It will then be triggered when the corresponding menu is selected, but you can also trigger it manually:
```
CommandManager.execute(commandID);
```



## Quality

Code quality is absolutely essential for any serious project, and of course Brackets is no exception. 

It uses [JSLint](http://www.jslint.com/) to evaluate the quality of its code. JSLint is one of the most popular Javascript static analysis tool. It was made by *Douglas Crockford* as a way to easily scan your code and report any trouble, as described by its comment based configuration mechanism. 

You'll find JSLint configuration directives on top of every module:

```
/*jslint vars: true, plusplus: true, devel: true, nomen: true, indent: 4, forin: true, maxerr: 50, regexp: true */
/*global define, $, FileError, brackets, window */
```

Note that JSLint is also used by default in the software itself for providing developers an immediate feedback on the quality of their code.

But Brackets also uses a testing framework, [Jasmine](http://pivotal.github.com/jasmine/), to better guarantee the quality of the code. Jasmine is a popular Behavior Driven Development (a flavor of *Test Driven Development*) framework used for JS development.

A test runner has been included in the editor itself, so that it can easily be tested... from itself. 



## Early Extensibility Model

An early extensibility mechanism is already described, allowing us to easily extend Brackets functionalities without having to become an official contributor. The typical workflow for doing so would be to launch a second instance of Brackets for testing the actual code, using the `Debug > Experimental > New Window` command.

The extension itself is of course described as a module, but in a somewhat separate scope from the rest of the source code. To load dependencies from Brackets, you have to use `bracket.getModules()`.

Here's a complete [Hello World extension](https://github.com/adobe/brackets/wiki/Simple-%22Hello-World%22-extension) originally written by *Mike Chambers*.

```
/*jslint vars: true, plusplus: true, devel: true, nomen: true, regexp: true, indent: 4, maxerr: 50 */
/*global define, $, brackets, window */

/** Simple extension that adds a "File > Hello World" menu item */
define(function (require, exports, module) {
    'use strict';

    var CommandManager = brackets.getModule("command/CommandManager"),
        Menus          = brackets.getModule("command/Menus");

    
    // Function to run when the menu item is clicked
    function handleHelloWorld() {
        window.alert("Hello, world!");
    }
    
    
    // First, register a command - a UI-less object associating an id to a handler
    var MY_COMMAND_ID = "helloworld.sayhello";
    CommandManager.register("Hello World", MY_COMMAND_ID, handleHelloWorld);

    // Then create a menu item bound to the command
    // The label of the menu item is the name we gave the command (see above)
    var menu = Menus.getMenu(Menus.AppMenuBar.FILE_MENU);
    menu.addMenuItem(MY_COMMAND_ID);
    
    // We could also add a key binding at the same time:
    //menu.addMenuItem(MY_COMMAND_ID, "Ctrl-Alt-H");
    // (Note: "Ctrl" is automatically mapped to "Cmd" on Mac)
    
    // Or you can add a key binding without having to create a menu item:
    //KeyBindingManager.addBinding(MY_COMMAND_ID, "Ctrl-Alt-H");

    // For dynamic menu item labels, you can change the command name at any time:
    //CommandManager.get(MY_COMMAND_ID).setName("Goodbye!");
});
```

What's great is that your extension can take advantage of the pretty much all of Brackets features, like `InlineEditor` mechanism.



## Wrap up

That's pretty much what I wanted to tell you about Brackets for now, folks. I hope that this somewhat high level overview will make you want to know more about this exciting project, and hopefully help you get into Brackets quicker.

For a better, deeper understanding of how Brackets works, I recommend taking a look at its [wiki](https://github.com/adobe/brackets/wiki) and the various videos on youtube shot at the [first Brackets hackathon](http://www.youtube.com/watch?v=xm9kSWZyawg&list=PL69EAABFC3B569526&index=2&feature=plpp_video).



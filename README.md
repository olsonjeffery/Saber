## Saber: A web framework for Newspeak

Saber is a web application framework, built atop Newspeak (specifically the Newspeak 2 dialect). It leverages some useful features of the language and platform to deliver a sensical, hierarchal pattern for structured web applications and their exposed endpoints. It licensed under the terms of the Apache License, version 2.0. A copy of the license may be found [here](http://www.apache.org/licenses/LICENSE-2.0.html) and is also included with the source. Saber is kept under source control on github at [http://github.com/olsonjeffery/saber](http://github.com/olsonjeffery/saber).

### Introduction

Saber was initially created as an exercise to learn more about the Newspeak environment, but also as a demonstration of a viable alternative to other, prevelant approaches to the same problem on other platforms. What follows is a brief treatise on the utility of this demonstration.

The concept of Controllers and Actions as they are often found in modern web frameworks are an imperfect abstraction that doesn't scale well in larger web applications, in the author's opinion. As the state-of-the-art in web development moves forward, more and more 'RESTful' style routing is becoming more popular (since it tends have a much lower barrier to entry then SOAP/RPC for creation and integration with client-side toolkits like JQuery). But typically, the artificial Controller/Action abstraction remains, with explicit routing schemes overlaid that map requests to individual Actions.

Saber is an exploration of a possible response to this notion.

### Getting started with Saber

0. Learn the ropes with Newspeak. Explaining the 'image' development format, language/environment semantics, etc. is beyond the scope and responsibility of this document. The basic [download](http://newspeaklanguage.org/the-newspeak-programming-language/downloads/) for Newspeak includes a 'Newspeak 101' document that does a good job of helper a newcomer get their feet wet.
1. Read the rest of this document to get an idea for Saber's approach to web applications.
2. Obtain Saber by either:
  1. Cloning the [source repository](http://github.com/olsonjeffery/saber.git) and compiling all of the `.ns2` source files into your existing image (using the 'Compile File' option in the *operate menu* (the rightmost button in the menu at the top of a Newspeak window)).
  2. Downloading and opening the saber.image file (after installing the Newspeak VM). The image can be downloaded [here](http://github.com/olsonjeffery/Saber/downloads).
3. Open a new workspace and evaluate `Saber withSite: (SaberExampleSite usingPlatform: Platform new) start`. This will spawn a new instance of Saber running `SaberExampleSite` on port 8083, which is a basic example of how to build a Saber application. Its implementation covers many of the features of Saber.
4. Open up SaberExampleSite in the IDE and examine how it works.

### Routing and request handling

Saber's response to the above pattern of web application development is at the core of its structure. It looks like a hiearchy of nested classes, 'falling out' from a single SiteRoot class. Each class maps directly to a 'node' in an HTTP request url that it will match upon. These will be referred to as Handlers for the rest of the document. Essentially: *Instead of artificially aggregating endpoints into a Controller/Action hierarchy, Saber let's the structure of the routing handlers becoming the organizing taxonomy for the application*.

#### The Saber Way

But what does this look like in practice? An example:

    + SiteRoot
      - foo
        - baz
      - bar
  
In the above block, we have an example hierarchy of classes. All of the classes, including the SiteRoot, subclass `SaberHandler`. An HTTP request for `/` would match on the `SiteRoot` Handler. A request for `/foo` would match on the `foo` Handler, a request for `/foo/baz` would match on the `baz` Handler, and so on. A request that didn't map to one of these requests would return a 404 (or be handled custom 'Not found' Handler, if one is defined and nested with the SiteRoot).

#### HTTP Methods in Saber

Each `SaberHandler` or similar handler class (there are a few others that will be discussed in this document) must provide an implementation for one or more methods, each corresponding to an HTTP method. Currently, they are:

* onGet: for GET requests
* onPost: for POST requests
* onPut: for PUT requests
* onDelete: for DELETE requests

Any of the methods that do not have an implementation provided will return a 404/'Not Found' response.

#### 'RESTful' style parameters in routes

Saber supports embedding parameter information directly in routes, instead of confining it solely to query strings appended to the end of a request url. It works as follows:

A handler is nested within the SiteRoot hiearchy subclassing SaberParameterHandler. An example:

    + SiteRoot
      - foo
        - baz
        - nameParam << subclasses SaberParameterHandler instead of SaberHandler as is the case with the other classes
      - bar

If a request in a given node does not match on any other handlers, then the parameter Handler is matched and the *contents of the node* are stored as a field in the query with *name of the handler as the key*. In the above example, a request for `/foo/baz` would match the `baz` Handler. But a request for `/foo/johnny` would match the `nameParam` handler. Furthermore, the query value `johnny` would be available in the request with the key `nameParam`. Otherwise, a handler subclassing `SaberParameterHandler` behaves exactly as a normal `SaberHandler` and can have handlers nested within it. It is important to note that a parameter handler acts as a 'catch-all' if no other handler at the same node in the hierarchy is matched. As such a given Handler node can only have a single `SaberParameterHandler` nested within it. If more than one `SaberParameterHandler` is nested within a given Handler, then an error will occur at request-time.

#### Dealing with requests for non-existant routes

If a SiteRoot has a class nested within is that subclasses `SaberNotFoundHandler`, that handler will be evaluated whenever a request is made to the application that doesn't match on any other route. For example:

    + SiteRoot
      - foo
        - baz
        - nameParam 
      - bar
      - notFound << subclasses SaberNotFoundHandler

If a request were made for `/bar/42`, this wouldn't match on any existing route. Therefore, the `notFound` handler would be processed for this request. As with other Saber handlers, you must provide implementations of the onGet:, onPost:, onPut: and onDelete: methods to actually respond to these requests, otherwise a default 404 response is returned to the client. A given SiteRoot can only define a single `SaberNotFoundHandler` and the Handler *must* be nested within the SiteRoot or it will not be picked up at request-time.

#### Serving static files from Saber

As web applications move towards richer and richer client-side experiences, it becomes very important that applications are able to service static content such as JavaScript scripts, CSS stylesheets, etc. Saber enables this by making the `SaberStaticFileHandler` available. Classes that are nested within SiteRoot that subclass this class will be processed at start-up time and requests that match on them will be mapped to a file directory that you define (by providing an implementation for the `documentRoot` selector). For example:


    + SiteRoot
      - foo
        - baz
        - nameParam 
      - bar
      - notFound << subclasses SaberNotFoundHandler
      - static << subclasses SaberStaticFileHandler and implements 'documentRoot' as returning '/var/www/some/static/files', 'C:\static\files', etc

In this case, we have a class named `static` that has its `documentRoot` defined to return some arbitrary path. When a request is made that includes the site prefix 'static' (such as `/static/style.css`), this request would be mapped to a file request for '/var/www/some/static/files/style.css' on the filesystem and the file is returned. Unlike the other Saber handlers, `SaberStaticFileHandler` does not use implementations of methods that respond to HTTP request methods and classes that subclass `SaberStaticFileHandler` *must* provide an implementation of `documentRoot`.

### Views in Saber

Saber currently supports a minimalistic view rendering scheme based loosely upon the idea of 'view inheritance' found in Django's template system.

Any class that inherits from `SaberView` can be used as a view. It must have its initializer selector set to `platform:parent:` and call this initializer on `SaberView` (examine the classes nested within the `Views` class in `SaberExampleSite` to see what this looks like).

Besides this, a view class only needs to implement the `content` method, which returns a string representing a view's body. Currently, the `SaberView` scheme only supports two rendering extensions: Value substitution and block inheritnace.

#### Value substitution

When creating an instance of a view, it has a Dictionary within it called `model`. You can add values to the dictionary and have them be accessible to Saber at request-time to insert into the view.

For example, within a view we would have:

    content = (
      ^ '
      <html>
        <head>
          <title>{ title }</head>
        </head>
        <body>
          Hello world!
        </body>
      </html>
      '
    )

This is an example of a `content` method that returns a string representing a view's response. The item of note, here, is the `{ title }` text within the `<title>` tag in the document. This is a value that will be looked view in the view's `model` at render-time and substituted with whatever is there. If the `title` value isn't insert into the model, an error will result.

Within a handler that uses this view we have a method that handlers the request:

    onGet: request = (
      | view |
      view:: SomeView platform: platform parent: nil "this will be explained further down".
      view model at: 'title' put: 'Hello world handler'.
      ^ HttpResponse fromString: view render
    )

This would allow the value inserted into the model to be substituted in the view's output markup. Please note that only strings are supported for being resolved in the view's output, currently.

#### View inheritance

Views support being nested within each other compositionally at the time of their instantiation. That is, a view's parent is not explicitly known until the time a `SaberView` instance is created. For example:

    getView = (
    | parentView |
    parentView:: SomeParentView platform: platform parent: nil.
    ^ SomeChildView platform: platform parent: parentView.
    )

This would establish a parent-child relationship between the two views in this example and would enable block inheritance/substution of the child view on top of the parent view.

Blocks (not to be confused with Newspeak blocks) are regions of markup that are specified/called out in a parent view as being available for "overriding" by a child view.

In the parent view's `content` method:

    content = (
      ^ '
      <html>
        <head>
          <title><% block title %>template title<% endblock %></title>
        </head>
        <body>
          <% block content %>Hello, world!<% endblock %>
        </body>
      </html>
      '
    )

In this example, the parent view specifies two blocks: `title` and `content` (using `<% block name %>` format to open a block and `<% endblock %>` to close it). Basically, the view is saying: "Hey, I have these two blocks that I expect to be overridden in any view that inherits from me."

So, in the child view's content method:

    content = (
      ^ '
      <% block title %>child view<% endblock %>
      <% block content %>Hello, {username}!<% endblock %>
      '
    )

Here, the child view just specifies which blocks it wants to override and the new content for them. This overriding behavior can 'cascade' over any number of inherited views. If a child does not specify an overridden version of a block specified in a parent, the parent's original content will be rendered. This allows child views to only have to re-specify the parts of the UI that different from their parent, making site theming/styling trivial.


Currently, these are the only features of views in Saber. Looping/conditionals/etc can be acheived using language features.

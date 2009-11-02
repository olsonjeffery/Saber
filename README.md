## Saber: A web framework for Newspeak

Saber is a web application framework, built atop Newspeak (specifically the Newspeak 2 dialect). It leverages some useful features of the language and platform to deliver a sensical, hierarchal pattern for structured web applications and their exposed endpoints. It licensed under the terms of the Apache License, version 2.0. A copy of the license may be found [here](http://www.apache.org/licenses/LICENSE-2.0.html) and is also included with the source. Saber is kept under source control on github at [http://github.com/olsonjeffery/saber](http://github.com/olsonjeffery/saber).

### Introduction and feature overview

Saber is an example of a framework that was written from the ground-up to harness some of the benefits and features of its host platform. It was created as an exercise to learn more about the Newspeak environment, but also as a demonstration of a viable alternative to other, prevelant approaches to the same problem on other platforms. What follows is a brief treatise on the utility of this demonstration.

The abstraction of Controllers and Actions as they are often found in modern web frameworks are an imperfect abstraction that weren't ever intended to scale onto the world-wide web in the form of endpoints on a web server. As the state-of-the-art in web development moves forward, more and more 'RESTful' style routing is becoming more popular (since it tends have a much lower barrier to entry then SOAP/RPC for creation and integration with client-side toolkits like JQuery). But typically, the artificial Controller/Action abstraction remains, with explicit routing schemes overlaid that map requests to individual Actions.

#### The Saber Way

Saber's response to this pattern of web application development is at the core of its structure. It looks like a hiearchy of nested classes, 'falling out' from a single SiteRoot class. Each class maps directly to a 'node' in an HTTP request url that it will match upon. These will be referred to as Handlers for the rest of the document. For Example:

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

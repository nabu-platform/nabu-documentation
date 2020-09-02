@icon web
@image /resources/images/web.png
@description Glue can serve as a web framework, a preprocessor, a postprocessor, a complex css parser,...

# Web Framework

The glue web framework adds a number of methods to manipulate http requests and responses and a number of ways to get data into and out of a web page.

There are three parts:

- :glue listener: an event driven listener that works on the nabu http server
- :glue preprocess: a preprocessor where the "request" is the original request and the "response" is the rewritten request
- :glue postprocess: a post processor where the "request" is the original response and the "response" is the rewritten response

## User Data

You can insert user data from various sources:

```python
@get
myVar ?= null

@post fieldName
myOtherVar ?= null
```

If nothing is passed in with the annotation, the variable name is assumed to match the field name in the source. If this is not the case you can give the original field name after the annotation. You can inject data using:

- **@get**
- **@post**: for file submits, the name of the field will return the byte content. To get metadata about the file itself, you can use: 
	- @post myFile:fileName
	- @post myFile:contentType
- **@cookie**
- **@header**: some headers are preparsed (unless you disable preparsing), for example the "If-Modified-Since" header is a date object, not the original string. You can always access the original headers using the method `request.header()`
- **@session**
- **@path**: check REST support
- **@meta**: this injects metadata about the request, possible values are:
	- contentType
	- contentLength
	- charset
	- contentRange
	- name
	- method
	- target
	- url
- **@content**: if no typing is given for the variable, you will get the actual byte stream of the content. However if a type was set, the data will be unmarshalled if it was originally JSON or XML into the given object. Note that you can manually parse the data as well (if you have the respective glue-json or glue-xml plugin):

```python
@content
theBytes ?= null

content = json.objectify(theBytes)
```

Note that data is only injected right before the line is executed so data for lines that are not executed (due to conditional circumstances) is never injected.
These annotations have method equivalents if you want more control.

## Validation

You can add validation to any variable assignment, including user input. These are the possible validations:

- **@pattern**: you can add a regex pattern that the variable should match, there are a few predefined patterns you can use, e.g. **@pattern word** will check that it matches "[\w]+", **whitespace** checks for any whitespace and **number** checks for valid integer or decimal numbers
- **@enumeration**: you can check that a value belongs to a given enumeration. The enumeration can be given in a comma-separated fashion, e.g. **@enumeration a, b, c** will check that the value is either "a", "b" or "c" (the values are auto-trimmed). Note that there are also a few predefined enumerations:
	- **language** will check that it is a valid ISO language code
	- **country** will check that it is a valid ISO country code
	- **timezone**: will check that it is a valid timezone
	- **charset**: will check that it is a valid charset
- **@null**: depending on the (optional) boolean (default true) it will check that the value _is_ null (if true) or not null (if false), so **@null false** will check that the value is not null
- **@max**: for numbers it will check that the number is not bigger than the given max value, for strings it will check that the string is not longer than the given amount. You can use the optional typing system to make sure the variable is of the correct type for the matching, e.g. this will check that the query parameter is not null, an integer and not larger than 10:

```python
@get
@max 10
@null false
integer myValue ?= null 
```  

- **@min**: sets a minimum for numbers or a minimum length for strings


# Response Data

There are two ways to set the response data:

- **response.content()**: you can use this method to set the response content. This content can be a stream, bytes, string, object,... If an object, it will use (if allowed) the `Accept` header from the client to determine whether he would prefer the data as JSON or XML and provide it as such. This method will also set `Content-Type` (if possible) and `Content-Length` (again if possible) headers.
- **echo()**: anything you echo is basically sent back as response to the client **unless** response.content() is used.

# Permission Handling

Assuming you use the provided methods for authentication, you can do this:

```python
@role guest
doSomething()

@role user
doSomethingElse()

@permission toDoStuff
sequence
	doTheStuff()
	notifyThatTheStuffIsDone()
```

The respective lines will only be executed if the user has the required roles/permissions. If multiple roles are provided, the user must have **one** of the listed roles, not **all**.

These annotations also have method equivalents. 

# REST Support

There is native rest support which you can use by setting an annotation at the script level:

```python
@path {id}/view

@path
id ?= null
``` 

If your script is called `users.glue` for example, the following URL would work on the server: `http://example.com/users/1234/view`. This behavior can be turned off in the listener.
The syntax for the variables follows that of the JAX-RS API, which means you can specify a regex to match: `{id: [0-9]+}/view` would make sure the id was numeric.

Using the `@content` annotation described above you can easily get the parsed data from the request. 

You can use the above described `response.content()` method to set an object as response.

# CSRF Support

Unless specifically disabled in the listener, it will automatically add a CSRF token to any outgoing form and check it on any incoming form.

# Available Methods

The methods are divided by their domain. Unlike most glue methods they do not rely on their uniqueness in naming which means the coding guideline is to use the full name always.

- Request methods
	- **request.content()**: get the full content (the actual HTTPEntity object)
	- **request.header(name)**: get the first header object for the given name
	- **request.headers(name)**: get all header objects for the given name
	- **request.method()**: returns the method of the requests
	- **request.target()**: returns the target of the request
	- **request.version()**: returns the version of the request
	- There are a few methods that are structured the same for different data sources:1
		- **request.cookies()**: get a full map of all the cookies
		- **request.cookies(name)**: get a list of all the values for a specific cookie
		- **request.cookie(name)**: get the value for a specific cookie (if there are multiple, the first)
		- **request.gets()**
		- **request.gets(name)**
		- **request.get(name)**
		- **request.posts()**
		- **request.posts(name)**
		- **request.post(name)**
		- **request.paths()**
		- **request.paths(name)**
		- **request.path(name)**
- Response methods
	- **response.header(key, value, removeExisting)**: Adds a header to the response with the given key and value (and optionally deletes the current headers with that name). If you have comments, put them in the value as well. Note that if you leave out the value, it will simply remove the headers with that name.
	- **response.code(number)**: sets the response code
	- **response.content(content, contentType)**: set the given content and (optionally) the content type for it. If no content type is given, it will be a best effort guess.
	- **response.cookie(key, value, expires, path, domain, secure, httpOnly)**: set a cookie in the response. Note that as for any method, all parameters are optional and most have sensible defaults.
	- **response.charset(name)**: set a specific charset for the response
	- **response.redirect(location, isPermanent)**: redirect the user
	- **response.notModified()**: send back a response that nothing was modified
	- **response.cache(maxAge, revalidate, private)**: directly set the cache header
	- **response.method(method)**: can be used to update the method
	- **response.target(target)**: can be used to update the target
- Session methods
	- **session.get(key)**: get the value for the given key
	- **session.set(key, value)**: set the value for the given key
	- **session.destroy()**: destroys the current session
	- **session.exists()**: checks if there is a session
	- **session.create(shouldCopy)**: creates a new session. If the shouldCopy is set to true (default false), all values of the current session (if any) are copied into the new one
- User methods
	- **user.authenticate(name, password, shouldRemember)**: authenticate the user with the given name and password. The shouldRemember boolean (default true) indicates whether or not a secret should be used to remember the user outside of the session. The availability of the remember option is dependent on the authenticator that is used however as not all authenticators support secrets. **Important**: successful authentication is automatically followed by generation of a new session to prevent fixation.
	- **user.remember()**: tries to remember the user based on a shared secret. This is again dependent on the authenticator.
	- **user.authenticated()**: checks if the user is authenticated
	- **user.hasRoles(roles)**: checks if the user has **all** of the roles listed
	- **user.hasPermission(context, action)**: checks if the user has the given permission
	- **user.salt()**: generates a new salt based on a type 4 UUID
- Server methods
	- **server.root()**: returns the root path of your current application
	- **server.fail(message, code)**: a HTTP error is created for the given code (500 by default) with the given message
	- **server.abort()**: stop the script from running without failure. Any response set up to that point is returned.
	- There are some log methods at different levels that use slf4j in the background
		- **server.debug(message)**
		- **server.info(message)**
		- **server.warn(message)**
		- **server.error(message)**

# Pre- and Postprocessing

There are quite a few neat preprocessing features available in most http servers, for example apache has `mod_rewrite` to rewrite request targets and methods, `mod_headers` to rewrite headers and `mod_substitute` to rewrite some content.

My biggest gripe with these modules is the new -and rather complex- syntax you have to adhere to to make it work and you are still somewhat limited in how complex your instructions can be.

In the postprocessing department you can usually set error pages based on for example error codes but this is again custom syntax in some config file and usually limited to error codes.

This library provides a glue-based preprocessor and a glue-based postprocessor.

They function more or less the same as the actual glue listener except where the glue listener has an incoming `HTTPRequest` and an outgoing `HTTPResponse`, the preprocessor has an incoming `HTTPRequest` and outgoing `HTTPRequest` as well while the postprocessor has an incoming `HTTPResponse` and also an outgoing `HTTPResponse`.

The existing methods for the glue listener are reused but mapped the "request" methods to the incoming http entity and the "response" methods to the outgoing http entity.

A lot of the same functionality is available in all three scenarios but there are of course some differences. For example the `response.method()` was added solely for preprocessing (as you can not set a method on the response object), sessions are available in pre- and postprocessing but should be used for read-only purposes etc.

**Important**: it is critical that you register the preprocessor as one of the first listeners on the eventing queue, otherwise it might be ignored for some requests. Likewise it is important to register the post processor at an appropriate level depending on what else you have on the pipeline.

## Preprocessing

For example suppose you want to update the method and target of a request:

```python
@meta
target ?= null
# Redirect a request on the root
if (target == "/")
	response.target("somethingElse/entirely")
	response.method("POST")
```

Or for example add some content to the request:

```python
@meta
target ?= null
@meta
method ?= null

if (target == "/" && method == "get")
	response.content(structure(test1: "some stuff here", test2: 1), "application/json")
```

Or rewrite the incoming request, for example in this case we take an incoming JSON structure, change the value of a field in it and transform it to XML:

```python
@meta
target ?= null
@meta
method ?= null
@meta
contentType ?= null
if (target == "/" && method == "post" && contentType == "application/json")
	@content
	myStructure content ?= null
	response.content(structure(content, test1: "something else entirely!"), "application/xml")
```

The incoming request is:

```javascript
{"test1": "test", "test2": 5}
```

The rewritten request becomes:

```xml
<anonymous>
	<test2>5</test2>
	<test1>something else entirely!</test1>
</anonymous>
```

## Postprocessing

You can rewrite error responses, for example let's redirect a 404 to another page:

```python
@meta
code ?= null
if (code == 404)
	redirect("/404.html")
```

During testing I did not actually have a 404.html available which in this case will trigger an infinite loop of the browser requesting a page that triggers a 404 that redirects to a non-existing 404.html page ad nauseam.

Even though you are in postprocess mode and both request and response are "HTTPResponse" entities, you can (normally) still access the original HTTPRequest because it is generally included in the HTTPResponse, so you could try:

```python
@meta
code ?= null
@meta
target ?= null
if (code == 404 && target != null && target != "/404.html")
	redirect("/404.html")
```

Note that the null check is used to make sure we actually have the original request target. Some rogue component could be generating responses without the original request attached.

# CSS Generation

A lot of people use css preprocessing to generate css using methodologies that css itself does not support. Scss is a good example of an extension on top of CSS that allows for more complex stuff like variables, mixins,...

While using scss is always still an option, this package ships with an alternative css "preprocessor" based on glue. It is not really a preprocessor as much as it is simply a script that -when run- outputs valid css.

You of course have access to all the niceties of glue in a webcontext:

- variables
- lambda's
- contextual access (get, post, session, cookies,...)
- ...

But it also offers some features to rival scss functionality, especially with regards to nesting.

**Important**: to activate the css preprocessor, you **have** to set the css annotation at the script level, for example:

```python
@css

@element div
sequence
	padding("10px", top: "20px")
```

These are the annotations that are supported for the css preprocessing:

- **@css**: necessary at the script level to indicate that this is a css stylesheet. This will activate the css preprocessor and make sure the content type is set correctly
- **@class**: allows you to target a class e.g. ``@class myClass``
- **@element**: allows you to target an element e.g. ``@element div``
- **@id**: allows you to target an id e.g. ``@id myId``
- **@attribute**: allows you to target an attribute e.g. ``@attribute title=test``
- **@state**: allows you to target a state (first-child, hover,...), e.g. ``@state first-child``
- **@select**: allows you to write full css e.g. ``@select .myClass``
- **@relation**: the default relation is descendant (e.g. ``.myClass .myDescendantClass``) but you can also set:
	- **child**: e.g. ``.myClass > .myChildClass``
	- **sibling**: e.g. ``.myClass ~ .mySiblingClass``
	- **adjacent**: e.g. ``.myClass + .myAdjacentClass``
	- **self**: this allows you to build upon a single identifier, for example:

```python
@element div
sequence
	padding("10px", top: "20px")
	@class special
	@relation self
	sequence
		margin("10px")
```

Generates:

```html
div {
	padding: 10px;
	padding-top: 20px;
}
	
div.special {
	margin: 10px;
}
```

The code will generate cross products when needed:

```python
@element div, span
sequence
	padding("10px", top: "20px")
	@class test, test2
	@relation self
	sequence
		margin("10px")
```

Will generate:

```html
div, span {
	padding: 10px;
	padding-top: 20px;
}
	
div.test, div.test2, span.test, span.test2 {
	margin: 10px;
}
```

For those not entirely convinced of the glue syntax (this generally includes most people familiar with css and/or scss), there is an alternative parser that rewrites a scss-like syntax to the glue css annotations as described in the above.

# GCSS

If you have a page that ends with the extension ``.gcss`` you can use a syntax that resembles scss. It no longer uses whitespace to delineate blocks but instead old-fashioned brackets.

Apart from the css like syntax, you still have full access to the other features of glue, you can still use lambdas, variables,....

```html
.navigation {
	width: 100%
	height: ${navigationHeight}
	position: fixed
	top: 0px
	border: solid 1px ${borderColor}
	border-style: none none solid none
	color: ${lightTextColor}
	background-color: #000000
	
	a {
		background-color: rgba(255,255,255,0.2)
		color: #FAFAFA;
		text-decoration: none;
		display: inline-block;
		width: 150px;
		text-align: center;
		vertical-align: top;
		padding-top: 25px;
		height: 100%;
		margin-left: 10px;
		border: solid 1px ${borderColor};
		border-style: none solid
		
		&:hover {
			background-color: rgba(255,255,255,0.4);
		}
	}
}
```

There are a number of selectors:

- comment: uses general code-style comments meaning "//"
- by id: ``#id1, id2``
- by attribute: ``[myattr=value]``
- by class: ``.class1, class2``
- by state: ``:before, after``
- by element: ``div, a``
- append: ``&``

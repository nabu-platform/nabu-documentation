# Introduction

Glue is a scripting language that is based on the same engine that drives the blox orchestration engine. It integrates completely with the nabu stack and java itself at every level, allowing for powerful features.

Glue plays an important role in the Nabu ecosystem but mostly behind the scenes which means most people building applications on Nabu will rarely if ever need glue. It can however be a powerful tool if necessary and it does offer some more advanced functionalities in the nabu stack should the need arise. 

For instance, glue works behind the scenes to build all our web applications, it can be used to do pre- and postprocessing of http requests (including stuff like url rewriting), it is the language used in the cloud console,...

I will be posting a number of articles that are aimed at programmers looking to pick up another language.

## Design Goals

Glue is a mostly functional scripting language where the main goals are:

- **lightweight syntax**: syntax is very lean and new syntax is added rarely, especially if the same can be achieved with existing syntax. Glue often opts to express something as an additional function rather than a syntactical construct
- **easy to use**: the lightweight syntax and the fact that there are few concepts make it an easy to learn and use language
- **immutable**: for the most part data in glue is immutable, but glue provides tools to derive new values from old values. There are some notable exceptions to this like methods and the integration with java objects that allow for state mutation
- **extensible**: every part of glue is modular and can be replaced, down to the parser itself
- **hooks**: there is an extensive hooking system in glue where you can interact with every line of code being executed

**Update**: The immutability, while very useful for parallelization and other purposes, can be annoying to create certain types of scripts. Glue has since been updated to allow for more traditional mutation of data.

Glue obviously did not spring up from a vacuum, it draws inspiration from a number of other languages.

## Heritage

### Java

Glue is based on java so it runs in the JVM, this offers some advantages:

- full access to all things that run on the JVM
- cross platform
- varargs: glue not only supports the java varargs for java-based methods, it also supports varargs for its own functions

Note that the integration with java is at every level. Java beans can be accessed through the query system and all (non-static) methods on objects are exposed as lambdas.

### Xpath

Glue (like blox) has native support for a query syntax that is halfway between xpath and java. This allows for very advanced queries to be written concisely. It also means that all variable access is with the ``/`` instead of the perhaps more traditional ``.``. The latter is reserved for namespace access.

There are quite a few differences with xpath though:

- all operators are java-style operators (with some additions), not the xpath ones
- not all operations in xpath are supported, the query engine is (or at least can be) fully statically typed, some operations like the ``|`` in xpath would break this so are not supported
- the engine operates on a high level API to perform its queries, the API has many implementations, so apart from XML it also supports java beans, JSON,...

### Haskell

Glue is mostly a functional language so it shares at least some ideology with haskell. One of the most important features from haskell also made it to glue: lazy data series! This allows for infinite series. In fact series can be extra-lazy in glue as not only is series unfolding done lazily, in a lot of cases the calculation for a single item in the unfolding is also lazy (note that this is not true of all series!).

This means glue can iterate over such extra-lazy series and not incur the computation overhead of actually calculating every item but still running through the series. This is interesting for features like slicing, reversing, getting items at certain offsets,...

It is also interesting for computing such series in parallel which is what ``resolve()`` will do on such super-lazy series.

### Python

Glue shares very little with python except for one very visible feature: the mandatory whitespace (at least in its default parser) to denote blocks. It is possible to use both spaces and tabs as it just counts the amount of whitespace characters but it is very much recommended to use only tabs.

Glue also supports multiline of any non-block statement (so a line that does not start a new block) by having more whitespace on following lines, for example:

```python
echo("this
	is
	some
		text
	right
	here")
```

This will print out:

```
this
is
some
	text
right
here
```

As you can see the "additional" whitespace needed to indicate that it belongs to the echo statement is ignored but any whitespace beyond that is maintained.

## Own Features

### Optional Typing

In general I'm a big fan of static typing as it allows for a lot of design time guarantees. On the other hand I also get and support the flexibility that dynamic typing can offer.

To balance the two, glue comes with optional typing: you can but are not required to type your variables. If you do type them, the runtime will ensure that the variable is of the correct type (or can be converted to it).

### Optional Namespaces

Glue supports namespaces but they are optional in that all functions in glue have namespaces, but you are not required to type them (though you can). Because glue comes with a small core of methods designed not to conflict name-wise and you can easily scope a glue runtime to your needs, you are unlikely to bump into many naming conflicts. If you do, you can always use the namespace to specify what you want.

### Operator Overloading

Operators can be overloaded! Yay!

### Resources

Each script can have resources attached to it. This allows you to easily package a script with any resources you want without depending on specific file system layouts.

If your script is located at ``/home/alex/scripts/examples/test.glue``, you can make a folder **with the same name** as the script: ``/home/alex/scripts/examples/test``. All files in said folder are automatically linked to the script and can be found through the ``resources()`` method:

```python
for (name : resources())
	content = resource(name)
```

### Templates

Glue comes with a fully functional templating engine where the template can contain:

- ``${statement}``: here the statement can be either a variable or a computation. The output of either is put in place of the original placeholder
- ``${{script}}``: the script is a full embedded glue script where you can echo() the output that should come in place of the placeholder

Depending on the environment where it is running, other combinations may be allowed as this is also a pluggable feature. For example the web framework uses ``%`` ``{My Sentence}`` syntax to automatically translate content in a webpage.

For example we can do this:

```python
content = "This is the message: ${message}"

echo(template(content, 
	structure(message: "first message"), 
	structure(message: "second message")))
```

In this case we create a new lazy series of templates that outputs:

```
[This is the message: first message, This is the message: second message]
```

You can also do this with lists with simple values where the value is injected as ``$value``:

```python
content = "This is the message: ${$value}"
echo(template(content, "first message", "second message"))
```

This will print out the same as the above.

Note that if you only pass in a template and no values to template it against, it will use the current pipeline, for example:

```python
message = "this is it!"
echo(template("message is: ${message}"))
```

This will print:

```
message is: this is it!
```

#### E(mbedded) Glue

The correct parser to use by the glue framework is based on the extension of the file. The default parser is linked to files ending in ``.glue``. However there is also a parser for ``.eglue``. It is basically a template file where you can use the above outlined syntax, for instance if you run the following eglue file:

```xml
<html>
	<body>
${{
for (10)
	echo("		<div>" + $value + "</div>")
}}
	</body>
</html>
```

It will print out:

```
<html>
	<body>
		<div>0</div>
		<div>1</div>
		<div>2</div>
		<div>3</div>
		<div>4</div>
		<div>5</div>
		<div>6</div>
		<div>7</div>
		<div>8</div>
		<div>9</div>
	</body>
</html>
```

### Metadata

There are three types of metadata:

- **comment**: meant for people _reading_ the code, a comment line should start with a single "#". There is no dedicated syntax for multiline comments but multiple lines prefixed with "#" will be interpreted as a single multiline comment
- **description**: meant for people _running_ the code, a description line should start with a double hashtag "##"
- **annotation**: key value pairs that you can set, some have actual meaning depending on the environment and others are merely structured metadata

Metadata is **bound** to the **next** line of code. There is only one exception to this and it must match these requirements:

- the metadata has to be at the top of the script
- it may be followed by more metadata of a different or the same kind
- there is an __empty line__ between the metadata and the first line of code (or the metadata on said line of code)

**Then** the metadata belongs not to the first line but to the script as a whole. This is the standard way to set script-wide comments, descriptions and annotations.

For example:

```python
## This script performs some sort of magic
@tag magic

# This line of code starts off the magic!
startMagic()
```

The top description and annotation belong to the entire script, **not** the first line. The comment right above the line and (more crucially) below the empty line belong to the first line of code. In a lot of cases this distinction is arbitrary but in others it is not.

For example in the web framework -which makes heavy use of annotations- you could do this:

```python
@get
a ?= null
echo("The parameter was: " + a)
```

This page would take an input parameter call "a" in the url and display the parameter in the content. The annotation in this case belongs to the first line of code as there is no empty line above it.

Another example could be a rest-based service:

```python
@path path/to/{id}

@path
id ?= null
echo(id)
```

In this case the top line is a script-level annotation telling the web framework what the rest endpoint should look like.

The second "path" annotation is preceded by an empty line and as such belongs to the first line of code, in this case telling the web framework to inject the parameter "id" into it.

### Advanced

The glue parser builds a memory model of the code to be run, the metadata that is set in the script can be requested from the model and acted upon. 

The deep hooking of the glue runtime allows for example for detailed logging (e.g. based on the descriptions) and for things like conditional execution. The web framework has support for example for role-based differentation:

```python
echo("For all to see")

@role admin
echo("Only the admin can see this")

@role user
sequence
	echo("This is for the user")
	echo("And this...")
```

Another use for the metadata is the loggers. One of the provided loggers outputs in markdown syntax and uses the descriptions in the code to build a narrative where sequences act like headers. This was used for example to create structured test reports in an environment where glue acted as an automated test framework.

## Applications

Glue is a very pluggable language and as such can be used in a lot of different settings:

- **Glue Web Framework**: there is a web framework built on glue where it acts not only as the main processor but also as pre- and postprocessing
- **Integration Service**: the nabu development platform uses glue as its dynamic scripting language and as the console language in the cloud
- **System Management**: glue is fully integrated with the operating system it runs on making it ideal to write system scripts, it drives the build process of the nabu platform and its installation and updates on servers for example
- **SCSS-lookalike**: there is a derivate of the glue parser that uses a syntax more in line with scss
- **Glue Testing**: there is an automated testing framework built on glue

# Data Types

## Basics

At the core, glue is still based on java and as such reuses a lot of the existing types. It does however redefine them to be simpler.

Glue is optionally typed so you do not _need_ to define the type but if you do, it will be checked by the runtime and converted where possible. If it can not be converted an exception is thrown.

These are the data types that are supported by default:

- **integer**: this actually translates to a "java.lang.Long" in the background, it simply means that the number is integer and has no decimal part
- **decimal**: this maps to a java.lang.Double in the background meaning it has a decimal part
- **date**: this maps to java.util.Date, note that there is operator overloading for dates
- **string**: this maps to java.util.String
- **boolean**: this maps to java.lang.Boolean
- **bytes**: this maps to byte[]
- **uuid**: this maps to java.util.UUID
- **uri**: this maps to java.net.URI
- **lambda**: this checks that the given variable is of the type lambda

Especially when writing utility scripts or lambdas it can be nice to have more control over the types that are being sent into your script because they often dictate exactly what will happen at runtime, for instance take this sum lambda:

```python
sum = lambda
	a ?= null
	b ?= null
	@return
	sum = a + b
	
echo(sum(1, 2))
echo(sum(1, "2"))
echo(sum("1", 2))
echo(sum("1", "2"))
echo(sum("a", "b"))
```

What will this output?

```
3
3
12
12
ab
```

Why?

- sum(1, 2): normal calculation
- sum(1, "2"): left operand wins when a type has to be chosen so the string "2" is converted to a number
- sum("1", 2): left operand wins again but it is a string so the number 2 is converted to a string and we get default string concatenation
- sum("1", "2"): strings all around so we get concatenation
- sum("a", "b"): same as the previous

Can we make it crash? Sure do ``sum(1, "a")``

This will throw a ``java.lang.NumberFormatException: For input string: "a"`` because the left type is a number and glue tries to convert the string `a` to a number as well.

It is in such cases that it is easier to use typing to guarantee the end result:

```python
sum = sequence
	integer a ?= null
	integer b ?= null
	@return
	sum = a + b
	
echo(sum(1, 2))
echo(sum(1, "2"))
echo(sum("1", 2))
echo(sum("1", "2"))
```

All results will now be the same: `3`. If you try the ``sum("a", "b")`` it will fail because `a` can not be converted to a number.

## Advanced

The typing system in glue is entirely pluggable so you can plug in whatever you want, there is default support for structures and for all java objects. If however you stray outside of the supported types by default it would be wise to add your own converter logic (and optionally operator overloading etc) as well.

For example you can do this:

```python
java.net.URI uri = "http://google.be"
echo(typeof(uri))
echo(uri)
```

This will print:

```
java.net.URI
http://google.be
```

Glue is also fully integrated with the nabu development platform which supports many ways to define types: XML Schema, UML Class Diagram, Dynamic Modeling,... If you use glue in that context you can use any of the types that nabu knows about.

## Narrow Typing

Note that all typing is only applicable to the line itself, it is not part of the identity of the variable, for example this is perfectly legal:

```python
integer a = "5"
string a = 5
a = 5
```

After the first line, a will indeed be an integer with the value `5`. After the second line it will be a string with the value `5` and after the third line it will be an integer again because it is not explicitly typed and 5 is expressed as a number.

# Operators

## Existing

These are the operators that are supported out of the box in glue:

- +: addition
- -: subtraction
- *: multiplication
- **: power
- /: division (make sure it doesn't collide with variable access)
- ?: in: checks if the left operand is in the right operand, assuming the right is a series or a collection (note in blox this is the "#" operator)
- !?: not in (in blox this is the "!#" operator)
- &&: and where the right operand is _not_ executed if the left one is false
- ||: or where the right operand is _not_ executed if the left one is true
- >, >=, <, <=: the greater than, greater or equals, smaller than and smaller or equals comparison operators. They work out of the box on any type that is Comparable.
- ~: matches: checks if the left operand matches the regex expressed in a string in the right operand
- !~: not matches
- !: not
- %: modulus
- ^: XOR
- ==, !=: equals and not equals.
- °: composition

## Overloading

Glue supports both groovy-esque overloading where the object must implement the respective interfaces for each operation and it supports external overloading where a class is provided that implements overloading as it sees fit.

For example there are overloads for series, lambdas and dates.

# Variables

Variables are obviously pretty important in any language and they work the same as they do in most languages: you can assign stuff to them and you can get that stuff back.

There are however some important differences with most languages. One key difference is how you access a field within a "complex" variable, for example suppose you have a variable that contains a person and said person has a first name, in most languages you would do: ``person.firstName``, however the separator in glue is ``/`` so it would become ``person/firstName``.

## Named access

Building on the example of the person object, these two lines are identical:

```python
echo(person/firstName)
echo(person["firstName"])
```

Both accessors work in the same way and can be performed on any "complex" type. This ranges from structures to maps to java beans to json objects,...

## XPath-like

As noted previous: glue comes with an xpath-like engine that allows you to perform complex lookups on objects. For example suppose we have a collection of persons and we want to find everyone who is over the age of 30, we could do: ``persons[age > 30]``.

We could even build a collection of all the last names of everyone over 30: ``persons[age > 30]/lastName``.

If you want to compare a "local" variable (age belongs to person) to a "global" variable (so not in person but for example at the same level), you need to prepend it with a root slash:

```python
age = 30
echo(persons[age > /age]/lastName)
```

This will print the last name of everyone over 30.

# Functions

Everything is either a variable or a function in glue. What a function does in the background can differ wildly depending on the implementation. There are several ways of adding new functions to glue at the java level, from simply exposing static methods to much more in-depth implementations. 

In the default glue distribution a function is one of these things:

- another glue script
- a lambda
- glue ships with a number of defaults functions for date manipulation, string manipulation, series...
- you can call any static java method directly, e.g. ``echo(java.util.UUID.randomUUID())``

Glue is fully integrated with the nabu development platform and if you use it in that setting, you can call any service that nabu knows about directly as a function.

A number of plugins have been written already that have done stuff like:

- allow you to run a selenium script using the method ``selenese()``
- allow you to run a soapui script using the method ``soapui()``
- actually call a webservice platform running on JBoss remotely

## Namespaces

All functions (except anonymous lambdas) live in a namespace, but as a programmer using that namespace is _optional_. For example the method ``generate()`` lives in the namespace `series` while the method ``lambda()`` lives in the `script` namespace. This means these two lines are identical:

```python
fibonacci = generate(lambda(t2: 0, t1: 1, t2 + t1))
fibonacci = series.generate(script.lambda(t2: 0, t1: 1, t2 + t1))
```

If glue finds a fully qualified method, there is little doubt as to which one it will take. It is theoretically possible to have two functions with the exact same full name and then the first one wins.

If you use only the function name, glue will take the first function that matches that name, regardless of the namespace it is in. This can obviously lead to conflict pretty easily but the core of glue has been designed in such a way that such conflicts are avoided. When in doubt, simply use the full name!

## Scripts

Scripts written in glue are exposed as functions to other scripts. The namespace of the script depends on the folder structure it has relative to the "root" of the scripts folder that glue is scanning. For example suppose you tell glue all your scripts are in "/home/alex/scripts" and you have a script at "/home/alex/scripts/examples/test.glue" the script will have the name "test", the namespace "examples" and the full name "examples.test".

You can set the location of the scripts by either adding it to your system `PATH` variable but this tends to be slow, especially on windows as glue needs to recursively scan each folder listed there. It is better to set a dedicated system variable `GLUEPATH` which it will search recursively for scripts.

There is a slight exception to this rule when using the commandline glue client in that it will **always** add the current directory as well, so if you make a ``test.glue`` script in any folder on your system and on the commandline go to that directory you can run ``glue test``.


### Inputs

In a script you can indicate that a variable is an input parameter with the optional assignment ``?=``. It is called optional assignment because it will still assign the value to the variable **if** it has no value yet (== null). For example if we make a script called ``sum.glue`` and it contains:

```python
a ?= null
b ?= 1
echo(a + b)
```

We can make a second script ``test.glue`` that does:

```python
sum(1, 2)
sum(1)
```

On the first line we give both `a` and `b` values, in the second call we only give `a` a value which means `b` will get its default value of `1`. If in this case we simply call ``sum()`` it will fail because the addition can not deal with ``null`` as the left operand.

### Named Parameters

Glue has support for named parameters for any type of function **if** said function has a runtime definition attached to it so the function call can be rewritten. This is true of almost all methods that are shipped with the default glue distribution and includes any scripts and lambdas you yourself write. It is notably not true for the OS integration because that plugin merely redirects any incoming function call to the system without having a full view of what actually exists there. 

There are also some slightly deeper integrated functions that reuse the named parameter syntax for other reasons, for example the lambda function will use it to define default values, the structure function will use it to pinpoint fields to be set but these are few and they are (hopefully) logical.

For example let's take the ``sum.glue`` script we defined above, we could call it like this:

```python
sum(b: 0, a: 1)
```

Where we specifically set the variables in a different order than they are defined. If you "skip" over variables and don't assign them, they will get their default values or "null" if there are none.

You can combine named and unnamed parameters but it then becomes very important that you know how this works which is: normally input values are assigned in sequential manner meaning the first value is assigned to the first input parameter and so on. If you however throw named parameters in the mix, you "reset" the internal assignment point to wherever the named variable is in the list, so we could for instance do:

```python
sum(b: 0, a: 1, 1)
```

This will output `2` because we first set `b` to `0`, then `a` to `1` which means the internal pointer is "after a", if we then do an unnamed assign, it is assigned again to `b`.

### Varargs

In the above example if we do:

```python
sum(1, 2, 3)
```

We will get an exception: ``Too many parameters when calling: sum``

This is because sum only expects two parameters but gets three.

Much like java, glue supports varargs. It supports both varargs in java methods but also varargs in glue scripts. If you set the last variable to be an array, glue will allow any number of parameters to be sent in, for example let's update the sum script:

```python
[] numbers ?= null
sum = 0
for (number : numbers)
	sum = sum + number
echo(sum)
```

It now accepts any amount of numbers for summation and we can call:

```python
sum(1, 2, 3, 4)
```

Which will output: `10`.

### Multiple Return

All scripts in glue and long-form lambdas (by default) are multiple return: you don't return an actual value but running a script returns its entire pipeline to the calling function, for example using the above sum script we could do:

```python
result = sum(1, 2, 3, 4)
echo("The sum is: " + result/sum)
```

Which will echo at the end: ``The sum is: 10``

Note however that in lambdas you can optionally define the return parameters while the glue services into nabu require explicit definition of the return parameters (in both cases there can be multiple though).

For example let's define a lambda that returns a single value which makes it more like traditional functions:

```python
sum = lambda
	[] numbers ?= null
	@return
	sum = 0
	for (number : numbers)
		sum = sum + number
echo(sum(1, 2, 3, 4))
```
This outputs ``10``

We can however just as easily return multiple values:

```python
sum = lambda
	@return
	[] numbers ?= null
	@return
	sum = 0
	for (number : numbers)
		sum = sum + number
echo(sum(1, 2, 3, 4))
```

This outputs ``{numbers=[1, 2, 3, 4], sum=10}``

# Control Structures

Glue comes with most default control structures you know from java although it generally has some additional niceties to them.

## For

The for loop is versatile, you can loop over elements in a series:

```python
series = series("a", "b", "c")
for (series)
	echo("Element[" + $index + "] = " + $value)
```

This will print:

```
Element[0] = a
Element[1] = b
Element[2] = c
```

As you can see both the index and value are injected automatically into the context. If you want you can give a different name to the value variable:

```python
series = series("a", "b", "c")
for (element : series)
	echo("Element[" + $index + "] = " + element)
```

Which prints out the same.

You can also loop over numbers easily:

```python
for (3)
	echo("Value: " + $value)
```

This will print:

```
Value: 0
Value: 1
Value: 2
```

In this case the "$index" and "$value" contain the same value.

You can use a `break` to pre-emptively break out of a for loop:

```python
for (2)
	for (2)
		echo("Right here!")
		break
```

In this case the message will be echoed twice because the inner loop always breaks immediately.

However you can also tell it to break a certain number of control structures:

```python
for (2)
	for (2)
		echo("Right here!")
		break 2
```

In this case the message will only be printed once as you break out of two control structures.

## While

You can do a classic while loop:

```python
i = 0
while (i < 3)
	echo("Message")
	i = i + 1
```

This will print the message 3 times.

## Switch

The switch statement comes in two flavors (do note the brackets used for the case statements!), the one you would expect:

```python
value = "some value"
switch(value)
	case("is it this value?")
		echo("no")
	case("some value")
		echo("yes")
	default
		echo("hmm, doesn't seem to work")
```

This prints out `yes`.

There is also another variant that works more like an if/else:

```python
value = "some value"
switch
	case(value == "is it this value?")
		echo("no")
	case(value == "some value")
		echo("yes")
	default
		echo("hmm, doesn't seem to work")
```

This also prints out `yes`.

## If

There is classic if/else if/else support:

```python
value = "some value"
if (value == "is it this value?")
	echo("no")
else if (value == "some value")
	echo("yes")
else
	echo("hmm, doesn't seem to work")
```

Prints out the same `yes` as the switch in the previous example.

## Sequence

A sequence is merely a group of steps to run and offer a way to structure your code. You can nest sequences in one another.

```python
# Define Variables
sequence
	a ?= null
	b ?= null
# Do stuff
sequence
	doStuffWith(a, b)
```

One additional important attribute of a sequence is that you can add a catch/finally to it:

```python
sequence
	doSomething()
	
	catch
		echo($exception/message)
	finally
		doSomethingElse()
```

If the ``doSomething()`` throws an exception you can handle it in the catch.

### Try

A try is very much like a sequence but it has one very important difference: if an exception occurs and you don't catch it, it will jump out of the try but the rest of the script will continue. For example:

```python
try
	doSomething()
	doSomethingElse()
	
doSomethingEntirelyDifferent()
```

If the ``doSomething()`` throws an exception the ``doSomethingElse()`` is no longer executed. We don't have a catch in this case so it will continue after the try with ``doSomethingEntirelyDifferent()``.

### Catch

A catch allows you to intercept an exception and optionally handle it. The exception is automatically injected into the variable ``$exception``.

To throw an exception, use the method ``throw()``:

```python
sequence	
	throw("this is a test")
	catch
		echo($exception/message)
		throw($exception)
```

In this case we print out the error message and simply rethrow it.

### Finally

The finally block works the same as in java: it is guaranteed to run whether the sequence/try was successful or not.

### Parameters

You can add parameters to an exception you throw and access them again in the catch:

```python
sequence
	throw("file.not.found", structure(name: "somefile.txt"))
	
	catch
		echo("Type of error: " + cause($exception)/message)
		echo("Parameters: " + cause($exception)/parameters)
```

This will print out:

```
Type of error: file.not.found
Parameters: {name=somefile.txt}
```

That's it for the first deep dive into glue! The next article will focus on series as they are one of the foundational concepts in glue.

@image /resources/images/database.png
@icon database
@description Structures allow for complex type creation based on the multiple return property of glue

# Structures

## Basics

Glue is a functional language and has no object oriented syntactical constructs. However because of how glue works it is possible to have lightweight objects that we call structures. They are based on the fact that glue has multiple return from its scripts, for example let's assume you have a script called 'person.glue':

```python
firstName ?= null
lastName ?= null
age ?= 30
```

It is then possible from another script 'test.glue' to do this:

```python
jim = person("Jim", "Smith")
john = person("John", "Smith")
```

These return values from the person script act more or less like an object in that you can access their values:

```python
echo(jim/firstName + " " + jim/lastName + " aged " + jim/age)
```

This will print out: ``Jim Smith aged 30``

## Typing

By default it is also possible to use scripts like the 'person.glue' as a definition of expected parameters, for example we could make another test script 'test2.glue' which contains:

```python
person thePerson ?= null
echo(thePerson/firstName + " " + thePerson/lastName + " aged " + thePerson/age)
```

As you can see we use the optional typing system to declare the variable `thePerson` to be of the type 'person' which points to the glue script. In 'test.glue' we can now write:

```python
test2(person("John", "Smith"))
```

This will print out: ``John Smith aged 30``

If we now send in some random value, for example:

```python
test2("random value")
```

We get a class cast exception that states: ``The original 'random value' is not compatible with the required parameters``

## Dynamic Structures

The typing information contained in `test2.glue` is to validate that the incoming data contains the necessary fields. It does **not** check that the incoming data was actually constructed by running the ``person.glue`` script. This means any data structure that happens to have the required fields will be able to pass. This could be either the result of yet another script or it could be the result of a dynamic structure created by calling the method ``structure()``.

For example we could do this:

```python
test2(structure(firstName: "John", lastName: "Smith"))
```

This will print: ``John Smith aged 30``

Apart from being allowed to pass by the typing system, it is very important to notice that the age is still 30, even though we didn't set it in the structure. Because the ``person.glue`` script has a default value for age and no age was set in this structure, it will be automatically added by the casting.

Two more things to note here:

```python
test2(structure(firstName: "John", lastName: "Smith", age: null))
```

This will output: ``John Smith aged null``

Here we see that the age can be explicitly set to null, even with the default value.

Additionally if we run this:

```python
test2(structure(firstName: "John"))
```

You get an exception: ``The original '{firstName=John}' is not compatible with the required parameters``

Because lastName was not specified and does not have a default value in the person script.

## Immutability

Structures are -like all other data in glue- immutable. This means in the above example we could never update the firstName of a person once the instance is created.

However, it is entirely possible to create a new instance that inherits all the state of the first instance and overwrites some or all of it. For example:

```python
jim = person("Jim", "Smith")
john = structure(jim, firstName: "John")
echo(jim, john)
```

This prints out:

```
{firstName=Jim, lastName=Smith, age=30}
{firstName=John, lastName=Smith, age=30}
```

## Lambdas

It is possible to add functionality to structures because they, just like any other script, can contain and return lambda's. For example we could update our ``person.glue`` script to contain:

```python
firstName ?= null
lastName ?= null
age ?= 30
fullName = lambda(firstName + " " + lastName)
```

We could now do:

```python
jim = person("Jim", "Smith")
john = structure(jim, firstName: "John")

echo(jim/fullName())
echo(john/fullName())
```

Because lambdas have an **immutable** context that is captured at creation, both methods will print the same.

```
Jim Smith
Jim Smith
```

However if you use a method, the ``structure()`` logic will create **new** methods with the same logic but a new context allowing for context aware structure adaptation:

```python
Person = lambda
	firstName ?= null
	lastName ?= null
	age ?= 30
	fullName = method(firstName + " " + lastName)
	
jim = Person("Jim", "Smith")
john = structure(jim, firstName: "John")

echo(jim/fullName())
echo(john/fullName())
```

At this point we get output that corresponds to the structure:

```
Jim Smith
John Smith
```

## Lambda Types

You can also use lambdas as structures, for example:

```python
person = sequence
	firstName ?= null
	lastName ?= null
	fullName = firstName + " " + lastName
	
person p = structure(firstName: "John", lastName: "Smith")
echo(p/fullName)
```

This will print:

```
John Smith
```

## Composition

You can compose two structures into a new one using the ``+`` operator. This will create a new structure (without modifying the old ones) that contains a merged combination of both. For example:

```python
a = structure(test1: "string1", child: structure(a: "firstA", b: "firstB"))
b = structure(test2: "string2", child: structure(b: "secondB"))

echo(a + b)
```

This will print:

```
{test1=string1, child={a=firstA, b=secondB}, test2=string2}
```
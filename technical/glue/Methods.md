@image /resources/images/service.png
@icon service
@description Methods are lambdas with state, use with caution as they break immutability and all the implicated goodness

# Methods

Methods are essentially the same as [lambdas|$Lambdas.md] with one very important difference: their enclosed context is _mutable_. Where lambda's take an immutable snapshot of the pipeline at creation, methods share the same pipeline as they were defined in.

## Mutable Context

Having a mutable context means they can see updates to variables. For example:

```python
Person = lambda
	firstName ?= null
	lastName ?= null
	f = lambda(firstName + " " + lastName)
	g = method(firstName + " " + lastName)
	firstName = "Bob"
	
Person person = Person("John", "Smith")
echo(person/f())
echo(person/g())
```

The lambda captures the value of firstName when the lambda is created and it becomes immutable. The method however shares the pipeline with its parent (the Person in this case) so it sees the update that happens to the firstName afterwards. This means the code prints:

```
John Smith
Bob Smith
```

## Setters

The short form methods do not allow you to overwrite something in the context because you can not make any assignments. Long form methods however can feed back data to the parent scope which can be shared by other methods, for example:

```python
Person = lambda
	firstName ?= null
	lastName ?= null
	fullName = method(firstName + " " + lastName)
	update = method
		@persist
		firstName ?= null
		@persist
		lastName ?= null

# Create a person called John Smith
Person person = Person("John", "Smith")
echo(person/fullName())

# Update the name
person/update("Bob", "Jones")
echo(person/fullName())
```

Because both methods share the same context, any update from a method to its context is seen by other methods sharing the same context. The code outputs:

```
John Smith
Bob Jones
```

## Lambda Heritage

As noted above, methods are _identical_ to lambdas apart from their mutable context. This means that all logic concerning series, higher order functions, lambda math etc can be done with methods as well.

```python
Item = lambda
	counter = 0
	increment = method
		amount ?= 1
		@return
		@persist
		counter = counter + amount

Item item = Item()

f = lambda(x, x + 1)
g = item/increment
h = f ° g

echo(h(1), h(1), h(1))
```

This prints out:

```
2
3
4
```

@image /resources/images/lambda.png
@icon lambda
@description Short form lambdas, long form lambdas, composition, currying...

# Lambdas

Lambdas are anonymous inline functions and are at the core of many features in glue. There is support for higher order functions and current lambdas are all fully enclosed.

## Short Form

The short form of creating a lambda is by calling the method ``lambda()`` where all but the last parameter counts as an input parameter. The last parameter is the operation that is executed. The input parameters can use the syntax for named variables to define a default value.

For example let's create a lambda that sums two numbers:

```python
sum = lambda(x: 0, y: 1, x + y)
echo(sum(1, 2))
echo(sum(1))
echo(sum())
echo(sum(y: 2))
```

This outputs:

```
3
2
1
2
```

As you can see it is possible to target specific input parameters in a lambda as well.

The important downside of the short form lambda is that it can not take control structures. There is a ``when(condition, then, else)`` method that allows you to use a control method instead but it is only advised for short lambda's.

## Long Form

The long form to create a lambda allows you to write full scripts instead of being limited to oneliners. They also act as full scripts in that they have multiple return by default but it is possible to tag one or more values for return. If a single value is tagged as return value, it becomes a classical single return lambda.

```python
sum = lambda
	x ?= 0
	y ?= 1
	@return
	sum = x + y

echo(sum(1, 2))
echo(sum(1))
echo(sum())
echo(sum(y: 2))
```

This outputs the same as the short form lambda we defined above.

If you remove the return annotation, you get:

```
{x=1, y=2, sum=3}
{x=1, y=1, sum=2}
{x=0, y=1, sum=1}
{x=0, y=2, sum=2}
```

## Scope

Lambda's enclose their definition scope but in an immutable manner, for example:

```python
a = 5
b = lambda(a + 1)
a = 10
echo(b())
```

This will print out ``6`` because the lambda can access the variable ``a`` in its parent scope but does not see any changes made to `a` after the lambda definition. As such lamda's follow the immutability goal of glue.

Local variables win out on parent scope variables, for example:

```python
a = 5
b = lambda(a, a + 1)
a = 10
echo(b(a))
```

This will print: `11` because it does not use the "a" from the parent scope at all but rather the locally defined a that serves as an input parameter, which at the point of call contains 10.

**Note**: [Structures|$Structures.md] still follow the immutability of the enclosed scope for lambda's but define new lambda's with updated scope when you extend other structures.

## Higher Order Lambda's

It is of course possible to return a lambda from a lambda, this can be used to have a more dynamic lambda where the scope is set by its parent. It also offers a paradigm similar to ``bind()`` in the javascript world where you take a function and fix one or more of its input parameters, for instance:

```python
sum = lambda(a, b, a + b)
increment = lambda(a, sum(a, 1))
echo(sum(1, 2))
echo(increment(3))
```

This prints out predictably:

```
3
4
```

## Composition

You can create a composition of lambda's where the resulting composite gets the same input parameters as the first lambda. All subsequent lambdas get the output of the previous lambda as an input and the resulting output of the final lambda is returned as the composite result. This means the ``compose()`` method pipelines the input **from left to right**. For example:

```python
sum = lambda(a, b, a + b)
increment = lambda(a, a + 1)

total = compose(sum, increment)

echo(total(1, 2))
```

This will echo `4` as the 1 and 2 are first added by the first lambda, then fed into the second lambda where they are incremented.

If we update the composite:

```python
total = compose(sum, increment, increment, increment)
```

The above code would now print `6`.

## Math Composition

In the math world composition works from **right to left**, for example ``(f ∘ g)(x) = f(g(x))``. As you can see the right most function (g) is first executed and the result of that execution is given to the left most function (f) to operate on.

Glue supports math based composition of lambda's:

```python
f = lambda(a, a + 1)
g = lambda(a, a * 2)

result = f ° g

echo(result(2))
```

This prints `5` as the value "2" is first multiplied with 2 to yield 4 which is then incremented to 5.

## Lambda Math

The composition operator takes precedence over all operators but you can still use the other operators as well. This will create a new lambda where the given calculation is executed. It is important that all lambda's used in such calculations take a single parameter as input, for example:

```python
f = lambda(a, a + 1)
g = lambda(a, a * 2)

result = f + g * 2 + f ° g
echo(result(2))
```

This prints 16 because it executes:

```
(2 + 1) + ((2 * 2) * 2) + ((2 * 2) + 1)
```

If we add:

```python
result = result ** 2
echo(result(2))
```

This will print: ``256`` (= 16 * 16)

Note that this is not limited to math operations:

```python
f = lambda(a, a * 2)
f = "The result is: " + f
echo(f(2))
```

This prints: ``The result is: 4``

## Polymorphism

You can combine multiple lambdas into a new lambda that will dispatch any incoming call depending on amount of parameters and (if available) type of parameters. This can not (currently) be combined with varargs.

For example parameter overloading:

```python
f = lambda(a, b, a + b)
g = lambda(a, b, c, (a + b) * c)
h = dispatch(f, g)
echo(h(1, 2))
echo(h(1, 2, 3))
```

Once a lambda has been found with the correct amount of parameters, it will check the types of all the parameters. It will try to convert the incoming values to the requested types. If one or more parameters can not be converted, the lambda is ignored. If all parameters can be converted, it will check if the resulting match is exact or not. For example:

```python
f = lambda
	integer a ?= null
	integer b ?= null
	@return
	result = a + b

g = lambda
	string a ?= null
	string b ?= null
	@return
	result = "The resulting string is: " + a + b
	
h = dispatch(f, g)

echo(h(1, 2))
echo(h("test", "this"))
echo(h(1, "2"))
echo(h("1", 2))
echo(h("1", "2"))
```

This prints:

```
3
The resulting string is: testthis
3
3
The resulting string is: 12
```

The dispatcher will dispatch as follows:

- the first call is matched to the integer method as that is an exact match
- the second call is matched to the string method as that is an exact match
- the third and fourth calls are matched to the integer one as both have an equal match for both dispatch options and the integer is listed **before** the string one
- the final one is again matched to the string method even though the strings contain valid numbers because it has a better match than the numeric one

You can obviously combine this with complex types as well to get more interesting results, for example:

```python
animal = lambda
	type ?= null

dog = lambda
	name ?= null
	type = "dog"

f = lambda
	dog d ?= null
	echo("A dog with the name: " + d/name)

g = lambda
	animal a ?= null
	echo("An animal of type: " + a/type)

h = dispatch(f, g)

h(animal("cat"))
h(dog("Spike"))
```

Here we create two types "animal" and "dog" using the long lambda form and two methods that act upon them which we combine in a dispatcher. Note that in this case the more generic "animal" handler is registered after the more specific "dog" handler as any dog is an animal in this example.

We can now use the dispatching lambda to trigger different behavior depending on the parameters, the above outputs:

```
An animal of type: cat
A dog with the name: Spike
```

### Adding Lambdas

If you wrap a ``dispatch()`` around an already existing dispatcher, it will "extend" the original one in that it will get the lambdas it contains rather than the lambda wrapper, for instance:

If we add the following code to the above example at the end:

```python
human = lambda
	firstName ?= null
	lastName ?= null

i = lambda
	human h ?= null
	echo("A human called: " + h/firstName + " " + h/lastName)

h = dispatch(h, i)

h(human("John", "Smith"))
```

We get the full output:

```
An animal of type: cat
A dog with the name: Spike
A human called: John Smith
```

## Java Integration

All non-static methods on java objects are exposed as lambdas, for example suppose you have this class:

```java
class MyExample {
	private List<String> strings = new ArrayList<String>();
	public void add(String...strings) {
		this.strings.addAll(Arrays.asList(strings));
	}
	public List<String> get() {
		return strings;
	}
}
```

Suppose the variable `example` has an instance of the above class, we could do this:

```python
example = ...
echo(example/get())
example/add("test1", "test2")
echo(example/get())
```

This prints out:

```
[]
[test1, test2]
```

This -much like [methods|$Methods.md]- allows for mutable state.

## Capabilities

### Functions

Functions are broader than lambdas in that all lambdas are functions but not all functions are lambdas. A function can be many things from a static java method to a complex plugin to a script, a lambda,...

You can wrap _any_ function as a lambda however, for example let's take the venerable ``echo()`` function:

```python
test = function("echo")
test("this is a test")
```

This will print out:

```
this is a test
```

This allows us to pass functions that are not inherently lambdas around as if they were. As they are for all intents and purposes actual lambdas, all the above rules apply.

### Apply

You can use the ``apply()`` method to call a lambda with a series that contains the required parameters, for example:

```python
test = lambda(a, b, a + b)
echo(apply(test, 1, 2))
echo(apply(test, series(1, 2)))
```

This will print out `3` twice.

### Curry

You can curry a lambda where you partially apply parameters and return a new lambda with the remaining parameters. If all parameters have a value, the original lambda is executed and the result is returned, for example:

```python
test = curry(lambda(a, b, c, a + b + c))

# Execute it normally
echo(test(1, 2, 3))

# Named parameters still work
echo(test(c: 1, b: 2, a: 3))

# Partially apply with the first parameter (a) filled in, then run it with the remaining parameters:
echo(test(1)(2, 3))

# Partially apply all parameters
echo(test(1)(2)(3))

# Partially apply using named parameters
test = test(c: 1, a: 2)
echo(test(3))

# Named parameters still work on partially applied as well
echo(test(b: 3))
```

This will print:

```
6
6
6
6
6
6
```

### Decorate

The ``decorate()`` function allows you to wrap functionality around a lambda. For example:

```python
timer = lambda
	function ?= null
	args ?= null
	started = date()
	apply(function, args)
	echo("Took: " + (date() - started) + "ms")
test = decorate(function("echo"), timer)
test("this is a test")
```

This prints out:

```
this is a test
Took: 0ms
```

We can also wrap multiple decorators:

```python
printer = lambda
	function ?= null
	args ?= null
	echo("Arguments are", args)
	apply(function, args)

test = decorate(function("echo"), timer, printer)
test("this is a test")
```

This prints out:

```
Arguments are
[[this is a test]]
this is a test
Took: 1ms
```

As you can see the decorating lambda should have exactly two inputs: the original lambda and the arguments which will be passed in as a series.

Note that the resulting decorator has the same interface as the original lambda so you can still used named parameters.

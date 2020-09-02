# Introduction

Series are at the heart of glue which is why it comes with a lot of tools to manipulate them.

As mentioned in the introduction: series are generally lazy, that means that the next is only calculated on demand, this allows for example for infinite series. Glue goes one step further however and where possible creates series where the actual calculation of each item in the series can also be done lazily. That means traversing the series does not always incur the overhead of calculating each item in it. 

It also means that the actual calculation can be done in parallel when required.

## Simple Series

You can create static series easily:

```python
simple = series(1, 2, 3)
```

If you want to reverse the series so it contains [3, 2, 1] you could do:

```python
reversed = reverse(simple)
```

Getting the first element and the last element of a series is equally easy:

```python
first = first(simple)
last = last(simple)
```

Note that the last method will have to run over the entire series (so does not work on infinite series) but does not execute lazy parts of the list along the way.

You can always access a specific entry in a series: ``echo(simple[1])`` will print "2" in this case as series are 0-based.

## Fibonacci

Apart from the simple examples, data series revolve mostly around lambda's. For example let's generate a fibonacci sequence:

```python
fibonacci = generate(lambda(t2: 0, t1: 1, t2 + t1))
```

This generates a data series in memory that will continue indefinitely if it is ever executed in its entirety. For example this would run forever printing out numbers:

```python
for (item : fibonacci)
	echo(item)
```

Note that each subsequent number is only calculated when it is required by its context, for instance in this case the for loop is asking for the next item.

## Slice and Dice

Suppose you only wanted to print out the first 10 elements of fibonacci, you could do:

```python
for (item : limit(10, fibonacci))
	echo(item)
```

This would print out:

```
0
1
1
2
3
5
8
13
21
34
```

If we wanted to start at a certain offset, we could do:

```python
fibonacciWithOffset = offset(10, fibonacci)
for (item : limit(10, fibonacciWithOffset))
	echo(item)
```

This would print:

```
55
89
144
233
377
610
987
1597
2584
4181
```

The offset can also be negative at which point it will take off items at the end:

```python
series = series("a", "b", "c", "d")

for (value : offset(-2, series))
	echo(value)
```

This would print out:

```
a
b
```

Note that negative offsets use runtime lookahead and do not resolve the series beyond the window.

## Derive

You can derive a new series from one or more input series, for example let's create the multiple of the fibonacci and lucas series:

```python
fibonacci = generate(lambda(t2: 0, t1: 1, t2 + t1))
lucas = generate(lambda(t2: 2, t1: 1, t2 + t1))

multiple = derive(lambda(x, y, x * y), fibonacci, lucas)

for (value : limit(10, multiple))
	echo(value)
```

This prints out:

```
0
1
3
8
21
55
144
377
987
2584
```

Note that a simple multiplication of series can also be achieved like this:

```
fibonacci = generate(lambda(t2: 0, t1: 1, t2 + t1))
lucas = generate(lambda(t2: 2, t1: 1, t2 + t1))

echo(limit(10, fibonacci * lucas))
```

Note that the operation ``fibonacci * lucas`` creates a new series that is _not_ calculated at that time (which would be impossible as both are infinite), which is why we can still wrap a limit around it.

## Explode

You can explode a series into a new series where each value of the original series represents 0 or more values in the new series, for example we explode a series of [1, 2, 3] into a series that also contains the negative numbers:

```python
series = explode(lambda(a, series(a, -a)), 1, 2, 3)

echo(series)
```

This will print:

```
[1, -1, 2, -2, 3, -3]
```

You could remove values from the resulting list as well:

```
series = explode(lambda(a, when(a != 1, series(a, -a))), 1, 2, 3)

echo(series)
```

This will print:

```
[2, -2, 3, -3]
```

## Filter

Suppose we want to take the first 100 items in the fibonacci sequence and only use those that are between 25 and 50:

```python
fibonacci = generate(lambda(t2: 0, t1: 1, t2 + t1))
filtered = filter(lambda(x, x >= 25 && x < 50), limit(100, fibonacci))

for (value : filtered)
	echo(value)
```

This prints out: `34`

## Merge

You can merge multiple series together:

```python
fibonacci = generate(lambda(t2: 0, t1: 1, t2 + t1))
merged = merge(
	limit(5, fibonacci), 
	reverse(limit(5, fibonacci)))

for (value : merged)
	echo(value)
```

This will print out:

```
0
1
1
2
3
3
2
1
1
0
```

## Sort

You can sort a series but this will trigger a resolve() in the background so it should not be run on infinite series.

For example let's reverse sort this list of integers:

```python
series = series(1, 2, 3, 4)
echo(sort(lambda(a, b, b - a), series))
```

This will output:

```
4
3
2
1
```

## Repeat

You can repeat a series, note that this repeated series becomes infinite:

```python
series = series(1, 2, 3)
series = repeat(series)
echo(limit(9, series))
```

This will print out:

```
1
2
3
1
2
3
1
2
3
```

## Unique

You can generate a series where each element is unique, there are no doubles. Note that this triggers a ``resolve()``.

```python
series = series(1, 2, 3, 3, 2, 4, 5)
echo(unique(series))
```

This prints:

```
1
2
3
4
5
```

## Position

You can find the position of an arbitrary element, this will only resolve the list until the position is found. For example let's find the position of the element '2':

```python
series = series(1, 2, 3, 4, 5, 2)
echo(position(lambda(x, x == 2), series))
```

This outputs: `1`.

### Subsequent

If you need subsequent positions, there are a number of ways to get them, the easiest is to add a second parameter to your lambda which will contain the current index:

```python
series = series(1, 2, 3, 4, 5, 2)

firstIndex = position(lambda(x, x == 2), series)
secondIndex = position(lambda(x, index, x == 2 && index > firstIndex), series);

echo("Indexes: " + firstIndex + ", " + secondIndex)
```

This outputs:

```
Indexes: 1, 5
```

Alternatively you can use offsets:

```python
series = series(1, 2, 3, 4, 5, 2)

finder = lambda(x, x == 2)
firstIndex = position(finder, series)
secondIndex = position(finder, offset(firstIndex + 1, series))

echo("Indexes: " + firstIndex + ", " + (firstIndex + 1 + secondIndex))
```

This will echo the same result as the above.

### Advanced

If you need to do a lot of position finding, it could be easier to use a lambda generator:

```python
series = series(1, 2, 3, 4, 5, 2)
positionFinder = lambda(valueToFind, currentIndex: -1, 
	lambda(x, index, x == valueToFind && index > currentIndex))

firstIndex = position(positionFinder(2), series)
secondIndex = position(positionFinder(2, firstIndex), series)

echo("Indexes: " + firstIndex + ", " + secondIndex)
```

### Last Position

If you want the last position of a match, the list will have to be resolved fully anyway. This combined with the (in my experience) rare need for such a method means there is not a dedicated one, there are however ways to get your last position:

```python
series = series(1, 2, 3, 4, 5, 2)
lastIndex = size(series) - 1 - position(lambda(x, x == 2), reverse(series))

echo("Last index: " + lastIndex + " = " + series[lastIndex])
```

This will print out:

```
Last index: 5 = 2
```

## From/Until

You can select a starting point and an endpoint based on a lambda using the from and until methods, for example let's select the fibonacci sequence numbers that are between 10 and 10000:

```python
fibonacci = generate(lambda(t2: 0, t1: 1, t2 + t1))
series = from(lambda(x, x > 10), fibonacci)
series = until(lambda(x, x > 10000), series)
echo(series)
```

This will print out:

```
[13, 21, 34, 55, 89, 144, 233, 377, 610, 987, 1597, 2584, 4181, 6765]
```

## Aggregate

### Sum

You can aggregate a new series from an existing one, for example suppose we want to make a sum series that returns the sum of another series at any point in time:

```python
fibonacci = generate(lambda(t2: 0, t1: 1, t2 + t1))
sum = aggregate(lambda(current: 0, new, current + new), fibonacci)

for (value : limit(10, sum))
	echo(value)
```

This will print out the total sum of the fibonacci series at each stage:

```
0
1
2
4
7
12
20
33
54
88
```

### Average

You can use aggregation in combination with other methods to create more complex calculations, for example a naive average calculation could be:

```python
fibonacci = generate(lambda(t2: 0, t1: 1, t2 + t1))
counter = generate(lambda(x: 1, x + 1))
sum = aggregate(lambda(current: 0, new, current + new), fibonacci)
average = derive(
	lambda(currentSum, currentCount, currentSum / currentCount), 
	sum, 
	counter)

for (value : limit(10, average))
	echo(value)
```

This will print out:

```
0
0
0
1
1
2
2
4
6
8
```

### Min

You can calculate the min of a series using the aggregate function:

```python
series = series(3,2,1,2,4,3)
# a series that calculates at each position the min until then
min = aggregate(lambda(current, new, when(current == null || new < current, new, current)), series)
# the actual min of the entire series
echo(last(min))
```

This prints out: `1`

### Max

You can calculate the max of a series in the same way:

```python
series = series(3,2,1,2,4,3)
# a series that calculates at each position the min until then
max = aggregate(lambda(current, new, when(current == null || new > current, new, current)), series)
# the actual min of the entire series
echo(last(max))
```

This prints out: `4`

## Parallel Execution

By calling ``resolve()`` some series can be executed in parallel which massively optimize the result time. It is important to note that not all series support this, for example aggregation and generation have a sequential order to them as they base new values on historic values. The code below will still work, except it will run single threaded instead of multi threaded:

```python
# This list can not be generated in a multithreaded way
fibonacci = generate(lambda(t2: 0, t1: 1, t2 + t1))
# This one can
derivation = derive(lambda(x, x + 1), fibonacci)
# The resolve will perform multithreaded computation if possible 
# but the end result is always the same: a fully resolved (as opposed to lazy) series
result = resolve(limit(10, derivation))

echo("Result = " + result)
```

This will print:

```
Result = [1, 2, 2, 3, 4, 6, 9, 14, 22, 35]
```

## Maps

You can take one or more series of data and make a map out of them:

```python
result = map("firstName", "lastName",
	series("John", "Smith"),
	series("Jim", "Smith"),
	series("Bob", "Smith"))
echo(result)
```

This will print out:

```
[{firstName=John, lastName=Smith}, {firstName=Jim, lastName=Smith}, {firstName=Bob, lastName=Smith}]
```

You can then loop over the map:

```python
for (person : result)
	echo("Person is: " + person/firstName + " " + person/lastName)
```

This prints:

```
Person is: John Smith
Person is: Jim Smith
Person is: Bob Smith
```

# Advanced Usage

## Contains

There is no contains method as there is an operator that check for contains:

```python
series = series(1, 2, 3)
# Check if 1 is in the series
echo(1 ? series)
# Check if 1 is not in the series
echo(1 !? series)
# Check if 4 is in the series
echo(4 ? series)
# Check if 4 is not in the series
echo(4 !? series)
```

This prints out:

```
true
false
false
true
```

## Operator Overloading

You can use classic operators on series of data, note that these new series are also lazily executed. For example let's take a series and add "1" to each element in it.

```python
simple = series(1, 2, 3) + 1
```

This will create a series that contains "2, 3, 4".

You can also combine series with one another using classic operators, for example let's multiply two series:

```python
simple = series(1, 2, 3)
result = simple * reverse(simple)

for (value : result)
	echo(value)
```

This will print out:

```
3
4
3
```

This can also be used for example for string concatenation:

```python
strings = series("a", "b", "c") + " - test"
```

This creates a series that contains "a - test", "b - test", "c - test".

Obviously more complex operations are also possible and they follow the classic rules to operator order:

```python
simple = series(1, 2, 3)
result = simple - simple * 2

for (value : result)
	echo(value)
```

This prints out:

```
-1
-2
-3
```

## Indexed Access

You can access a series (infinite or not) with a numeric index:

```python
fibonacci = generate(lambda(t2: 0, t1: 1, t2 + t1))
echo(fibonacci[5])
```

This will print out: 5 without resolving the rest of the series.

Note that if the series is lazily executed (not the case here), items 0-4 will **not** be executed.

As long as glue can determine that the result is an integer, you can also use variable statements as numeric index without evaluating the rest of the infinite list:

```python
fibonacci = generate(lambda(t2: 0, t1: 1, t2 + t1))
a = 5
echo(fibonacci[a + 4 - (2 * 2)])
```

## Query Access

You can also access series with a query, it is very important to note however that the series will be resolved single-threaded before this query is actually run. It can not be run on infinite series and if you want to take advantage of parallel processing, you should ``resolve()`` the series first.

For example let's find all the fibonacci numbers that are larger than 2 but smaller than 10 within the first 50 numbers:

```python
fibonacci = limit(50, generate(lambda(t2: 0, t1: 1, t2 + t1)))
echo(fibonacci[$this > 2 && $this < 10])
```

This will print out:

```
3
5
8
```

Note that the ``$this`` used in the query refers to the element in the iteration. If you have a series of complex objects you can obviously use those variables, for example:

```python
series = series(
	structure(name: "John", age: 25),
	structure(name: "Bob", age: 30),
	structure(name: "Jim", age: 35))
# Find all the people that are 30 or older:
echo(series[age >= 30])
```

This will print out:

```
{name=Bob, age=30}
{name=Jim, age=35}
```

## Grouping

You can also group elements together using arbitrary keys. An element can even belong to multiple keys, just return a series of keys from the lambda.

For example if in the above example we wanted to create two groups of people: those under 30 and those over:

```python
series = series(
        structure(name: "John", age: 25),
        structure(name: "Bob", age: 30),
        structure(name: "Jim", age: 35))

echo(group(lambda(person, person/age >= 30), series))
```

This will output:

```
{false=[{name=John, age=25}], true=[{name=Bob, age=30}, {name=Jim, age=35}]}
```

In this case you see the key was a boolean but it could be anything. You can also access the grouped result directly:

```
grouped = group(lambda(person, person/age >= 30), series)
echo(grouped[false])
```

This will output only those under 30:

```
{name=John, age=25}
```

In this example you could also opt for more expressive keys:

```python
grouped = group(lambda(person, when(person/age >= 30, "old", "young")), series)
echo(keys(grouped))
```

This will output the keys of the grouped values being:

```
young
old
```

## Unwrap

When working with the varargs capabilities, we sometimes get a series of series that we want to pass along as separate elements, for example:

```python
example = sequence
        [] parameters ?= null
        @return
        result = derive(lambda(x, y, x + y), unwrap(parameters))

echo(example(series(1, 2, 3), series(2, 3, 4)))
```

While this example is rather dumb because the ``example()`` method takes any amount of parameters but the derive inside of it expects exactly two, it does serve to illustrate the point of unwrapping.

We pass in two series to the example lambda. They are combined into a new series and become a matrix if you will. However we want to pass two distinct series to the ``derive()`` method instead of a matrix. We can at that point call the ``unwrap()`` method which will strip the outer series and replaces it with an array that in turn plays nice with the varargs supported by java.

# Classics

There is a strong focus in glue to offer one way of solving a problem rather than 5 different ways, each with their caveats. In this section I cover some methods that are lacking in glue but can actually be done with other methods:

## Push/Unshift/Offer/...

These methods revolve around adding elements at the back or front of a series, they can be achieved using the ``merge()`` functionality:

```python
series = series(1, 2, 3)
# push at the end
series = merge(series, 4, 5)
# unshift at the beginning
series = merge(0, series)
```

At this point it contains: [0, 1, 2, 3, 4, 5]

In fact merge will allow you to merge a combination of collections and standalone values.

## Pop/Shift/Pull/...

These methods revolve around removing elements at the back or the front of a series, this can be achieved with ``offset()``:

```python
series = series(0, 1, 2, 3, 4, 5)
# remove at front
series = offset(1, series)
# remove at back
series = offset(-2, series)
```

The series now contains: [1, 2, 3]

## Slice/Sublist/...

The slice functionality in general allows you to take a part of a list, you can do this with a combination of ``offset()`` and ``limit()``:

```python
series = series(0, 1, 2, 3, 4, 5)
# base it on an initial offset and a number of items
first = limit(3, offset(1, series))
# base it on an initial offset and an end offset
# in this particular case we remove the first and last items of the series
second = offset(-1, offset(1, series))
```

## Splice

The splice functionality allows you to remove parts of a list and/or replace them with some other elements, this can be achieved with a combination of ``offset()``, ``limit()`` and ``merge()``:

```python
series = series(0, 1, 3, 4)
series = merge(
	limit(2, series),
	2,
	offset(2, series))
```

At this point the list contains: [0, 1, 2, 3, 4]

## Zip

The python ``zip()`` method is a special case of ``derive()``:

```python
series1 = series(1, 2, 3)
series2 = series(5, 4, 3)
zip = lambda(x, y, series(x, y))
echo(derive(zip, series1, series2))
```

If you run this it prints:

```
[[1, 5], [2, 4], [3, 3]]
```

You could also implement your own ``zip()`` method as follows:

```python
zip = lambda
	[] series ?= null

	helper = sequence
		[] input ?= null
		@return
		result = series(unwrap(input))

	@return
	result = derive(helper, unwrap(series))

echo(zip(series(1, 2, 3),
	series(5, 4, 3)))

echo(zip(series(1, 2, 3),
	series(5, 4, 3),
	series(10, 9, 8)))
```

This prints out:

```
[[1, 5], [2, 4], [3, 3]]
[[1, 5, 10], [2, 4, 9], [3, 3, 8]]
```

## Any/All

Given a series of booleans, you can check whether any boolean is true or all booleans are true. For example:

```python
regexes = series("a.*", "b.*")
## Generate a series of booleans
echo("aa" ~ regexes)
## Check if the boolean "true" is in that series (= any)
echo(true ? ("aa" ~ regexes))

## Check that the boolean "false" is not in that series (= all)
echo(false !? ("aa" ~ regexes))
```

This prints out:

```
[true, false]
true
false
```

When working with if clauses it is easiest to work with the "in" operator for ``any()`` and use the default boolean conversion for ``all()``.

```python
regexes = series("a.*", "b.*")

if (true ? ("aa" ~ regexes))
	echo("Matches!")
else if (!("aa" ~ regexes))
	echo("No match :(")
```

This prints out:

```
Matches!
```

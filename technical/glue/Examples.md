@priority 1
@icon example
@description Hands on examples of glue script, sometimes the easiest way to learn is to see it in action
@image /resources/images/example.png

# Examples

## Key/Value Pairs

Suppose you have a series of objects that have actual fields and you want to transform them into a key/value pair list, you could do this with:

```python
series = series(
	structure(name: "John", age: 25),
	structure(name: "Bob", age: 30),
	structure(name: "Jim", age: 35))

values = derive(lambda(person, resolve(derive(lambda(key, structure(key: key, value: person[key])), keys(person)))), series)

echo(resolve(values))
```

This will print out:

```
[{value=John, key=name}, {value=25, key=age}]
[{value=Bob, key=name}, {value=30, key=age}]
[{value=Jim, key=name}, {value=35, key=age}]
```

To get the key value pairs back into objects you could do:

```python
result = derive(lambda(pairs, last(aggregate(lambda(current: structure(), pair, structure(current, lambda(pair/key): pair/value)), pairs))), values)
echo(resolve(result))
```

This will output:

```
{name=John, age=25}
{name=Bob, age=30}
{name=Jim, age=35}
```

## Pi

### Leibniz

```python
# We define a infinite list that simply counts upwards
counter = generate(lambda(x: 0, x + 1))

# We define the leibniz algorithm using the counter
leibniz = (-1.0 ** counter) / (2.0 * counter + 1)

# We define a sum lambda that uses big decimals as sum to ensure accuracy
sum = lambda(current: 0.0b, new, current + new)

# We sum the leibniz series and multiply the series by 4
pi = aggregate(sum, leibniz) * 4

# We print out a the millionth position
echo(pi[1000000])
```

This prints out: ``3.14159365358879342913000``

### Nilakantha

```python
# Regulates the sign
sign = generate(lambda(x: 1.0, x * -1))

# Regulates the counter we use for nilakantha
counter = generate(lambda(x: 2, x + 2))

# The actual formula
nilakantha = sign * (4.0 / (counter * (counter + 1) * (counter + 2)))

# A summation using large decimals
sum = aggregate(lambda(sum: 3.0b, new, sum + new), nilakantha)

echo(sum[100])
```

This prints out: ``3.1415928891420810760002``

## Euler's Number

We can generate an approximation of Euler's number using:

```python
# An incremental counter
counter = generate(lambda(x: 1, x + 1))

# We use the counter to generate the factorials
factorial = 1.0b / aggregate(lambda(sum: 1b, new, sum * new), counter)

# We then aggregate the sum of the factorial series
result = aggregate(lambda(sum: 1.0b, new, sum + new), series)

# And echo the 100th position
echo(result[100])
```

This prints:

```
2.7182818284590452353602874713526625341873350612088928537977623225062688778363522264614906840877992143877605493458308527028670580627857829570187638863945686571794897466427318126273892787895838400
```

## 99 Bottles of Beer

In keeping with tradition, a tiny script that generates the lyrics for "99 bottles of beer":

```python
counter = limit(99, generate(lambda(x: 99, x - 1)))

lyrics = merge("Aaaand " + counter + " bottle(s) of beer on the wall, " + counter + " bottle(s) of beer 
	Take one down and pass it around, " + (counter - 1) + " bottle(s) of beer on the wall.",
	"No more bottles of beer on the wall, no more bottles of beer.
	Go to the store and buy some more, 99 bottles of beer on the wall.")

echo(join("\n\n", lyrics))
```

This prints out (skipped some parts):

```
Aaaand 99 bottle(s) of beer on the wall, 99 bottle(s) of beer
Take one down and pass it around, 98 bottle(s) of beer on the wall.

Aaaand 98 bottle(s) of beer on the wall, 98 bottle(s) of beer
Take one down and pass it around, 97 bottle(s) of beer on the wall.

Aaaand 97 bottle(s) of beer on the wall, 97 bottle(s) of beer
Take one down and pass it around, 96 bottle(s) of beer on the wall.

...

Aaaand 4 bottle(s) of beer on the wall, 4 bottle(s) of beer
Take one down and pass it around, 3 bottle(s) of beer on the wall.

Aaaand 3 bottle(s) of beer on the wall, 3 bottle(s) of beer
Take one down and pass it around, 2 bottle(s) of beer on the wall.

Aaaand 2 bottle(s) of beer on the wall, 2 bottle(s) of beer
Take one down and pass it around, 1 bottle(s) of beer on the wall.

Aaaand 1 bottle(s) of beer on the wall, 1 bottle(s) of beer
Take one down and pass it around, 0 bottle(s) of beer on the wall.

No more bottles of beer on the wall, no more bottles of beer.
Go to the store and buy some more, 99 bottles of beer on the wall.
```

## Birthday Paradox

We can calculate the birthday paradox:

```python
# Create a counter for the days
days = until(lambda(x, x <= 0), generate(lambda(x: 365.0b, x - 1)))

# Calculate the chance
paradox = 1.0b - aggregate(lambda(total: 1.0b, new, total * (new / 365)), days)
```

When using the [visual console|$Visual Console.md], you can draw the paradox:

```python
display(area("Birthday Paradox", "Amount of People", "Chance", 
	plot("Paradox", paradox), 
	plot("Inverse", 1.0b - paradox)))
```

It looks like this:

[:.resources/birthday_paradox.png]

## Timeout

You can create a timeout function that works more or less like ``setTimeout`` in javascript but has the option to repeat itself, for example:

```python
timeout = lambda
	lambda executor ?= null
	integer amount ?= null
	boolean repeat ?= false
	boolean start ?= true
	
	helper = lambda
		while (!aborted())
			sleep(amount)
			if (!aborted())
				executor()
				if (!repeat)
					break
	@return
	return = lambda(run(helper))
	if (start)
		return = return()
```

You can then run:

```python
timer = timeout(lambda(echo("Run at: " + date())), 1000, true)
```

This will print:

```
Run at: 2016-08-01T14:27:02.103
Run at: 2016-08-01T14:27:03.104
Run at: 2016-08-01T14:27:04.104
Run at: 2016-08-01T14:27:05.105
Run at: 2016-08-01T14:27:06.106
Run at: 2016-08-01T14:27:07.107
Run at: 2016-08-01T14:27:08.108
Run at: 2016-08-01T14:27:09.109
Run at: 2016-08-01T14:27:10.111
Run at: 2016-08-01T14:27:11.111
Run at: 2016-08-01T14:27:12.112
```

Because it by default returns a future, you can abort it:

```python
abort(timer)
```

## Sum

Calculates the sum of a series:

```python
sum = lambda(series, last(aggregate(lambda(sum: 0, new, new + sum), series)))
```

## Not

Inverts a series of booleans

```python
not = lambda(series, derive(lambda(entry, !entry), series))
```

## Retain

Takes a series of data and a series of booleans and only retains the data if the boolean is true. It uses the ``zip()`` method outlined in the [series|$Series.md] documentation.

```python
retain = lambda
	[] series ?= null
	[] booleans ?= null
	# retain only the entries that have a boolean true
	retained = filter(lambda(entry, entry[1]), zip(series, booleans))
	# strip the boolean value and simply return the item
	@return
	unwrapped = derive(lambda(entry, entry[0]), retained)
```

For example you could do:

```python
series = series("test1", "test2", "something", "test3", "else")
echo(retain(series, series ~ "test.*"))
```

This would print:

```
[test1, test2, test3]
```

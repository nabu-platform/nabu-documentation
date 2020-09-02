# Math

## Methods

Glue supports most basic math commands you would expect. The following all support series:

- abs
- ceil
- floor
- sqrt
- cos
- cosh
- sin
- sinh
- tan
- tanh
- acos
- asin
- atan
- cbrt
- log
- log10
- degrees
- radians

There are a few methods that have fixed input values:

- atan2(y, x)
- hypot(x, y)

And a few that simply return a value:

- random
- pi
- e

Though for the latter two you can optionally pass in a boolean indicating that you want to use big numbers, e.g.:

```python
echo(pi())
echo(pi(big: true))
```

Prints out (ellipsis added for readability):

```
3.141592653589793
3.141592653589793238462643383279502884197169399375105820974944592307816406286208998...
```

## Big Numbers

There is inherent support in glue for big numbers, there are two additional data types to the ones listed in the introduction:

- **bigInteger**: an arbitrarily large integer number
- **bigDecimal**: an arbitrarily large decimal number

They work mostly as their smaller counterparts except that they are obviously slower. It is also important to note that combined with the narrow typing it is important to annotate your numbers as being big for the parser to pick it up correctly, for example:

```python
bigInteger a = 5
```

After this line of code, the variable a will indeed contain a big integer with value 5 **but** the parser will have first parsed the number 5 as a long and then converted it to the big integer.

If you want to express large numbers in glue, you need to add "b" at the end, for example:

```python
a = 5b
```

The parser will actually parse 5 as a big integer instead of a long which means a will automatically be a big integer.

As an example, let's take this code:

```python
a = 200
echo(a ** a)
```

This will print out: ``2147483647``

```python
decimal a = 200
echo(a ** a)
```

This will print out: `Infinity`

If we actually want to perform such large calculations, we could do:

```python
a = 200b
echo(a ** a)
```

Which yields us (linefeeds added for readability):

```
160693804425899027554196209234116260252220299378279283530137600000000000000000000000000000
000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
00000000000
```


### Arbitrary precision

Suppose we run this:

```python
echo(4.0 / 24)
```

We get: ``0.16666666666666666``

However if we use big decimals, this number would have a non-terminating amount of 6 after the decimal point so we need to make _some_ decision on where to round it off. By default the math context used for numbers is `DECIMAL128` which means a scale of 34 with rounding mode `HALF_EVEN`.

This means if we run the above calculation using big numbers ``echo(4.0b / 24)`` we get: ``0.1666666666666666666666666666666667``.

You can however update the math context used:

```
rounding(100)
echo(4.0b / 24)
rounding(5, "DOWN")
echo(4.0b / 24)
```

This prints:

```
0.1666666666666666666666666666666666666666666666666666666666666666666666666666666666666666666666666667
0.16666
```

Note that the math context is set in a threadlocal so in multithreaded mode you may want to set it for every thread.

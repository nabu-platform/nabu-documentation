# Data Mutation

Glue strived for immutable data structures for a long time. This provides guarantees towards lambda's enclosed state, parallelization and other things that might depend on data not changing after definition. As an alternative to directly mutating data, glue contains a lot of ways to derive new data from existing data.

However, when writing simple scripts that don't really care about parallelism or other nuanced complexities, the immutability make certain actions more difficult than they should be. For that reason, mutability was added to glue.

As a simple example of something that did not work before but does now:

```python
a = structure(test: "test1")
a/test = "test2"
```

When you run this script, ``a/test`` will contain "test2".

You can also use the query engine to perform more complex mutations:

```python
a = structure(test: "for")
b = structure(test: "example")
a = series(a, b)

a[test="example"]/test = "nothing"
```

When you run this, glue will find all the instances in the series that have ``test`` set to ``example`` and update them to ``nothing``.

You can combine this with more complex logic:

```python
b = series(1, 2)
b[2] = 3

integer c = random() * 5
b[when(c >= 3, 3, 4)] = 4
```

Mutating series does force a calculation at that point. If you have infinite series and perform a direct mutation on them, this will become problematic..

```python
x = series(1, 2)
y = series(3, 4)
z = merge(x, y)
# Force a resolving into a new list where we can override the given index
z[4] = 10
```

```python
fibonacci = generate(lambda(t2: 0, t1: 1, t2 + t1))
# Without this line of code, the script would hang indefinitely trying to resolve an infinite list
fibonacci = limit(10, fibonacci)
fibonacci[10] = "woho"
echo(fibonacci)
```

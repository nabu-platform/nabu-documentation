# Querying

## Basic Queries

Glue has native support for queries that are have a syntax somewhere between xpath and java. There are some fundamental differences with xpath though:

- it works on all types that are exposed through the types API: java beans, xml, json,...
- the operators are the default glue operators (instead of the more xpath like "and", "or",...) and follow the same rules (including overloading)
- you can use any method known to glue, including lambda's

For example let's use this as our people generator:

```python
counter = generate(lambda(x: 0, x + 1))
people = derive(lambda(counter, structure(firstName: "John" + counter, lastName: "Smith", index: counter)), counter)
```

This is an infinite series so we limit it:

```python
people = limit(100, people)
```

Now we want to find the first name of everyone with an index greater than 90:

```python
echo(users[index > 90]/firstName)
```

This prints:

```
[John91, John92, John93, John94, John95, John96, John97, John98, John99]
```

We can wrap random logic in the query, for example:

```python
calculator = lambda(x, x + 1)
echo(users[calculator(index) > 90])
```

Here we get one more user that fits the criteria:

```
[John90, John91, John92, John93, John94, John95, John96, John97, John98, John99]
```

We can also offload the logic entirely:

```python
selector = lambda(person, person/index > 90)
echo(users[selector($this) == true]/firstName)
```

Here we get the same result as in the beginning:

```
[John91, John92, John93, John94, John95, John96, John97, John98, John99]
```

## Nested Queries

We can also run nested queries, for example let's run a query on a JSON object:

```python
people = json.objectify('[
		{ "firstName": "John", "lastName": "Smith", "age": 30, "pets": [{ "name": "Rex", "type": "dog" }, { "name": "Delilah", "type": "goldfish" }] },
		{ "firstName": "Bob", "lastName": "Smith", "age": 35, "pets": [{ "name": "Max", "type": "cat" }, { "name": "Spike", "type": "dog" }] }
	]')/array

echo("Pets: " + people[age > 30]/pets[type = "dog"]/name)
```

This will print out the name of all the dogs that belong to people over 30 years of age:

```
Pets: [Spike]
```

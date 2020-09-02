# Strings

In glue v2 everything revolves around series of data and -if possible- lazy execution on top of them. Following this logic most string methods will allow you to pass in either a single item or a series of items and return respectively a single result set or a series of result sets. Unless specifically indicated otherwise, the last parameter can be either a single item or a series of items.

## Replace

The venerable "replace" that exists in every language and replaces parts of a string (based on a regex):

```python
echo(replace(" ", ";", "this is a test"))
```

This would predictably output:

```
this;is;a;test
```

However you can also pass in a series:

```python
series = series("this is a test", "this is another test", "another test right here!")
echo(replace(" ", ",", series))
```

This would output:

```
[this;is;a;test, this;is;another;test, another;test;right;here!]
```

Like most methods you don't have to create an actual series as the method allows for any number of arguments at the end (note that the linefeeds are only for readability):

```python
result = replace(" ", ",", 
		"this is a test", 
		"this is another test", 
		"another test right here!")

echo(result)
```

This prints the same as the previous.

### Quoting

When using regexes, there are a lot of pre-defined characters that can not occur in the string. Some languages provide a replace that does not use regexes, but in glue there are two helper methods that you can use:

- ``quoteRegex()``: will quote a string that you want to use as regex (or part of one) to be taken literally instead of interpreted
- ``quoteReplacement()``: will quote a string that you want to use as a replacement (or part of one)

### Advanced

Replace has an advanced mode where you pass in a lambda as second parameter instead of a string. The problem this solves is if you want to replace multiple hits with different replacements, even if the hits are exactly the same.

Take this simple example:

```
test = "aaa"

counter = 0
alphabet = series("a", "b", "c", "d", "e", "f", "g")

replacer = method
	text ?= null
	@return
	result = alphabet[/counter]
	@persist
	counter = counter + 1

echo(replace("a", replacer, test))
```

This outputs ``abc``.

As another example suppose we have a large html document and we want to add `id` attributes to all buttons and links if it doesn't have one yet.

Traditionally you could do something like:

```python
counter = 0
for (entry : find("(?s)<(?:a|button)\b[^>]+", content))
	if (entry !~ "(?s).*\bid[\s]*=.*")
		content = replace(quoteRegex(entry), quoteReplacement(entry 
			+ ' id="element' + counter + '"'), content)
		counter = counter + 1
```

Apart from not being the most performant solution, this also has the problem that if the entries we find are identical, they are hard to pinpoint with replacement. You can try to include an offset in the replacement or a combination of substrings with replaceFirst etc but it is problematic to do this properly.

Consider this lambda solution:

```python
counter = 0
injecter = method
	@return
	string content ?= null
	if (content !~ "(?s).*\bid[\s]*=.*")
		content = content + ' id="element' + counter + '"'
		@persist
		counter = counter + 1

content = replace("(?s)<(?:a|button)\b[^>]+", injecter, content)
```

Internally the ``replace()`` will loop over the entire string just once and each match is given to the lambda who can return the replacement. By using a stateful method instead of a stateless lambda we can persist the updated counter in the outer scope.

## Find

```python
result = find("[\s]+is.*", 
		"this is a test", 
		"this is another test", 
		"another test right here!")

derivation = derive(lambda(result, "Found result: " + result), result)

for (single : derivation)
	echo(single)
```

This will print:

```
Found result:  is a test
Found result:  is another test
```

Note that the find is an "exploding" function in that each entry in the series can return multiple entries in the resulting series. This type of function can not be resolved lazily as many operations need the final series to perform their logic.

## Padding

You can pad a string on the left:

```python
result = padLeft("0", 10, 123)
echo(result)
```

This will print out: `0000000123`

You can once again pad a series as well:

```python
result = padLeft("0", 10, 123, 234, 345)
echo(result)
```

This will print: ``[0000000123, 0000000234, 0000000345]``

You can also pad a string on the right:

```python
result = padRight("0", 5, 123)
echo(result)
```

This will print out: `12300`

## Upper/Lower

You can upper case or lower case a string using the respective methods:

```python
result = upper("This is a sentence")
echo(series(result, lower(result)))
```

This prints out: ``[THIS IS A SENTENCE, this is a sentence]``

## Substring

You can select part of a string using the substring method:

```python
string = "This is my string"
echo(substring(0, 4, string))
echo(substring(5, string: string))
echo(substring(stop: 4, string))
```

This prints out:

```
This
is my string
This
```

## Replace

You can replace parts of a string using a regex:

```python
string = "This is my string"
echo(replace("my", "your", string))
```

This will print: ``This is your string``

Another example of series usage:

```python
echo(replace("my", "your", "This is my string", "This is my example"))
```

This will print:

```
[This is your string, This is your example]
```

### Escaping

Note that because the ``replace()`` uses the java regex replace, there are some caveats:

```python
echo(replace("'", "\\'", "test ' test"))
```

This will print out:

```
test ' test
```

This is the same as you would get from java:

```java
System.out.println("test ' test".replaceAll("'", "\\'"));
```

This is because the escape character has special meaning in the replacement as well. To actually replace it we need an additional slash:

```python
echo(replace("'", "\\\'", "test ' test"))
```

This will output the desired result:

```
test \' test
```

Additionally suppose you want to print out: ``test "\" test``

You can either use the single quote:

```python
echo('test "\\" test')
```

Or if you insist on using the double quote for string demarcation as well you could do this:

```python
echo("test \"" + "\\" + "\" test")
```

## Join

You can join together a number of strings into a single one. Note that this is one of the few methods that never returns a series, even if given one as it implodes the series into a single value. For example:

```python
echo(join(" ", "this", "is", "an", "example"))
```

This will print: ``this is an example``

## Split

You can split a string into substrings based on a regex:

```python
echo(split(" ", "this is an example", "this is another example"))
```

This prints out one single series that contains: ``[this, is, an, example, this, is, another, example]``

## Trim

You can trim the whitespace off a string using:

```python
echo(trim("   remove this whitespace!   "))
```

Which will print -to noone's surprise-: ``remove this whitespace!``

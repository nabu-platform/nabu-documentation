@tag date
@icon date
@image /resources/images/date.png
@description Everything you need to know about date handling in glue

# Dates

## Current

The easiest method for dates is the ``date()`` method that without parameters allows you to get the current timestamp.

If you give it a string parameter, it will try a number of different formats to parse it as a valid date.

```python
echo(date())
echo(date("2016/01/31"))
```

This will print:

```
2016-07-03T22:29:57.347+02:00
2016-01-31T00:00:00.000+01:00
```

Note that you can also pass in multiple dates which will yield a series:

```python
echo(date("2016/01/31", "2016/02/03"))
```

Prints:

```
[Sun Jan 31 00:00:00 CET 2016, Wed Feb 03 00:00:00 CET 2016]
```

## Parse

If you want more control over the parsing of the date, you can use the ``parse()`` method. The parse method takes the following parameters:

- :format: the format to use to parse the date, you can check the [java documentation](https://docs.oracle.com/javase/7/docs/api/java/text/SimpleDateFormat.html) for the allowed formats
- :timezone: the timezone used for parsing
- :language: the language used for parsing, useful for those rare dates where a translated word is shown
- :date: one or more dates to be parsed

Note that all except the last parameter are optional. This may seem strange but it is done to be consistent with the rest of glue where the series is the last parameter.

This is probably a good time to remind you that glue supports named parameters so you can target the last parameter if you are not interested in one or more of the others.

```python
echo(parse("yyyy/MM/dd", date: "2016/01/31", "2016/02/03"))
```

This will print out the same as above:

```
[Sun Jan 31 00:00:00 CET 2016, Wed Feb 03 00:00:00 CET 2016]
```

## Format

The format takes exactly the same parameters as the parse except that the the last parameter should be of type string:

```python
echo(format("yyyy/MM/dd", date: date()))
```

This will print:

```
2016/07/03
```

## Operator Overloading

Dates have operator overloading for + and -. They allow you to add and substract numbers which are interpreted as milliseconds but they also allow you to add and substract strings that have a specific format: ``<number> <descriptor>``. The descriptor defines how the number is interpreted, currently the rules are:

- starts with `y`: year
- starts with `min`: minutes
- starts with `m`: months
- starts with `d`: days
- starts with `h`: hours
- starts with `s`: seconds
- starts with `mil` or equals `ms`: milliseconds

So you could do:

```python
echo(date() + 3600 * 1000)
echo(date() + "1 day" + "30 minutes")
```

It will print out:

```
2016-07-03T23:44:27.673+02:00
2016-07-04T23:13:39.751+02:00
```

## Range Generation

To generate a range of dates, for example every hour, use the series method `generate`:

```python
series = generate(lambda(x: date("2016/01/01"), x + "1 hour"))
echo(limit(4, series))
```

This will print:

```
[Fri Jan 01 00:00:00 CET 2016, Fri Jan 01 01:00:00 CET 2016, Fri Jan 01 02:00:00 CET 2016, Fri Jan 01 03:00:00 CET 2016]
```

# System Integration

Glue integrates completely with the commandline of the OS it's running on. This is obviously platform specific (as opposed to most everything else in glue) so be careful when distributing such scripts. 

Basically any method you call with the namespace ``system`` will be redirected to the OS, for example (examples based on linux but broadly applicable):

```python
echo(system.ps())
```

This will print out a snapshot of the current processes.

## Parameters

We can add any parameters to the method call that will be passed to the commandline, for example:

```python
echo(system.ps("-ef"))
```

Which parameters exist is of course dependent on the command. 

## File Traversal

There is special handling for file traversal related methods like ``cd`` and ``pwd``, they are intercepted by the system and internally used to scope future operations from the current point. Some other operations like file operations also refer to this central location if relative paths are given.

For example:

```python
echo("Browsing to: " + system.cd("/tmp"))
write("test.txt", "some content here")
echo(system.cat("test.txt"))
```

This will print:

```
Browsing to: /tmp
some content here
```

## Input Streaming

You can stream content to commands much like a pipe would by prefixing the content with ``1<``:

```python
system.cd("/tmp")
echo(system.grep("test", 1< system.ls()))
```

This prints out:

```
test.glue
test.txt
```

This allows complex chaining not only of OS commands but obviously also of anything else that can be expressed in glue.

You can also add multiple inputs:

```python
system.cd("/tmp")
echo(system.grep("test", 1< system.ls(), 1< "this is a test too!"))
```

This prints out:

```
test.glue
test.txt
this is a test too!
```

## I/O Redirection

By default the output of the command you run is buffered while it is running and returned as the return value of the method at the end.

In some cases however you want the output (and potentially further user input) to be redirected to the standard system streams available in java, you can do this by setting:

```python
system.cd("/tmp")
system.ls(1> system)
```

This will echo the result of the ls command through the main system stream.

### Combining

It is possible to combine this with input streaming however it is important to note that once you combine I/O redirection with input streaming you can no longer accept input from the user on the commandline as the input stream is not redirected.

For example:

```python
system.grep("test", 1< system.ls(), 1> system)
```

The ls command has its I/O in memory (not redirected to standard input/output) and returns the result as its value. This result is fed into the grep command of which the output is redirected to the standard output.

## Bash

In the above examples we are using glue syntax to call system commands, we can however also run bash scripts in general:

```
script = "cd /tmp
	ls | grep test"

echo(bash(script))
```

This outputs:

```
test.glue
test.txt
```

Note that the cd command is not intercepted by the system here, so following glue commands will still be pointed at the original directory you were in.

## Input

You can also request input from the user when running scripts on the console, for example:

```
integer a = input("Pick a number, any number: ")
integer b = input("Pick a secret number: ", true)
echo("The result is...", a + b)
```

The boolean set when requesting the secret number means you will not see what you type (this is generally used for entering passwords), so this outputs:

```
Pick a number, any number: 1
Pick a secret number: 
The result is....
3
```

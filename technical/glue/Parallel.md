# Parallel

By default everything executes on a single thread: the main thread. The method ``series.resolve()`` already has an optimized way of resolving series in parallel where possible but it is not a generic tool for parallellism, it only simplifies a common usecase.

True parallellism is also possible in glue and it takes two methods:

- **parallel.run(lambda, params...)**: runs the given lambda with the given params (if any). The run method returns a future object that can track the state of the calculation. If any parameter you pass in is a future, the run method will wait (asynchronously) for those future to be resolved.
- **parallel.wait(futures...)**: waits for the given futures to be resolved and returns the result of said futures as a series in the order you passed the futures in. If you give no futures at all, it will wait for all asynchronous actions to complete but will have no return value.

## Basic Example

We could offload some calculation to other threads:

```python
calculator = lambda(a, a * a)

futures = series(
	run(calculator, 1),
	run(calculator, 2),
	run(calculator, 3),
	run(calculator, 4))

echo(wait(futures))
```

This will output: ``[1, 4, 9, 16]``

## Long Running Example

The above example returns immediately because while we are working in a multithreaded way, each thread finishes very rapidly, let's add a 5 second sleep to actually notice the multithreaded behavior:

```python
calculator = sequence
	amount ?= null
	started = date()
	sleep(5000)
	@return
	result = structure(amount: amount * amount, started: started, stopped: date())

future1 = run(calculator, 1)
future2 = run(calculator, 2)
future3 = run(calculator, 3)
future4 = run(calculator, 4)

echo("Waiting for the results - " + date())

for (result : wait(future1, future2, future3, future4))
	echo("Ran from " + result/started + " till " + result/stopped + ": " + result/amount)
```

This will print:

```
Waiting for the results - 2016-07-05T22:10:36.191
Ran from 2016-07-05T22:10:36.186 till 2016-07-05T22:10:41.193: 1
Ran from 2016-07-05T22:10:36.189 till 2016-07-05T22:10:41.193: 4
Ran from 2016-07-05T22:10:36.189 till 2016-07-05T22:10:41.193: 9
Ran from 2016-07-05T22:10:36.188 till 2016-07-05T22:10:41.193: 16
```

## Sequential Runs

We can update the above example to make sure the last calculator takes the previous ones into account:

```python
calculator = sequence
	[] amount ?= null
	started = date()
	sleep(2500)
	@return
	sum = 0
	for (value : amount)
		sum = sum + value

future1 = run(calculator, 1, 2)
future2 = run(calculator, 2, 3)
future3 = run(calculator, 3, 4)
future4 = run(calculator, future1, future2, future3)

echo("Waiting for the results - " + date())
echo("Received: " + wait(future4) + " - " + date())
```

Here we have a sum lambda that takes a variable amount of input parameters. We start three sum calculations in separate threads and have the fourth wait for all three and do a final sum and return that. This script outputs:

```python
Waiting for the results - 2016-07-05T22:21:50.488
Received: 15 - 2016-07-05T22:21:55.498
```

## Pipelining

You can use function composition to pipeline results easily in a single thread:

```python
sum = lambda(a, a + a)
multiply = lambda(a, a * a)
increment = lambda(a, a + 1)

calculator = compose(sum, multiply, increment, multiply)

future1 = run(calculator, 1)
future2 = run(calculator, 2)

echo("Waiting for the results - " + date())
echo("Received: " + wait(future1, future2) + " - " + date())
```

This prints out:

```
Waiting for the results - 2016-07-05T22:29:53.410
Received: [25, 289] - 2016-07-05T22:29:53.434
```

## Abort

You can abort a parallel run though it is not always guaranteed to work. It will try to cancel the task altogether (if it is not yet running) and if it is running it will try to abort as soon as possible.

```python
future = ...
abort(future)
```

If you pass in no arguments, it will abort the current script in its entirety.

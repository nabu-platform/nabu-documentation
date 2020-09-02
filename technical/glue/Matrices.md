@image /resources/images/matrix.png
@icon matrix
@description N-dimensional matrices, depth calculation, rotation,...

# Matrices

Matrices in glue are simply series with more than 1 dimension. As such all series methods still apply to matrices.

## Create

You can easily create a matrix by creating a series of series:

```python
matrix = series(
	series(1, 2, 3),
	series(2, 3, 4))
# Print the entire matrix
echo(matrix)
# Print the first row
echo(matrix[0])
# Print the second element of the first row
echo(matrix[0][1])
```

This prints:

```
[[1, 2, 3], [2, 3, 4]]
[1, 2, 3]
2
```

There is currently no builder method to create matrices but you can easily create a lambda to generate n-dimensional matrices:

```python
matrix = sequence
	value ?= 0
	integer [] dimensions ?= null
	dimensions = reverse(dimensions)
	@return
	matrix = null
	for (dimension : dimensions)
		if ($index == 0)
			matrix = series(resolve(limit(dimension, repeat(value))))
		else if ($index == size(dimensions) - 1)
			matrix = resolve(limit(dimension, repeat(matrix)))
		else
			matrix = series(resolve(limit(dimension, repeat(matrix))))
			
echo(matrix(value: 1, dimensions: 2, 3, 4))
```

This prints out:

```
[[[1, 1, 1, 1], [1, 1, 1, 1], [1, 1, 1, 1]], [[1, 1, 1, 1], [1, 1, 1, 1], [1, 1, 1, 1]]]
```

## Depth

You can get the number of dimensions of any series:

```python
matrix = series(
	series(1, 2, 3),
	series(2, 3, 4))
echo(depth(matrix))
```

This prints: `2`.

## Dimensions

You can also request the dimensions of a series, this gives you the size of each dimension in a series.

```python
matrix = series(
	series(1, 2, 3),
	series(2, 3, 4))
echo(dimensions(matrix))
```

Prints: ``[2, 3]`` because this matrix has two rows and per row 3 elements.

## Rotate

There is no dedicated function for matrix rotation but it is trivial to achieve with the ``derive()`` method:

```python
rotate = lambda
	[] series ?= null
	helper = lambda
		@return
		[] series ?= null
	@return
	result = derive(helper, unwrap(series))

value = matrix(value: 1, dimensions: 2, 3)
rotated = rotate(value)
echo(dimensions(value), dimensions(rotated))
```

This will print out:

```
[2, 3]
[3, 2]
```

---
title: Transform data with mathematic operations
seotitle: Transform data with mathematic operations in Flux
description: This guide describes how to use Flux to transform data with mathematic operations.
v2.0/tags: [math, flux]
menu:
  v2_0:
    name: Transform data with math
    parent: How-to guides
weight: 209
---

[Flux](/v2.0/reference/flux), InfluxData's data scripting and query language,
supports mathematic expressions in data transformations.
This article describes how to use [Flux arithmetic operators](/v2.0/reference/flux/language/operators/#arithmetic-operators)
to "map" over data and transform values using mathematic operations.

If you're just getting started with Flux queries, check out the following:

- [Get started with Flux](/v2.0/query-data/get-started/) for a conceptual overview of Flux and parts of a Flux query.
- [Execute queries](/v2.0/query-data/execute-queries/) to discover a variety of ways to run your queries.

##### Basic mathematic operations
```js
// Examples executed using the Flux REPL
> 9 + 9
18
> 22 - 14
8
> 6 * 5
30
> 21 / 7
3
```

<p style="font-size:.85rem;font-style:italic;margin-top:-2rem;">See <a href="/v2.0/reference/cli/influx/repl">Flux read-eval-print-loop (REPL)</a>.</p>

{{% note %}}
#### Operands must be the same type
Operands in Flux mathematic operations must be the same data type.
For example, integers cannot be used in operations with floats.
Otherwise, you will get an error similar to:

```
Error: type error: float != int
```

To convert operands to the same type, use [type-conversion functions](/v2.0/reference/flux/stdlib/built-in/transformations/type-conversions/)
or manually format operands.
The operand data type determines the output data type.
For example:

```js
100 // Parsed as an integer
100.0 // Parsed as a float

// Example evaluations
> 20 / 8
2

> 20.0 / 8.0
2.5
```
{{% /note %}}

## Custom mathematic functions
Flux lets you [create custom functions](/v2.0/query-data/guides/custom-functions) that use mathematic operations.
View the examples below.

###### Custom multiplication function
```js
multiply = (x, y) => x * y

multiply(x: 10, y: 12)
// Returns 120
```

###### Custom percentage function
```js
percent = (sample, total) => (sample / total) * 100.0

percent(sample: 20.0, total: 80.0)
// Returns 25.0
```

### Transform values in a data stream
To transform multiple values in an input stream, your function needs to:

- [Handle piped-forward data](/v2.0/query-data/guides/custom-functions/#functions-that-manipulate-piped-forward-data).
- Use the [`map()` function](/v2.0/reference/flux/stdlib/built-in/transformations/map) to iterate over each row.

The example `multiplyByX()` function below includes:

- A `tables` parameter that represents the input data stream (`<-`).
- An `x` parameter which is the number by which values in the `_value` column are multiplied.
- A `map()` function that iterates over each row in the input stream.
  It uses the `with` operator to preserve existing columns in each row.
  It also multiples the `_value` column by `x`.

```js
multiplyByX = (x, tables=<-) =>
  tables
    |> map(fn: (r) => ({
        r with
        _value: r._value * x
      })
    )

data
  |> multiplyByX(x: 10)
```

## Examples

### Convert bytes to gigabytes
To convert active memory from bytes to gigabytes (GB), divide the `active` field
in the `mem` measurement by 1,073,741,824.

The `map()` function iterates over each row in the piped-forward data and defines
a new `_value` by dividing the original `_value` by 1073741824.

```js
from(bucket: "example-bucket")
  |> range(start: -10m)
  |> filter(fn: (r) =>
    r._measurement == "mem" and
    r._field == "active"
  )
  |> map(fn: (r) => ({
      r with
      _value: r._value / 1073741824
    })
  )
```

You could turn that same calculation into a function:

```js
bytesToGB = (tables=<-) =>
  tables
    |> map(fn: (r) => ({
        r with
        _value: r._value / 1073741824
      })
    )

data
  |> bytesToGB()
```

#### Include partial gigabytes
Because the original metric (bytes) is an integer, the output of the operation is an integer and does not include partial GBs.
To calculate partial GBs, convert the `_value` column and its values to floats using the
[`float()` function](/v2.0/reference/flux/stdlib/built-in/transformations/type-conversions/float)
and format the denominator in the division operation as a float.

```js
bytesToGB = (tables=<-) =>
  tables
    |> map(fn: (r) => ({
        r with
        _value: float(v: r._value) / 1073741824.0
      })
    )
```

### Calculate a percentage
To calculate a percentage, use simple division, then multiply the result by 100.

{{% note %}}
Operands in percentage calculations should always be floats.
{{% /note %}}

```js
> 1.0 / 4.0 * 100.0
25.0
```

#### User vs system CPU usage
The example below calculates the percentage of total CPU used by the `user` vs the `system`.

{{< code-tabs-wrapper >}}
{{% code-tabs %}}
[Comments](#)
[No Comments](#)
{{% /code-tabs %}}

{{% code-tab-content %}}
```js
// Custom function that converts usage_user and
// usage_system columns to floats
usageToFloat = (tables=<-) =>
  tables
    |> map(fn: (r) => ({
      _time: r._time,
      usage_user: float(v: r.usage_user),
      usage_system: float(v: r.usage_system)
      })
    )

// Define the data source and filter user and system CPU usage
// from 'cpu-total' in the 'cpu' measurement
from(bucket: "example-bucket")
  |> range(start: -1h)
  |> filter(fn: (r) =>
    r._measurement == "cpu" and
    r._field == "usage_user" or
    r._field == "usage_system" and
    r.cpu == "cpu-total"
  )

  // Pivot the output tables so usage_user and usage_system are in each row
  |> pivot(rowKey: ["_time"], columnKey: ["_field"], valueColumn: "_value")

  // Convert usage_user and usage_system to floats
  |> usageToFloat()

  // Map over each row and calculate the percentage of
  // CPU used by the user vs the system
  |> map(fn: (r) => ({
      // Preserve existing columns in each row
      r with
      usage_user: r.usage_user / (r.usage_user + r.usage_system) * 100.0,
      usage_system: r.usage_system / (r.usage_user +  r.usage_system) * 100.0
    })
  )
```
{{% /code-tab-content %}}

{{% code-tab-content %}}
```js
usageToFloat = (tables=<-) =>
  tables
    |> map(fn: (r) => ({
      _time: r._time,
      usage_user: float(v: r.usage_user),
      usage_system: float(v: r.usage_system)
      })
    )

from(bucket: "example-bucket")
  |> range(start: timeRangeStart, stop: timeRangeStop)
  |> filter(fn: (r) =>
    r._measurement == "cpu" and
    r._field == "usage_user" or
    r._field == "usage_system" and
    r.cpu == "cpu-total"
  )
  |> pivot(rowKey: ["_time"], columnKey: ["_field"], valueColumn: "_value")
  |> usageToFloat()
  |> map(fn: (r) => ({
      r with
      usage_user: r.usage_user / (r.usage_user + r.usage_system) * 100.0,
      usage_system: r.usage_system / (r.usage_user +  r.usage_system) * 100.0
    })
  )
```
{{% /code-tab-content %}}
{{< /code-tabs-wrapper >}}

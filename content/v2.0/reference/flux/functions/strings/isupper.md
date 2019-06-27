---
title: strings.isUpper() function
description: The strings.isUpper() function indicates if a single character string is uppercase.
menu:
  v2_0_ref:
    name: strings.isUpper
    parent: Strings
weight: 301
related:
  - /v2.0/reference/flux/functions/strings/islower
---

The `strings.isUpper()` function indicates if a single character string is uppercase.

_**Output data type:** Boolean_

```js
import "strings"

strings.isUpper(v: "A FLUX OF FOXES")

// returns true
```

## Parameters

### v
The single-character string value to test.

_**Data type:** String_

## Examples

###### Filter by columns with single-letter uppercase values
```js
import "strings"

data
  |> filter(fn: (r) => strings.isUpper(v: r.host))
```
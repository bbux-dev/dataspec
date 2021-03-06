Field Spec Definitions
========================


1. [Quick Reference](#Quick_Reference)
1. [Overview](#Overview)
1. [Field Spec Structure](#Field_Spec_Structure)
    1. [The Full Format](#The_full_format.)
    1. [Values Shorthand](#Values_Shorthand)
    1. [Inline Key Type Shorthand](#Inline_Key_Type_Shorthand)
    1. [Inline Key Config Shorthad](#Inline_Key_Config_Shorthad)
1. [Spec Configuration](#Spec_Configuration)
    1. [Common Configurations](#Common_Configurations)
1. [Field Spec Types](#Field_Spec_Types)
    1. [Values](#Values)
        1. [Constant Values](#Constant_Values)
        1. [List Values](#List_Values)
        1. [Weighted Values](#Weighted_Values)
        1. [Sample Mode](#Sample_Mode)
    1. [Combine](#Combine)
    1. [Date](#Date)
    1. [Range](#Range)
    1. [Uuid](#Uuid)
    1. [Character Class](#CharClass)
       1. [Built In Classes](#SupportedClasses)
    1. [Unicode Ranges](#UnicodeRange)
    1. [Geo](#Geo)
    1. [IP Addresses](#IP_Addresses)
        1. [Precise CIDR Addresses](#Precise_IP)
    1. [Weighted Ref](#Weighted_Ref)
    1. [Select List Subset](#Select_List_Subset)
        1. [Quoting Sublist Elements](#quoting_sublist)
    1. [CSV Data](#CSV_Data)
    1. [CSV Select](#CSV_Select)
    1. [nested](#Nested)

# <a name="Quick_Reference"></a>Quick Reference

| type                        | description                            | config params                |
|-----------------------------|----------------------------------------|------------------------------|
|[values](#Values)            | constant, list, or weighted dictionary |                              |
|[range](#Range)              | range of values                        |                              |
|[rand_range](#RandRange)     | random value in a range                |                              |
|[combine](#Combine)          | refs or fields                         | join_with                    |
|[combine-list](#CombineList) | list of lists of refs to combine       | join_with                    |
|[date](#Date)                | date strings                           | many see details below       |
|[date.iso](#Date)            | date strings in ISO8601 format no microseconds| many see details below|
|[date.iso.us](#Date)         | date strings in ISO8601 format w/ microseconds| many see details below|
|[uuid](#Uuid)                | generates valid uuid                   |                              |
|[char_class](#CharClass)     | generates strings from character classes| many see details below      |
|[unicode_range](#UnicodeRange)| generates strings from unicode ranges | many see details below       |
|[geo.lat](#Geo)              | generates decimal latitude             | start_lat,end_lat,precision  |
|[geo.long](#Geo)             | generates decimal longitude            | start_long,end_long,precision|
|[geo.pair](#Geo)             | generates long,lat pair                | join_with,start_lat,end_lat,start_long,end_long,precision|
|[ip/ipv4](#IP_Addresses)     | generates ip v4 addresses              | base, cidr /8,/16,/24 only   |
|[ip.precise](#IP_Addresses)  | generates ip v4 addresses              | cidr(required) i.e. 192.168.1.0/14 |
|[weightedref](#Weighted_Ref) | produces values from refs in weighted fashion |                       |
|[select_list_subset](#Select_List_Subset) | selects subset of fields that are combined to create the value for the field | join_with |
|[csv](#CSV_Data)             | Uses external csv file to supply data  | many see details below       |
|[csv_select](#CSV_Select)    | Efficient way to select multiple csv columns | many see details below |
|[nested](#Nested)            | For nested fields                      |                              |
# <a name="Overview"></a>Overview

Each field that should be generated needs a specification that describes the way the values for it should be created. We
refer to this as a Field Spec. The simplest type of Field Spec is a values spec. The main format of a values spec is a
list of values to use. By default, these values are rotated through incrementally. If the number of increments is larger
than the number of values in the list, the values start over from the beginning of the list. When combining values from
two values providers that are lists, they will be combined in incrementing order. i.e:

<details open>
  <summary>JSON Spec</summary>

```json
{
  "combine": {"type": "combine", "refs": ["ONE", "TWO"]},
  "refs": {
    "ONE": ["A", "B", "C"],
    "TWO": [1, 2, 3]
  }
}
```
</details>

<details>
  <summary>YAML Spec</summary>

```yaml
---
combine:
  type: combine
  refs: [ONE, TWO]
refs:
  ONE: [A, B, C]
  TWO: [1, 2, 3]
```
</details>
Will produce the values A1, B2, C3 continuously.

```shell script
dataspec -s ~/scratch/sample.json -i 7
A1
B2
C3
A1
B2
C3
A1
```

If an additional number is added to TWO, we now get 12 distinct values:

<details open>
  <summary>JSON Spec</summary>

```json
{
  "combine": {"type": "combine", "refs": ["ONE", "TWO"]},
  "refs": {
    "ONE": ["A", "B", "C"],
    "TWO": [1, 2, 3, 4]
  }
}
```
</details>

<details>
  <summary>YAML Spec</summary>

```yaml
---
combine:
  type: combine
  refs: [ONE, TWO]
refs:
  ONE: [A, B, C]
  TWO: [1, 2, 3 ,4]
```
</details>

```shell script
dataspec -s ~/scratch/sample.json -i 12 | sort
A1
A2
A3
A4
B1
B2
B3
B4
C1
C2
C3
C4
```

If we want our values to be generated randomly from the provided lists, we set the config param `sample` to true:

<details open>
  <summary>JSON Spec</summary>

```json
{
  "combine": {"type": "combine", "refs": ["ONE", "TWO"]},
  "refs": {
    "ONE?sample=true": ["A", "B", "C"],
    "TWO?sample=true": [1, 2, 3, 4]
  }
}
```
</details>

<details>
  <summary>YAML Spec</summary>

```yaml
---
combine:
  type: combine
  refs: [ONE, TWO]
refs:
  ONE?sample=true: [A, B, C]
  TWO?sample=true: [1, 2, 3 ,4]
```
</details>

# <a name="Field_Spec_Structure"></a>Field Spec Structure

There are several different ways to define a spec. There is the full spec format and a variety of short hand notations.

## <a name="The_full_format."></a>The Full Format.

The only required element is type. Each Type Handler requires different pieces of information. See the Field Type
reference below for details on each type.

```
{
  "type": "<the type>",
  "config": {
    "key1": "value1",
    ...
    "keyN": "valueN"
  },
  "data": ["the data"],
  "ref": "REF_POINTER_IF_USED",
  "refs": ["USES", "MORE", "THAN", "ONE"]
}
```

## <a name="Values_Shorthand"></a>Values Shorthand

The values type is very common and so has a shorthand notation. Below is an example full Field Spec for some values
types fields and the same spec in shorthand notation.

<details open>
  <summary>JSON Spec</summary>

```json
{
  "field1": {"type": "vaules", "data": [1, 2, 3, 4, 5]},
  "field2": {"type": "values", "data": {"A": 0.5, "B": 0.3, "C": 0.2}},
  "field3": {"type": "values", "data": "CONSTANT"}
}
```
</details>

<details>
  <summary>YAML Spec</summary>

```yaml
---
field1:
  type: vaules
  data: [1, 2, 3, 4, 5]
field2:
  type: values
  data:
    A: 0.5
    B: 0.3
    C: 0.2
field3:
  type: values
  data: CONSTANT
```
</details>

<details open>
  <summary>JSON Spec</summary>

```json
{
  "field1": [1, 2, 3, 4, 5],
  "field2": {"A": 0.5, "B": 0.3, "C": 0.2},
  "field3": "CONSTANT"
}
```
</details>

<details>
  <summary>YAML Spec</summary>

```yaml
---
field1: [1, 2, 3, 4, 5]
field2:
  A: 0.5
  B: 0.3
  C: 0.2
field3: CONSTANT
```
</details>

The value after the field name is just the value of the data element from the full Field Spec. Config params can be
added to the key using the URL syntax described below.

## <a name="Inline_Key_Type_Shorthand"></a>Inline Key Type Shorthand

Some specs lend themselves to being easily specified with few parameters. One short hand way to do this is the use a
colon in the key to specify the type after the field name. For example `{"id:uuid":{}}`. This says the field `id` is of
type `uuid` and has no further configuration. If no type is specified, the field is assumed to be a `values` type.

## <a name="Inline_Key_Config_Shorthad"></a>Inline Key Config Shorthad

It is also possible to specify configuration parameters in the key by using URL style parameters. For example.

<details open>
  <summary>JSON Spec</summary>

```json
{
  "network:ipv4?cidr=192.168.0.0/16": {}
}
```
</details>

<details>
  <summary>YAML Spec</summary>

```yaml
---
network:ipv4?cidr=192.168.0.0/16: {}
```
</details>

The `network` field is of type `ipv4` and the required `cidr` param is specified in the key.

# <a name="Spec_Configuration"></a>Spec Configuration

There are two ways to configure a spec. One is by providing a `config` element in the Field Spec and the other is by
using a URL parameter format in the key. For example, the following two fields will produce the same values:

<details open>
  <summary>JSON Spec</summary>

```json
{
  "ONE": {
    "type": "values",
    "data": [1, 2, 3],
    "config": {"prefix": "TEST", "suffix": "@DEMO"}
  },
  "TWO?prefix=TEST&suffix=@DEMO": [1, 2, 3]
}
```
</details>

<details>
  <summary>YAML Spec</summary>

```yaml
---
ONE:
  type: values
  config:
    prefix: TEST
    suffix: '@DEMO'
  data: [1, 2, 3]
TWO?prefix=TEST&suffix=@DEMO: [1, 2, 3]
```
</details>

# <a name="Common_Configurations"></a>Common Configurations

There are some configuration values that can be applied to all or a subset of types. These are listed below

| key   | argument  |effect |
|-------|-----------|-------|
|prefix | string    |Prepends the value to all results |
|suffix | string    |Appends the value to all results  |
|quote  | string    |Wraps the resulting value on both sides with the provided string |
|cast   | i,int,f,float,s,str,string|For numeric types, will cast results the provided type|
|join_with|string   |For types that produce multiple values, use this string to join them   |
|as_list|yes,true,on|For types that produce multiple values, return as list without joining |

Example:

<details open>
  <summary>JSON Spec</summary>

```json
{
  "field": {
      "type": "values",
      "config": {
        "prefix": "Hello "
      },
      "data": ["world", "beautiful", "destiny"]
    }
}
```
</details>

<details>
  <summary>YAML Spec</summary>

```yaml
---
field:
  type: values
  config:
    prefix: 'Hello '
  data: [world, beautiful, destiny]
```
</details>

# <a name="Field_Spec_Types"></a>Field Spec Types

These are the built in types

## <a name="Values"></a>Values

There are three types of values specs: Constants, List, and Weighted. Values specs have a shorthand notation where the
value of the data element replaces the full spec. See examples below.

### <a name="Constant_Values"></a>Constant Values

A Constant Value is just a single value that is used in every iteration

<details open>
  <summary>JSON Spec</summary>

```json
{
  "constant1": {"type": "values", "data": 42},
  "shorthand_constant": "This is simulated data and should not be used for nefarious purposes"
}
```
</details>

<details>
  <summary>YAML Spec</summary>

```yaml
constant1:
  type: values
  data: 42
shorthand_constant: This is simulated data and should not be used for nefarious purposes
```
</details>

### <a name="List_Values"></a>List Values

List values are rotated through in order. If the number of iterations is larger than the size of the list, we start over
from the beginning of the list. Use the `sample` config param to specify that the values should be selected at random
from the provided list.

<details open>
  <summary>JSON Spec</summary>

```json
{
  "list1": {"type": "values", "data": [200, 202, 303, 400, 404, 500]},
  "shorthand_list":  [200, 202, 303, 400, 404, 500],
  "random_pet?sample=true": ["dog", "cat", "bunny", "pig", "rhino", "hedgehog"]
}
```
</details>

<details>
  <summary>YAML Spec</summary>

```yaml
---
list1:
  type: values
  data: [200, 202, 303, 400, 404, 500]
shorthand_list: [200, 202, 303, 400, 404, 500]
random_pet?sample=true: [dog, cat, bunny, pig, rhino, hedgehog]
```
</details>

### <a name="Weighted_Values"></a>Weighted Values

Weighted values are generated according to their weights.

<details open>
  <summary>JSON Spec</summary>

```json
{
  "weighted1": {
    "type": "values",
    "data": {
      "200": 0.4, "202": 0.3, "303": 0.1,
      "400": 0.05, "403": 0.05, "404": 0.05, "500": 0.05
    }
  },
  "shorthand_weighted": {
    "200": 0.4, "202": 0.3, "303": 0.1,
    "400": 0.05, "403": 0.05, "404": 0.05, "500": 0.05
  }
}
```
</details>

<details>
  <summary>YAML Spec</summary>

```yaml
---
weighted1:
  type: values
  data:
    '200': 0.4
    '202': 0.3
    '303': 0.1
    '400': 0.05
    '403': 0.05
    '404': 0.05
    '500': 0.05
shorthand_weighted:
  '200': 0.4
  '202': 0.3
  '303': 0.1
  '400': 0.05
  '403': 0.05
  '404': 0.05
  '500': 0.05
```
</details>

The example above will generate 200 40% of the time and 400 and 403 5%. The higher the number of iterations the more
likely the values will match their specified weights.

### <a name="Sample_Mode"></a>Sample Mode

To increase the randomness of the data being generated you can configure a FieldSpec that contains a list of values to
be sampled instead of iterated through incrementally. Normally the spec below would create the repeating
sequence: `A1 B2 C3`, but since both fields `ONE` and `TWO` are in sample mode, we will get all nine combinations of
values after a significant number of iterations. This would also be true if only one was set to sample mode. To turn
sample mode on either use a URL param or config entry with one of `on`,  `yes`, or `true`. NOTE: Sample mode is only
valid with entries that are lists.

<details open>
  <summary>JSON Spec</summary>

```json
{
  "combine": {"type": "combine", "refs": ["ONE", "TWO"]},
  "refs": {
    "ONE?sample=true": ["A", "B", "C"],
    "TWO?sample=true": [1, 2, 3]
  }
}
```
</details>

<details>
  <summary>YAML Spec</summary>

```yaml
---
combine:
  type: combine
  refs:
  - ONE
  - TWO
refs:
  ONE?sample=true: [A, B, C]
  TWO?sample=true: [1, 2, 3]
```
</details>

## <a name="Combine"></a>Combine

A combine Field Spec is used to concatenate or append two or more fields or reference to one another.

The combine Field Spec structure is:

```
{
  "<field name>": {
    "type": "combine",
    "fields": ["valid field name1", "valid field name2"],
    OR
    "refs": ["valid ref1", "valid ref2"],
    "config": {"join_with": "<optional string to use to join fields or refs, default is none>"}
  }
}
```

Example below uses the first and last refs to create a full name field.

<details open>
  <summary>JSON Spec</summary>

```json
{
  "full name": {
    "type": "combine",
    "refs": ["first", "last"],
    "config": {"join_with": " "}
  },
  "refs": {
    "first": {
      "type": "values",
      "data": ["zebra", "hedgehog", "llama", "flamingo"]
    },
    "last": {
      "type": "values",
      "data": ["jones", "smith", "williams"]
    }
  }
}
```
</details>

<details>
  <summary>YAML Spec</summary>

```yaml
---
full name:
  type: combine
  refs:
  - first
  - last
  config:
    join_with: ' '
refs:
  first:
    type: values
    data: [zebra, hedgehog, llama, flamingo]
  last:
    type: values
    data: [jones, smith, williams]
```
</details>

## <a name="CombineList"></a>Combine List

A combine-list Field Spec is used to specify a list of lists of refs to combine. This is useful if there are a lot of
variations on the values that should be combined. This allows all the variations to be specified in one place. Note:
This approach requires the same join_with param for each set of refs.

The combine Field Spec structure is:

```
{
  "<field name>": {
    "type": "combine-list",
    "refs": [
      ["valid ref1", "valid ref2"],
      ["valid ref1", "valid ref2", "valid_ref3", ...], ...
      ["another_ref", "one_more_ref"]
    ],
    "config": {"join_with": "<optional string to use to join fields or refs, default is none>"}
  }
}
```

This is a slight modification to the above combine Example.

<details open>
  <summary>JSON Spec</summary>

```json
{
  "full name": {
    "type": "combine-list",
    "refs": [
      ["first", "last"],
      ["first", "middle", "last"],
      ["first", "middle_initial", "last"]
    ],
    "config": {"join_with": " "}
  },
  "refs": {
    "first": ["zebra", "hedgehog", "llama", "flamingo"],
    "last": ["jones", "smith", "williams"],
    "middle": ["cloud", "sage", "river"],
    "middle_initial": {"a": 0.3, "m": 0.3, "j": 0.1, "l": 0.1, "e": 0.1, "w": 0.1}
  }
}
```
</details>

<details>
  <summary>YAML Spec</summary>

```yaml
---
full name:
  type: combine-list
  refs: [
    [first, last],
    [first, middle, last],
    [first, middle_initial, last]
  ]
  config:
    join_with: ' '
refs:
  first: [zebra, hedgehog, llama, flamingo]
  last: [jones, smith, williams]
  middle: [cloud, sage, river]
  middle_initial:
    a: 0.3
    m: 0.3
    j: 0.1
    l: 0.1
    e: 0.1
    w: 0.1
```
</details>

## <a name="Date"></a>Date

A Date Field Spec is used to generate date strings. The default format is day-month-year i.e. Christmas 2050 would be:
25-12-2050. There is also a `date.iso` type that generates ISO8601 formatted date strings without microseconds and a
`date.iso.us` for one that generates them with microseconds. We use
the [format specification](https://docs.python.org/3/library/datetime.html#strftime-and-strptime-format-codes)
from the datetime module. The default strategy is to create dates around a center date. The default is to use today with
a spread of +-15 days. To make the base or anchor date the start or end of the date range, use the delta_days parameter
with an array of two elements, where one is zero. If the first is zero then all generated dates will only be after the
base/anchor date. If the second element is zero then all generated dates will be before the base/anchor date. There are
a lot of configuration parameters for the date module. Each are described below.

### Parameters

| param | description                                      | examples |
|-------|--------------------------------------------------|----------|
|format | datetime compatible format specification         | %Y-%m-%d, %m/%d/%Y, %H:%M:%S,... |
|delta_days | The number of days +- from the base/anchor date to create date strings for | 1, 12, \[14, 0\] |
|anchor | date string matching format or default format to use for base date | 22-02-2022 |
|offset | number of days to shift base date by, positive means shift backwards, negative means forward | 30, -7, ...|

The date Field Spec structure is:

```
{
  "<field name>": {
    "type": "date",
    OR,
    "type": "date.iso",
    OR,
    "type": "date.iso.us",
    "config": {"...": "..."}
  }
}
```

### Examples

To help with the number of variations of date formats, below is a table of examples. Assume today is 15 Jan 2050, so the
default date formatted for today would be 15-01-2050

|format  |delta_days|anchor     |offset|produces                 |spec|
|--------|----------|-----------|------|-------------------------|----|
|-       |-         |-          |-     |12-31-2049 ... 30-01-2050|`{"dates:date":{}}`|
|-       |-         |-          |1     |12-30-2049 ... 29-01-2050|`{"dates:date?offset=1":{}}`|
|-       |1         |-          |-     |14-01-2050 ... 16-01-2050|`{"dates:date?delta_days=1":{}}`|
|-       |-1        |-          |-     |same as above            |`{"dates:date?delta_days=-1":{}}`|
|-       |1         |-          |1     |13-01-2050 ... 15-01-2050|`{"dates:date?delta_days=1&offset=1":{}}`|
|-       |1         |-          |-1    |15-01-2050 ... 17-01-2050|`{"dates:date?delta_days=1&offset=-1":{}}`|
|-       |1         |15-12-2050 |1     |13-12-2050 ... 15-12-2050|`{"dates:date?delta_days=1&offset=1&anchor=15-12-2050":{}}`|
|%d-%b-%Y|1         |15-Dec-2050|-     |14-Dec-2050 ... 16-Dec-2050|`{"dates:date?delta_days=1&anchor=15-Dec-2050&format=%d-%b-%Y":{}}`|
|-       |\[1,2\]   |-          |-     |15-01-2050 ... 17-01-2050|`{"dates:date":{"config":{"delta_days":[0, 2]}}}`|

### ISO8601 formatted dates

The type `date.iso` will produce a ISO8601 formatted date in the bounds configured without milliseconds. Use
the `date.iso.us` type to generate them with microseconds.

## <a name="Range"></a>Range

A `range` spec is used to generate a range of values. The ranges are inclusive for start and end. The start, stop, and
step can be integers or floating point numbers.

The range Field Spec structure is:

```
{
  "<field name>": {
    "type": "range",
    "data": [<start>, <end>, <step> (optional)],
    or
    "data": [
      [<start>, <end>, <step> (optional)],
      [<start>, <end>, <step> (optional)],
      ...
      [<start>, <end>, <step> (optional)],
    ],
  }
}
```

Example: Range 0 to 10 with a step of 0.5

<details open>
  <summary>JSON Spec</summary>

```json
{
  "zero_to_ten": {
    "type": "range",
    "data": [0, 10, 0.5]
  },
  "range_shorthand1:range": {"data": [0, 10, 0.5]},
  "range_shorthand2:range": [0, 10, 0.5]
}
```
</details>

<details>
  <summary>YAML Spec</summary>

```yaml
---
range_shorthand1:range:
  data:
  - 0
  - 10
  - 0.5
range_shorthand2:range:
- 0
- 10
- 0.5
zero_to_ten:
  data:
  - 0
  - 10
  - 0.5
  type: range
```
</details>

Example: Multiple Ranges One Field

<details open>
  <summary>JSON Spec</summary>

```json
{
  "salaries": {
    "type": "range",
    "data": [
      [1000, 10000, 1000],
      [10000, 55000, 5000],
      [55000, 155000, 10000]
    ]
  }
}
```
</details>

<details>
  <summary>YAML Spec</summary>

```yaml
---
salaries:
  type: range
  data: [
    [1000, 10000, 1000],
    [10000, 55000, 5000],
    [55000, 155000, 10000]
  ]
```
</details>

This spec produces integer values for three different ranges each with different step sizes.

## <a name="RandRange"></a>Random Range

A `rand_range` spec is used to generate a number with in a range. Use the `cast` param to explicitly cast the value to
one of int, float, or string. The default is to return value as a string.

The range Field Spec structure is:

```
{
  "<field name>": {
    "type": "rand_range",
    "data": [<upper>],
    or
    "data": [<lower>, <upper>],
    or
    "data": [<lower>, <upper>, <precision> (optional)]
  }
}
```

If a single element is provided in the `data` array, it will be used as the upper bound and 0 will be the lower.

### Config Params

|param    |description|
|---------|-----------|
|precision|How many digits after decimal point to include|
|cast     |Type to cast result to, default is to return as string|

Example:

Two different population fields. The first generates an integer uniformly between 100 and 1000. The second generates a
float between 200.2 and 1222.7 with two values after the decimal place. Note the abbreviation for cast.

<details open>
  <summary>JSON Spec</summary>

```json
{
  "population": {
    "type": "rand_range",
    "data": [100, 1000],
    "config": {"cast": "int"}
  },
  "pop:rand_range?cast=f": [200.2, 1222.7, 2]
}
```
</details>

<details>
  <summary>YAML Spec</summary>

```yaml
---
population:
  type: rand_range
  data: [100, 1000]
  config:
    cast: int
pop:rand_range?cast=f: [200.2, 1222.7, 2]
```
</details>

## <a name="Uuid"></a>Uuid

A standard uuid.

The uuid Field Spec structure is:

```
{
  "<field name>": {
    "type": "uuid"
  }
}
```

Example Spec

<details open>
  <summary>JSON Spec</summary>

```json
{
  "id": {
    "type": "uuid"
  },
  "id_shorthand:uuid": {}
}
```
</details>

<details>
  <summary>YAML Spec</summary>

```yaml
---
id:
  type: uuid
id_shorthand:uuid: {}
```
</details>

## <a name="CharClass"></a>Character Classes

A `char_class` type is used to create strings that are made up of characters from specific character classes. The strings
can be of fixed or variable length. There are several built in character classes. You can also provide your own set of
characters to sample from. Below is the list of supported character classes:

### <a name="SupportedClasses"></a>Built In Classes

|class      |description|
|-----------|--------------------------------------------|
|ascii      |All valid ascii characters including control|
|lower      |ascii lowercase|
|upper      |ascii uppercase|
|digits     |Numbers 0 through 9|
|letters    |lowercase and uppercase|
|word       |letters + digits + '_'|
|printable  |All printable ascii chars including whitespace|
|visible    |All printable ascii chars excluding whitespace|
|punctuation|local specific punctuation|
|special    |local specific punctuation|
|hex        |Hexidecimal digits including upper and lower case a-f|
|hex-lower  |Hexidecimal digits only including lower case a-f|
|hex-upper  |Hexidecimal digits only including upper case A-F|

Helpful Links:

  * https://en.wikipedia.org/wiki/ASCII#Character_groups
  * https://www.cs.cmu.edu/~pattis/15-1XX/common/handouts/ascii.html
  * https://docs.python.org/3/library/string.html

### Usage

A `char_class` Field Spec takes the form

```
{
  "<field>": {
    # type definition
    "type": "char_class":
    or
    "type": "cc-<char_class_name>",
    # data definition
    "data": <char_class_name>,
    or
    "data": <string with custom set of characters to sample from>
    or
    "data": [<char_class_name1>, <char_class_name2>, ..., <custom characters>, ...]
    # configuration
    "config":{
      # General Parameters
      "exclude": <string of characters to exclude from output>,
      # String Size Based Config Parameters
      "min": <min number of characters in string>,
      "max": <max number of characters in string>,
      or
      "count": <exact number of characters in string>
      or
      "mean": <mean number of characters in string>
      "stddev": <standard deviation from mean for number of characters in string>
      "min": <optional min>
      "max": <optional max>
    }    
  }
}
```

### Shorthand Notation for Single Character Classes

If a single character class is needed, the type can be specified with a `cc-` prefix: `cc-<char_class_name>`
e.g. `"type": "cc-visible"` would only use characters from the `visible` class. If this format is used, the `data`
element is ignored and only characters from the single character class are sampled from.

<details open>
  <summary>JSON Spec</summary>

```json
{
  "one_to_five_digits:cc-digits?min=1&max=5": {}
}
```
</details>

<details>
  <summary>YAML Spec</summary>

```yaml
one_to_five_digits:cc-digits?min=1&max=5: {}
```
</details>

### Examples

Below is an example selecting several character classes along with a set of custom ones to use to generate passwords.
The generated passwords are between 10 and 18 characters in length with a mean size of 14 characters and a standard
deviation of 2.

<details open>
  <summary>JSON Spec</summary>

```json
{
  "password": {
    "type": "char_class",
    "data": [
      "word",
      "special",
      "hex-lower",
      "M4$p3c!@l$@uc3"
    ],
    "config": {
      "mean": 14,
      "stddev": 2,
      "min": 10,
      "max": 18,
      "exclude": ["'", "\""]
    }
  }
}
```
</details>

<details>
  <summary>YAML Spec</summary>

```yaml
---
password:
  type: char_class
  data:
  - word
  - special
  - hex-lower
  - M4$p3c!@l$@uc3
  config:
    mean: 14
    stddev: 2
    min: 10
    max: 18
    exclude: ['''', '"']
```
</details>

If we run this example:

```shell
dataspec -s password.json -i 10
61Nl=U35LVY^*
pe576Z*P198&f4t
1p}=HA9b?iJDc
7$C])[o<BDh&W9kz
F%iGot-j~eUfpT3qR
(TM9b$o$A4KG*{&
{WPm1;]O:w@y27l\
JeTmpZ@zYN2Ms3,
96i@FPA$^Q#%(|7Cba
Q=+6D*8I36R%<n`$8
```

The `stddev` config parameters is not required, but without it the sizes will tend to stack on the edges of the allowed
size range.

<details>
  <summary>Detailed Example</summary>

```shell
# no stddev specified
for p in $(dataspec -l off --inline "password:cc-word?mean=5&min=1&max=9: {}" -i 1000);
do
  echo $p | tr -d '\n' | wc -m
done | sort | uniq -c | sort -n -k2,2
# count num chars
    163 1
     59 2
     91 3
     92 4
    100 5
    110 6
     94 7
     71 8
    220 9
# with stddev of 3 specified
for p in $(dataspec -l off --inline "password:cc-word?mean=5&stddev=3&min=1&max=9: {}" -i 1000);
do
  echo $p | tr -d '\n' | wc -m
done | sort | uniq -c | sort -n -k2,2
# count num chars
     98 1
     72 2
     96 3
    126 4
    133 5
    128 6
    113 7
     90 8
    144 9
```

</details>

## <a name="UnicodeRange"></a>Unicode Ranges

The `unicode_range` type is similar to the `char_class` type, but it is used to generate characters from valid unicode
ranges. See [UnicodeRanges](https://www.ling.upenn.edu/courses/Spring_2003/ling538/UnicodeRanges.html) for a list of the
different valid ranges. One or more ranges can be specified in the `data` element by providing a list or list of lists
with two elements each specifying the start and end hex code points. If we wanted to generate Japanese Hiragana (Code
points 0x3040 to 0x30FF) characters as one of our fields we could use the following spec:

<details open>
  <summary>JSON Spec</summary>

```json
{
  "text": {
    "type": "unicode_range",
    "data": ["3040", "309f"],
    "config": {
      "mean": 5
    }
  }
}
```
</details>

<details>
  <summary>YAML Spec</summary>

```yaml
text:
  type: unicode_range
  data: ['3040', '309f']
  config:
    mean: 5

```
</details>

If we run this example:

```shell
dataspec -s unicode.yaml -i 10 -l off
そぬをく
が
としぶ
ぉけべ゙
ゃれげえ
っぴ
つざめよがぽ
えろぱづゖごそ぀べかっ
い゘
ぱまる
```

### Usage

A `unicode_range` Field Spec takes the form

```
{
  "<field>": {
    # type definition
    "type": "unicode_range":
    # data definition
    "data": [<start_code_point_in_hex>, <end_code_point_in_hex>],
    or
    "data": [
        [<start_code_point_in_hex>, <end_code_point_in_hex>],
        [<start_code_point_in_hex>, <end_code_point_in_hex>],
        ...
        [<start_code_point_in_hex>, <end_code_point_in_hex>],
    ],
    # configuration
    "config":{
      # String Size Based Config Parameters
      "min": <min number of characters in string>,
      "max": <max number of characters in string>,
      or
      "count": <exact number of characters in string>
      or
      "mean": <mean number of characters in string>
      "stddev": <standard deviation from mean for number of characters in string>
      "min": <optional min>
      "max": <optional max>
    }    
  }
}
```

## <a name="Geo"></a>Geo Related Types

There are three main geo types: `geo.lat`, `geo.long`, and `geo.pair`. The defaults will create decimal string values in
the valid ranges: -90 to 90 for latitude and -180 to 180 for longitude. You can bound the ranges in several ways. The
first is with the `start_lat`, `end_lat`, `start_long`, `end_long` config params. These will set the individual bounds
for each of the segments. You can use one or more of them. The other mechanism is by defining a `bbox` array which
consists of the lower left geo point and the upper right one.
See: [Bounding_Box](https://wiki.openstreetmap.org/wiki/Bounding_Box#)

Config Params:

|type    |param     |description                                  |
|--------|----------|---------------------------------------------|
|all     |precision |number of decimal places for lat or long, default is 4          |
|        |bbox      |array of \[min Longitude, min Latitude, max Longitude, max Latitude\]|
|geo.lat |start_lat |lower bound for latitude                                        |
|        |end_lat   |upper bound for latitude                                        |
|geo.long|start_long|lower bound for longitude                                       |
|        |end_long  |upper bound for longitude                                       |
|geo.pair|join_with |delimiter to join long and lat with, default is comma           |
|        |as_list   |One of yes, true, or on if the pair should be returned as a list instead of as a joined string|
|        |lat_first |if latitude should be first in the generated pair, default is longitude first|
|        |start_lat |lower bound for latitude                                        |
|        |end_lat   |upper bound for latitude                                        |
|        |start_long|lower bound for longitude                                       |
|        |end_long  |upper bound for longitude                                       |

Examples:

Generates a `longitude,latitude` pair with in the bounding box defining Egypt with 3 decimal points of precision.

<details open>
  <summary>JSON Spec</summary>

```json
{
  "egypt": {
    "type": "geo.point",
    "config": {
      "bbox": [31.33134, 22.03795, 34.19295, 25.00562],
      "precision": 3
    }
  }
}
```
</details>

<details>
  <summary>YAML Spec</summary>

```yaml
---
egypt:
  type: geo.point
  config:
    bbox:
    - 31.33134
    - 22.03795
    - 34.19295
    - 25.00562
    precision: 3
```
</details>

## <a name="IP_Addresses"></a>IP Addresses

Ip addresses can be generated using [CIDR notation](https://en.wikipedia.org/wiki/Classless_Inter-Domain_Routing) or by
specifying a base.

The ipv4 Field Spec structure is:

```
{
  "<field name>": {
    "type": "ipv4",
    "config": {
      "cidr": "<cidr value /8 /16 /24 only>",
      OR
      "base": "<beginning of ip i.e. 10.0>"
    }
  }
}
```

Example Spec:

<details open>
  <summary>JSON Spec</summary>

```json
{
  "network": {
    "type": "ipv4",
    "config": {
      "cidr": "2.22.222.0/16"
    }
  },
  "network_shorthand:ip?cidr=2.22.222.0/16": {},
  "network_with_base:ip?base=192.168.0": {}
}
```
</details>

<details>
  <summary>YAML Spec</summary>

```yaml
---
network:
  type: ipv4
  config:
    cidr: 2.22.222.0/16
network_shorthand:ip?cidr=2.22.222.0/16: {}
network_with_base:ip?base=192.168.0: {}
```
</details>

### <a name="Precise_IP"></a> Precise CIDR Addresses

The default ip type only supports cidr masks of /8 /16 and /24. If you want more precise ip ranges you need to use the
`ip.precise` type. This type requires a cidr as the single config param. For some cidr values the number of generated
ips becomes large, and the underlying module used becomes preventatively slow. Even with a /16 address it can take
multiple seconds to generate only 1000 ips. Anything smaller than that may not be worth it. The default mode
for `ip.precise` is to increment the ip addresses. Set config param `sample` to on of `true`, `on`, or `yes` to enable
random ip addresses selected from the generated ranges.

#### Examples

Ips in the 10.n.n.n range, extremely slow, this is around 16 Million unique ip addresses

<details open>
  <summary>JSON Spec</summary>

```json
{"network:ip.precise?cidr=10.0.0.0/8":{}}
```
</details>

<details>
  <summary>YAML Spec</summary>

```yaml
---
network:ip.precise?cidr=10.0.0.0/8: {}
```
</details>

Ips in the 192.168.0.0 to 192.171.255.255 range, relatively slow, creates around 250K addresses

<details open>
  <summary>JSON Spec</summary>

```json
{"network:ip.precise?cidr=192.168.0.0/14&sample=true": {}}
```
</details>

<details>
  <summary>YAML Spec</summary>

```yaml
---
network:ip.precise?cidr=192.168.0.0/14&sample=true: {}
```
</details>

Ips in the 2.22.220.0 to 2.22.223.255 range, speed is tolerable

<details open>
  <summary>JSON Spec</summary>

```json
{"network:ip.precise?cidr=2.22.222.0/22": {}}
```
</details>

<details>
  <summary>YAML Spec</summary>

```yaml
---
network:ip.precise?cidr=2.22.222.0/22: {}
```
</details>

## <a name="Weighted_Ref"></a>Weighted Ref

A weighted ref spec is used to select the values from a set of refs in a weighted fashion.

The weightedref Field Spec structure is:

```
{
  "<field name>": {
    "type": "weightedref",
    "data": {"valid_ref_1": 0.N, "valid_ref_2": 0.N, ...}
  }
}
```

For example if we want to generate a set of HTTP response codes, but we want mostly success related codes we could use
the follow spec.

<details open>
  <summary>JSON Spec</summary>

```json
{
  "http_code": {
    "type": "weightedref",
    "data": {
      "GOOD_CODES": 0.7,
      "BAD_CODES": 0.3
    }
  },
  "refs": {
    "GOOD_CODES": {"200": 0.5, "202": 0.3, "203": 0.1, "300": 0.1},
    "BAD_CODES": {"400": 0.5, "403": 0.3, "404": 0.1, "500": 0.1}
  }
}
```
</details>

<details>
  <summary>YAML Spec</summary>

```yaml
---
http_code:
  type: weightedref
  data:
    GOOD_CODES: 0.7
    BAD_CODES: 0.3
refs:
  BAD_CODES:
    '400': 0.5
    '403': 0.3
    '404': 0.1
    '500': 0.1
  GOOD_CODES:
    '200': 0.5
    '202': 0.3
    '203': 0.1
    '300': 0.1
```
</details>

## <a name="Select_List_Subset"></a>Select List Subset

A select list subset spec is used to select multiple values from a list to use as the value for a field.

The select_list_subset Field Spec structure is:

```
{
  "<field name>": {
    "type": "select_list_subset",
    "config": {
      "mean": N,
      "stddev": N,
      "min": N,
      "max": N,
      "join_with": "<delimiter to join with>"
    },
    "data": ["data", "to", "select", "from"],
    OR
    "ref": "REF_WITH_DATA_AS_LIST"
  }
}
```

The join_with config option is used to specify how the selected values should be combined. The mean and stddev config
options tell how many items should be chosen. For example a mean of 2 and stddev of 1, would mostly choose 2 items then
sometimes 1 or 3 or more. Set the stddev to 0 if only the exact number of items should be chosen.
You can also set a min and max. Example:

<details open>
  <summary>JSON Spec</summary>

```json
{
  "ingredients": {
    "type": "select_list_subset",
    "config": {
      "mean": 3,
      "stddev": 1,
      "min": 2,
      "max": 4,
      "join_with": ", "
    },
    "data": ["onions", "mushrooms", "garlic", "bell peppers", "spinach", "potatoes", "carrots"]
  }
}
```
</details>

<details>
  <summary>YAML Spec</summary>

```yaml
---
ingredients:
  config:
    join_with: ', '
    max: 4
    mean: 3
    min: 2
    stddev: 1
  data:
  - onions
  - mushrooms
  - garlic
  - bell peppers
  - spinach
  - potatoes
  - carrots
  type: select_list_subset
```
</details>

```shell script
dataspec -s ~/scratch/ingredients.json -i 10
garlic, onions
garlic, spinach
bell peppers, spinach
mushrooms, bell peppers, carrots, potatoes
mushrooms, potatoes, bell peppers
potatoes, onions, garlic, bell peppers
potatoes, bell peppers, onions, garlic
spinach, bell peppers
spinach, onions, garlic
carrots, garlic, mushrooms, potatoes
```

### <a name='quoting_sublist'></a> Quoting Sublist Elements

The default `quote` parameter will only quote the whole combined list of elements. To quote each individual element of
the sublist you need to use a special form of `join_with` along with the `quote` param. For example if we wanted all of
our ingredients surrounded with double quotes. We would update our spec this way.

<details open>
  <summary>JSON Spec</summary>

```json
{
  "ingredients": {
    "type": "select_list_subset",
    "config": {
      "mean": 3, "stddev": 1, "min": 2, "max": 4,
      "join_with": "\", \"",
      "quote": "\""
    },
    "data": ["onions", "mushrooms", "garlic", "bell peppers", "spinach", "potatoes", "carrots"]
  }
}
```
</details>

<details>
  <summary>YAML Spec</summary>

```yaml
---
ingredients:
  type: select_list_subset
  config:
    join_with: '", "'
    mean: 3
    stddev: 1
    min: 2
    max: 4
    quote: '"'
  data:
  - onions
  - mushrooms
  - garlic
  - bell peppers
  - spinach
  - potatoes
  - carrots
```
</details>

Now when we run our datespec we get:

```shell script
dataspec -s ~/scratch/quoted_ingredients.json -i 10
"spinach", "mushrooms", "bell peppers", "onions"
"spinach", "onions", "mushrooms", "garlic"
"carrots", "garlic", "mushrooms", "onions"
"mushrooms", "bell peppers", "carrots"
"carrots", "potatoes", "bell peppers", "onions"
"spinach", "mushrooms"
"mushrooms", "bell peppers", "onions"
"potatoes", "carrots", "bell peppers", "spinach"
"garlic", "mushrooms", "potatoes"
"carrots", "spinach", "bell peppers", "potatoes"
```

## <a name='CSV_Data'></a> CSV Data

If you have an existing large set of data in a tabular format that you want to use, it would be burdensome to copy and
paste the data into a spec. To make use of data already in a tabular format you can use a `csv` Field Spec. These specs
allow you to identify a column from a tabular data file to use to provide the values for a field. Another advantage of
using a csv spec is that it is easy to have fields that are correlated be generated together. All rows will be selected
incrementally, unless any of the fields are configured to use `sample` mode. You can use `sample` mode on individual
columns, or you can use it across all columns by creating a `configref` spec. See [csv_select](#csv_select) for an efficient way
to select multiple columns from a csv file.

The `csv` Field Spec structure is:

```json
{
  "<field name>": {
    "type": "csv",
    "config": {
      "datafile": "filename in datedir",
      "headers": "yes, on, true for affirmative",
      "column": "1 based column number or field name if headers are present",
      "delimiter": "how values are separated, default is comma",
      "quotechar": "how values are quoted, default is double quote",
      "sample": "If the values should be selected at random, default is false",
      "count": "Number of values in column to use for value"
    }
  }
}
```

#### Params

|param      |required?|default |description|
|-----------|---------|--------|-----------|
|datafile   |no       |data.csv|filename in datandir to use|
|headers    |no       |false   |yes, on, true for affirmative|
|column     |no       |1       |1 based column number or field name if headers are present|
|delimiter  |no       |,       |how values are separated|
|quotechar  |no       |"       |how values are quoted, default is double quote|
|sample     |no       |False   |If the values should be selected at random|
|count      |no       |1       |Number of values in column to use for value|

#### Examples

##### Single Field

The simplest example is a file with a single field that contains the values to generate for a field. For example if we
have a known list of cities, we can put this in a file and reference it from our spec. The advantage of this approach is
that it is easy to add new data points and to use small sets of data for testing by creating directories that have
smaller input files.

<details open>
  <summary>JSON Spec</summary>

```json
{
  "cities": {
    "type": "csv",
    "config": {
      "datafile": "cities.csv",
      "delimiter": "~",
      "sample": true
    }
  }
}
```
</details>

<details>
  <summary>YAML Spec</summary>

```yaml
---
cities:
  type: csv
  config:
    datafile: cities.csv
    delimiter: '~'
    sample: true
```
</details>

```shell
dataspec --spec cities.json --datadir ./data -i 5
Tokyo
Los Angeles
New York
Chicage
London
```

Note that if your data might have commas in it (the default delimiter), you should specify a delimiter that will not be
found in your data.

##### Multiple Fields Non Comma Separated

In this example we have a tab delimited file with multiple columns that we want to use.

```
status	status_description	status_type
100	Continue	Informational
101	Switching Protocols	Informational
200	OK	Successful
201	Created	Successful
202	Accepted	Successful
...
```

Our Data Spec looks like:

<details open>
  <summary>JSON Spec</summary>

```json
{
    "status": {
        "type": "csv",
        "config": {
            "configref": "tabs_config",
            "column": 1
        }
    },
    "description": {
        "type": "csv",
        "config": {
            "configref": "tabs_config",
            "column": 2
        }
    },
    "status_type:csv?configref=tabs_config&column=3": {},
    "refs": {
        "tabs_config": {
            "type": "configref",
            "config": {
                "datafile": "tabs.csv",
                "delimiter": "\\t",
                "headers": true
            }
        }
    }
}
```
</details>

<details>
  <summary>YAML Spec</summary>

```yaml
---
status:
  type: csv
  config:
    configref: tabs_config
    column: 1
description:
  type: csv
  config:
    configref: tabs_config
    column: 2
# shorthand notation
status_type:csv?configref=tabs_config&column=3: { }
refs:
  tabs_config:
    type: configref
    config:
      datafile: tabs.csv
      delimiter: '\t'
      headers: true
```
</details>

The `configref` exist so that we don't have to repeat ourselves for common configurations across multiple fields. If we
use the following template `{{ status }},{{ description }},{{ status_type }}` and run this spec we will get output
similar to:

```shell
dataspec --spec tabs.yaml --datadir ./data -t template.jinja -i 5
100,Continue,Informational
101,Switching Protocols,Informational
200,OK,Successful
201,Created,Successful
202,Accepted,Successful
```

## <a name="CSV_Select"></a>CSV Select

A common process is to select subsets of the columns from a csv file to use. The `csv_select` type makes this more
efficient than using the standard `csv` type. Below is an example that will Convert data from
the [Geonames](http://www.geonames.org/) [allCountries.zip](http://download.geonames.org/export/dump/allCountries.zip)
dataset by selecting a subset of the columns from the tab delimited file. The key in the data element is the new name
for the field. The value can either be the 1 indexed column number or the name of the field if the data has `headers`.
Our example doesn't have headers, so we are using the 1 based indexes.

<details open>
  <summary>JSON Spec</summary>

```json
{
    "placeholder": {
        "type": "csv_select",
        "data": {
            "geonameid": 1,
            "name": 2,
            "latitude": 5,
            "longitude": 6,
            "country_code": 9,
            "population": 15
        },
        "config": {
            "datafile": "allCountries.txt",
            "headers": false,
            "delimiter": "\t"
        }
    }
}
```
</details>

<details>
  <summary>YAML Spec</summary>

```yaml
---
placeholder:
  type: csv_select
  data:
    geonameid: 1
    name: 2
    latitude: 5
    longitude: 6
    country_code: 9
    population: 15
  config:
    datafile: allCountries.txt
    headers: no
    delimiter: "\t"
```
</details>

## <a name="nested"></a>Nested fields

Many documents or objects are not flat, but contain nested inner objects or child documents. To generate nested fields
use the `nested` type. 

### Example:

In this example a pseudo schema for our data might look like this:

```
 - id:str
 - user
   - user_id:str
   - geo
     - place_id:str
     - coordinates: List[float]
```

The user is a nested object, which has a geo, which is also a nested object. Below are the specs that will generate data
that matches this schema.

<details open>
  <summary>JSON Spec</summary>

```json
{
  "id:uuid": {},
  "user:nested": {
    "user_id:uuid": {},
    "geo:nested": {
      "place_id:uuid": {},
      "coordinates:geo.pair?as_list=true": {}
    }
  }
}
```
</details>

<details>
  <summary>YAML Spec</summary>

```yaml
---
id:uuid: {}
user:nested:
  user_id:uuid: {}
  geo:nested:
    place_id:uuid: {}
    coordinates:geo.pair?as_list=true: {}
```
</details>

If we run this example:

```shell
dataspec -s double-nested.json -i 1 --format json-pretty
{
    "id": "4278b060-442d-4558-bf2c-5f1df68cb265",
    "user": {
        "geo": {
            "coordinates": [
                "-167.4324",
                " 84.6883"
            ],
            "place_id": "510e5740-6a13-4c0b-8c53-2e3c1f88ca24"
        },
        "user_id": "13d5c2a6-80c8-4bdb-89b2-7da9699cd0fb"
    }
} 
```

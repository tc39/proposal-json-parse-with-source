# JSON.parse source text access

A proposal for extending `JSON.parse` behavior to grant reviver functions access to the input source text and extending `JSON.stringify` behavior to support object placeholders for raw JSON text primitives.

[2023 September slides](https://docs.google.com/presentation/d/1pg1gnNeMIcAbwq-CdQE7vlSldlNZ5q2hC5OA-U8ScI8/edit?usp=sharing)

[original 2018 September slides](https://docs.google.com/presentation/d/1PB0HCOxWZikFmTAqR5U2ZZjEiDV7NjhPN_-SK5NNG0w/edit?usp=sharing)

## Status
This proposal is at stage 3 of [the TC39 Process](https://tc39.github.io/process-document/).

## Champions
* Richard Gibson
* Mathias Bynens

## Motivation
Transformation between ECMAScript values and JSON text is lossy.
This is most obvious in the case of deserializing numbers (e.g., `"999999999999999999"`, `"999999999999999999.0"`, and `"1000000000000000000"` all parse to `1000000000000000000`), but also comes up when attempting to round-tripping non-primitive values such as Date objects (e.g., `JSON.parse(JSON.stringify(new Date("2018-09-25T14:00:00Z")))` yields a string `"2018-09-25T14:00:00.000Z"`).

Neither of these examples is hypothetical—serializing a [BigInt](https://github.com/tc39/proposal-bigint) as JSON is specified to throw an exception because there is no output that would round-trip through `JSON.parse`, and a similar concept has been raised regarding the [Temporal proposal](https://github.com/tc39/proposal-temporal).

`JSON.parse` accepts a reviver function capable of processing inbound values, but it is invoked bottom-up and receives so little context (a _key_, an already-lossy _value_, and a receiver upon which _key_ is an own property with value _value_) that it is practically useless.
We intend to remedy that.

## Proposed Solution
Update `JSON.parse` to provide reviver functions with more arguments, primarily conveying the source text from which a value was derived (inclusive of punctuation but exclusive of leading/trailing insignificant whitespace).

## Serialization 
Although originally not included in this proposal, support for non-lossy serialization with `JSON.stringify` (and thus also complete round-trippability) was requested and added (cf. [#12](https://github.com/tc39/proposal-json-parse-with-source/issues/12)), currently using special "raw JSON" frozen objects constructible with `JSON.rawJSON` but possibly subject to change (cf. [#18](https://github.com/tc39/proposal-json-parse-with-source/issues/18) and [#19](https://github.com/tc39/proposal-json-parse-with-source/issues/19)).

## Illustrative examples
```js
const digitsToBigInt = (key, val, {source}) =>
  /^[0-9]+$/.test(source) ? BigInt(source) : val;

const bigIntToRawJSON = (key, val) =>
  typeof val === "bigint" ? JSON.rawJSON(String(val)) : val;

const tooBigForNumber = BigInt(Number.MAX_SAFE_INTEGER) + 2n;
JSON.parse(String(tooBigForNumber), digitsToBigInt) === tooBigForNumber;
// → true

const wayTooBig = BigInt("1" + "0".repeat(1000));
JSON.parse(String(wayTooBig), digitsToBigInt) === wayTooBig;
// → true

const embedded = JSON.stringify({ tooBigForNumber }, bigIntToRawJSON);
embedded === '{"tooBigForNumber":9007199254740993}';
// → true
```

### Potential enhancements
#### Expose position and input information
`String.prototype.replace` passes position and input arguments to replacer functions and the return value from `RegExp.prototype.exec` has "index" and "input" properties; `JSON.parse` could behave similarly.
```js
const input = '\n\t"use\\u0020strict"';
let spied;
const parsed = JSON.parse(input, (key, val, context) => (spied = context, val));
parsed === 'use strict';
// → true
spied.source === '"use\\u0020strict"';
// → true
spied.index === 2;
// → true
spied.input === input;
// → true

```

#### Supply an array of keys for understanding value context
A reviver function sees values bottom-up, but the data structure hierarchy is already known and can be supplied to it, with or without the phantom leading empty string.
```js
const input = '{ "foo": [{ "bar": "baz" }] }';
const expectedKeys = ['foo', 0, 'bar'];
let spiedKeys;
JSON.parse(input, (key, val, {keys}) => (spiedKeys = spiedKeys || keys, val));
expectedKeys.length === spiedKeys.length;
// → true
expectedKeys.every((key, i) => spiedKeys[i] === key);
// → true
```

## Discussion
### Backwards Compatibility
Conforming ECMAScript implementations are not permitted to extend the grammar accepted by `JSON.parse`.
This proposal does not make any such attempt, and adding new function parameters is among the safest changes that can be made to the language.
All input to `JSON.parse` that is currently rejected will continue to be, all input that is currently accepted will continue to be, and the only changes to its output will be directly controlled by user code.

### Modified values
Reviver functions are intended to modify or remove values in the output, but those changes should have no effect on the source-derived arguments passed to them.
Because reviver functions are invoked bottom-up, this means that values may not correlate with source text.
We consider this to be acceptable, but mostly moot (see the following point). Where _not_ moot (such as when not-yet-visited array indexes or object entries are modified), source text is suppressed.

### Non-primitive values
Per https://github.com/tc39/proposal-json-parse-with-source/issues/10#issuecomment-704441802 , source text exposure is limited to primitive values.

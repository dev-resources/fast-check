# Fast Check
##### Yet another property based testing framework _BUT_ written in TypeScript

[![Build Status](https://travis-ci.org/dubzzz/fast-check.svg?branch=master)](https://travis-ci.org/dubzzz/fast-check)
[![npm version](https://badge.fury.io/js/fast-check.svg)](https://badge.fury.io/js/fast-check)
[![dependencies Status](https://david-dm.org/dubzzz/fast-check/status.svg)](https://david-dm.org/dubzzz/fast-check)
[![devDependencies Status](https://david-dm.org/dubzzz/fast-check/dev-status.svg)](https://david-dm.org/dubzzz/fast-check?type=dev)

[![Coverage Status](https://coveralls.io/repos/github/dubzzz/fast-check/badge.svg)](https://coveralls.io/github/dubzzz/fast-check)
[![Test Coverage](https://api.codeclimate.com/v1/badges/7cb8cb395740446a3108/test_coverage)](https://codeclimate.com/github/dubzzz/fast-check/test_coverage)
[![Maintainability](https://api.codeclimate.com/v1/badges/7cb8cb395740446a3108/maintainability)](https://codeclimate.com/github/dubzzz/fast-check/maintainability)

## Getting started

Install the module with: `npm install fast-check`

Running examples in RunKit: https://runkit.com/dubzzz/fast-check-basic-examples

## Usage

Using fast-check with [mocha](http://mochajs.org/) is really straightfoward.
It can be used directly in `describe`, `it` blocks with no extra care.

The following snippets written in Javascript shows an example featuring two properties:

```js
const fc = require('fast-check');

// Code under tests
const contains = (text, pattern) => text.indexOf(pattern) >= 0;

// Properties
describe('properties', () => {
    it('should always contain itself', () => {
        fc.assert(fc.property(fc.string(), text => contains(text, text)));
    });
    it('should always contain its substrings', () => {
        fc.assert(fc.property(fc.string(), fc.string(), fc.string(), (a,b,c) => contains(a+b+c, b)));
    });
});
```

In case of failure, the tests would raise a red flag and the output should help you to diagnose what went wrong in your implementation (example with a failing implementation of contain):

```
1) should always contain its substrings
    Property failed after 1 tests (seed: 1515709471288): [,,]
    Got error: Property failed by returning false
```

## Documentation

### Properties

- `fc.property`: define a new property ie. a list of arbitraries and a test function to assess the success

The predicate would be considered falsy if its throws or if `output == null || output == true` evaluate to `false`.
```typescript
function property<T1>(
        arb1: Arbitrary<T1>,
        predicate: (t1:T1) => (boolean|void)): Property<[T1]>;
function property<T1,T2>(
        arb1: Arbitrary<T1>, arb2: Arbitrary<T2>,
        predicate: (t1:T1,t2:T2) => (boolean|void)): Property<[T1,T2]>;
...
```

- `fc.asyncProperty`: define a new property ie. a list of arbitraries and an asynchronous test function to assess the success

The predicate would be considered falsy if its throws or if `output == null || output == true` evaluate to `false` (after `await`).
```typescript
function asyncProperty<T1>(
        arb1: Arbitrary<T1>,
        predicate: (t1:T1) => Promise<boolean|void>): AsyncProperty<[T1]>;
function asyncProperty<T1,T2>(
        arb1: Arbitrary<T1>, arb2: Arbitrary<T2>,
        predicate: (t1:T1,t2:T2) => Promise<boolean|void>): AsyncProperty<[T1,T2]>;
...
```

## Runners

- `fc.assert`: run the property and throws in case of failure

**This function has to be awaited in case it is called on an asynchronous property.**

This function is ideal to be called in `describe`, `it` blocks.
It does not return anything in case of success.

It can be parametrized using its second argument.

```typescript
export interface Parameters {
    seed?: number;     // optional, initial seed of the generator: Date.now() by default
    num_runs?: number; // optional, number of runs before success: 100 by default 
    timeout?: number;  // optional, only taken into account for asynchronous runs (asyncProperty)
                       // specify a timeout in milliseconds, maximum time for the predicate to return its result
                       // only works for async code, will not interrupt a synchronous code: disabled by default
    logger?: (v: string) => void; // optional, log output: console.log by default
}
```

```typescript
function assert<Ts>(property: IProperty<Ts>, params?: Parameters);
```

- `fc.check`: run the property and return an object containing the test status along with other useful details

**This function has to be awaited in case it is called on an asynchronous property.**

It should never throw whatever the status of the test.

It can be parametrized with the same parameters than `fc.assert`.

The details returned by `fc.check` are the following:

```typescript
interface RunDetails<Ts> {
    failed: boolean,         // false in case of failure, true otherwise
    num_runs: number,        // number of runs (all runs if success, up and including the first failure if failed)
    num_shrinks: number,     // number of shrinks (depth required to get the minimal failing example)
    seed: number,            // seed used for the test
    counterexample: Ts|null, // failure only: shrunk conterexample causig the property to fail
    error: string|null,      // failure only: stack trace and error details
}
```

```typescript
function check<Ts>(property: IProperty<Ts>, params?: Parameters);
```

- `fc.sample`: sample generated values of an `Arbitrary<T>` or `Property<T>`

It builds an array containing all the values that would have been generated for the equivalent test.

It also accept `Parameters` as configuration in order to help you diagnose the shape of the inputs that will be received by your property.

```typescript
type Generator<Ts> = Arbitrary<Ts> | IProperty<Ts>;

function sample<Ts>(generator: Generator<Ts>): Ts[];
function sample<Ts>(generator: Generator<Ts>, params: Parameters): Ts[];
function sample<Ts>(generator: Generator<Ts>, num_generated: number): Ts[];
```

- `fc.statistics`: classify the values produced by an `Arbitrary<T>` or `Property<T>`

It provides useful statistics concerning generated values.
In order to be able to gather those statistics it has to be provided with a classifier function that can classify the generated value in zero, one or more categories (free labels).

It also accept `Parameters` as configuration in order to help you diagnose the shape of the inputs that will be received by your property.

Statistics are dumped into `console.log` but can be redirected to another source by modifying the `logger` key in `Parameters`.

```typescript
type Generator<Ts> = Arbitrary<Ts> | IProperty<Ts>;
type Classifier<Ts> = ((v: Ts) => string) | ((v: Ts) => string[]);

function statistics<Ts>(generator: Generator<Ts>, classify: Classifier<Ts>): void;
function statistics<Ts>(generator: Generator<Ts>, classify: Classifier<Ts>, params: Parameters): void;
function statistics<Ts>(generator: Generator<Ts>, classify: Classifier<Ts>, num_generated: number): void;
```

## Arbitraries

Arbitraries are responsible for the random generation (but deterministic) and shrink of datatypes.
They can be combined together to build more complex datatypes.

### Boolean (:boolean)

- `fc.boolean()` either `true` or `false`

### Numeric (:number)

Integer values:

- `fc.integer()` all possible integers ie. from -2147483648 (included) to 2147483647 (included)
- `fc.integer(max: number)` all possible integers between -2147483648 (included) and max (included)
- `fc.integer(min: number, max: number)` all possible integers between min (included) and max (included)
- `fc.nat()` all possible positive integers ie. from 0 (included) to 2147483647 (included)
- `fc.nat(max: number)` all possible positive integers between 0 (included) and max (included)

Floating point numbers:

- `fc.float()` uniformly distributed `float` value between 0.0 (included) and 1.0 (excluded)
- `fc.double()`uniformly distributed `double` value between 0.0 (included) and 1.0 (excluded)

### String (:string)

Single character only:

- `fc.hexa()` one character in `0123456789abcdef` (lower case)
- `fc.base64()` one character in `A-Z`, `a-z`, `0-9`, `+` or `/`
- `fc.char()` between 0x20 (included) and 0x7e (included) , corresponding to printable characters (see https://www.ascii-code.com/)
- `fc.ascii()` between 0x00 (included) and 0x7f (included)
- `fc.unicode()` between 0x0000 (included) and 0xffff (included) but excluding surrogate pairs (between 0xd800 and 0xdfff). Generate any character of UCS-2 which is a subset of UTF-16

Multiple characters:

- `fc.hexaString()`, `fc.hexaString(maxLength: number)` or `fc.hexaString(minLength: number, maxLength: number)` string based on characters generated by `fc.hexa()`
- `fc.base64String()`, `fc.base64String(maxLength: number)` or `fc.base6'String(minLength: number, maxLength: number)` string based on characters generated by `fc.base64()`. Provide valid base 64 strings: length always multiple of 4 padded with '=' characters. When using `minLength` and `maxLength` make sure that they are compatible together. For instance: asking for `minLength=2` and `maxLength=4` is impossible for base 64 strings as produced by the framework
- `fc.string()`, `fc.string(maxLength: number)` or `fc.string(minLength: number, maxLength: number)` string based on characters generated by `fc.char()`
- `fc.asciiString()`, `fc.asciiString(maxLength: number)` or `fc.asciiString(minLength: number, maxLength: number)` string based on characters generated by `fc.ascii()`
- `fc.unicodeString()`, `fc.unicodeString(maxLength: number)` or `fc.unicodeString(minLength: number, maxLength: number)` string based on characters generated by `fc.unicode()`

Strings that mimic real strings, with words and sentences:

- `json()` or `json(maxDepth: number)` json strings having keys generated using `fc.string()`. String values are also produced by `fc.string()`
- `unicodeJson()` or `unicodeJson(maxDepth: number)` json strings having keys generated using `fc.unicodeString()`. String values are also produced by `fc.unicodeString()`
- `fc.lorem()`, `fc.lorem(maxWordsCount: number)` or `fc.lorem(maxWordsCount: number, sentencesMode: boolean)` lorem ipsum strings. Generator can be configured by giving it a maximum number of characters by using `maxWordsCount` or switching the mode to sentences by setting `sentencesMode` to `true` in which case `maxWordsCount` is used to cap the number of sentences allowed. **This arbitrary is not shrinkable**

### Combinors of arbitraries (:T)

- `fc.constant<T>(value: T): Arbitrary<T>` constant arbitrary only able to produce `value: T`
- `fc.oneof<T>(...arbs: Arbitrary<T>[]): Arbitrary<T>` randomly chooses an arbitrary at each new generation. Should be provided with at least one arbitrary. All arbitraries are equally probable and shrink is still working for the selected arbitrary
- `fc.option<T>(arb: Arbitrary<T>): Arbitrary<T | null>` or `fc.option<T>(arb: Arbitrary<T>, freq: number): Arbitrary<T | null>` arbitrary able to nullify its generated value. When provided a custom `freq` value it changes the frequency of `null` values so that they occur one time over `freq` tries (eg.: `freq=5` means that 20% of generated values will be `null` and 80% would be produced through `arb`). By default: `freq=5`
- `fc.array<T>(arb: Arbitrary<T>): Arbitrary<T[]>`, `fc.array<T>(arb: Arbitrary<T>, maxLength: number): Arbitrary<T[]>` or `fc.array<T>(arb: Arbitrary<T>, minLength: number, maxLength: number): Arbitrary<T[]>` array of random length containing values generated by `arb`. By setting the parameters `minLength` and/or `maxLength`, the user can change the minimal (resp. maximal) size allowed for the generated array. By default: `minLength=0` and `maxLength=10`
- `fc.tuple<T1,T2,...>(arb1: Arbitrary<T1>, arb2: Arbitrary<T2>, ...): Arbitrary<[T1,T2,...]>` tuple generated by aggregating the values of `arbX` like `generate: () => [arb1.generate(), arb2.generate(), ...]`. This arbitrary perfectly handle shrinks and is able to shink on all the generators
- `fc.dictionary<T>(keyArb: Arbitrary<string>, valueArb: Arbitrary<T>): Arbitrary<{[Key:string]:T}>` dictionary containing keys generated using `keyArb` and values gneerated by `valueArb`

### Objects (:any)

The framework is able to generate totally random objects in order to adapt to programs that do not requires any specific data structure. All those custom types can be parametrized using `ObjectConstraints.Settings`.

```typescript
export module ObjectConstraints {
    export interface Settings {
        maxDepth?: number;         // maximal depth allowed for this object
        key?: Arbitrary<string>;   // arbitrary for key
        values?: Arbitrary<any>[]; // arbitrary responsible for base value
    };
};
```

Default for `key` is: `fc.string()`.

Default for `values` are: `fc.boolean()`, `fc.integer()`, `fc.double()`, `fc.string()` and constants among `null`, `undefined`, `Number.NaN`, `Number.MIN_VALUE`, `Number.MAX_VALUE` , `Number.MIN_SAFE_INTEGER` or `Number.MAX_SAFE_INTEGER`.

- `fc.anything()` or `fc.anything(settings: ObjectConstraints.Settings)` generate a possible values coming from Settings and all objects or arrays derived from those same settings
- `fc.object()` or `fc.object(settings: ObjectConstraints.Settings)` generate an object
- `fc.jsonObject()` or `fc.jsonObject(maxDepth: number)` generate an object that is eligible to be stringified and parsed back to itself (object compatible with json stringify)
- `fc.unicodeJsonObject()` or `fc.unicodeJsonObject(maxDepth: number)` generate an object with potentially unicode characters that is eligible to be stringified and parsed back to itself (object compatible with json stringify)

## Custom arbitraries

### Derive existing arbitraries

All generated arbitraries inherit from the same base class: [Arbitrary](https://github.com/dubzzz/fast-check/blob/master/src/check/arbitrary/definition/Arbitrary.ts).

It cames with two useful methods: `filter(predicate: (t: T) => boolean): Arbitrary<T>` and `map<U>(mapper: (t: T) => U): Arbitrary<U>`. These methods are used internally by the framework to derive some Arbitraries from existing ones.

#### Filter values

`filter(predicate: (t: T) => boolean): Arbitrary<T>` can be used to filter undesirable values from the generated ones. It can be used as some kind of pre-requisite for the parameters required for your algorithm. For instance, you might need to generate two ordered integer values. One approach can be to use filter as follow:

```typescript
const minMax = fc.tuple(fc.integer(), fc.integer())
                    .filter(t => t[0] < t[1]);
```

But be aware that using `filter` may highly impact the time required to generate a valid entry. In the previous example, half of the generated tuples will be rejected. It can nontheless be a very useful and powerful tool to derive your arbitraries quickly and easily.

#### Transform values

`map<U>(mapper: (t: T) => U): Arbitrary<U>` in its side does not filter any of the generated entries. It take one entry (generated or shrinked) and map it to another.

For instance the previous example could have been refactored as follow:

```typescript
const minMax = fc.tuple(fc.integer(), fc.integer())
                    .map(t => t[0] < t[1] ? [t[0], t[1]] : [t[1], t[0]]);
```


Another example would be to derive `fc.integer()` and `fc.array()` to build `fc.char()` and `fc.string()`:

```typescript
const char = () => fc.integer(0x20, 0x7e).map(String.fromCharCode);
const string = () => fc.array(fc.char()).map(arr => arr.join(''));
```

Most of the [built-in arbitraries](https://github.com/dubzzz/fast-check/tree/master/src/check/arbitrary) use this trick to define themselves.

### Build your own

You can also fully customize your arbitrary and not derived from any of the buit-in arbitraries. What you have to do is to derive from [Arbitrary](https://github.com/dubzzz/fast-check/blob/master/src/check/arbitrary/definition/Arbitrary.ts) and implement `generate(mrng: MutableRandomGenerator): Shrinkable<T>`.

`generate` is responsable for the generation of one new random entity of type `T` (see signature above). In ordedr to fulfill it in a deterministic way it received a `mrng: MutableRandomGenerator`. It can then use it to generate integer values within `min` (included) and `max` (included) by using `UniformDistribution.inRange(min, max)(mrng)[0]`.

The generated value also came with a shrink method able to derive _smaller_ values in case of failure. It can be ignored making the arbitrary not shrinkable.

Once again the [built-in types](https://github.com/dubzzz/fast-check/tree/master/src/check/arbitrary) can be very helpful if you need an example.

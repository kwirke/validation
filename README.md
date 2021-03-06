[![npm version](https://badge.fury.io/js/%40rexform%2Fvalidation.svg)](https://badge.fury.io/js/%40rexform%2Fvalidation)
[![Build Status](https://travis-ci.org/kwirke/validation.svg?branch=master)](https://travis-ci.org/kwirke/validation)
[![codecov](https://codecov.io/gh/kwirke/validation/branch/master/graph/badge.svg)](https://codecov.io/gh/kwirke/validation)

# validation

Validation is a [Static Land](https://github.com/rpominov/static-land) compatible monad and ADT (Algebraic Data Type) designed to store immutable validation results while not losing any validation errors nor the validated value.

Its power shines when using atomic validator functions that return Either an error or the value (modified or not).

It implements the algebras Monad (map, ap, of, chain), Monoid for array values (concat, empty), and Semigroup for errors (concatErr).

## Install

```
npm i @rexform/validation
```

## Type

A Validation can be only one of these:

- A `Valid<T>` value of type T, or
- An `Invalid<E, T>` value of type T with a non empty errors array of type E[].

A value and an error can be whatever type you want: strings, booleans, and even objects. All the errors in the array must have the same type though.

## Example

```javascript
import Validation from '@rexform/validation';

const trim = str => Either.right(str.trim());
const isNotEmpty = str =>
  str.length > 0 ? Either.right(str) : Either.left('Can`t be empty');
const hasNumbers = str =>
  /[0-9]/.test(str) ? Either.right(str) : Either.left('Must have numbers');

const validators = [trim, isNotEmpty, hasNumbers];

Validation.of('123456').validateAll(validators); // => Valid('123456')
Validation.of('123456 ').validateAll(validators); // => Valid('123456')
Validation.of('wrong zipcode').validateAll(validators); // => Invalid(['Must have numbers'], 'wrong zipcode')
Validation.of('   ').validateAll(validators); // => Invalid(['Can`t be empty', 'Must have numbers'], '')
```

## Use Cases

- [Composable decoders](https://dev.to/kwirke/javascript-composable-decoders-with-validation-1ldh)
- Form validations (where you need to keep showing the value even when it's invalid).
- Good error reporting for libraries configurations.

## API

### Constructors

#### - `new Valid(value: T): Valid<T>`

Returns a Valid type.

```javascript
import { Valid } from '@rexform/validation';

const v = new Valid(42);
```

#### - `valid<T>(value: T): Valid<T>`

Returns a Valid type.

```javascript
import { valid } from '@rexform/validation';

const v = valid(42);
```

#### - `new Invalid(value: T, errors: E[]): Invalid<E, T>`

Returns an Invalid type. Throws if errors is an empty array.

```javascript
import { Invalid } from '@rexform/validation';

const i = new Invalid('', ['Empty value']);
```

#### - `invalid(value: T, errors: E | E[]): Invalid<E, T>`

Returns an Invalid type. Throws if errors is an empty array. It is curried, and casts the error to an array if it is not.

```javascript
import { invalid } from '@rexform/validation';

const i = invalid('', ['Empty value']);
const i = invalid('')('Empty value');
```

#### - `of(value: T, errors?: E[]): Validation<E, T>`

If errors is empty or a nil value, returns a Valid type, otherwise returns an Invalid type with the errors.

```javascript
import { of } from '@rexform/validation';

const valid = of(42, []);
const invalid = of('', ['Empty value']);
```

#### - `fromEither(initialValue: T, either: Either<E | E[], T>): Validation<E, T>`

If either is a Right, it returns a Valid type ignoring `initialValue`. Otherwise, returns an Invalid type with the `initialValue` and the errors wrapped in the Left type. It is curried and casts Left contents to an array.

```javascript
import { fromEither } from '@rexform/validation';

const valid = fromEither(3, Either.Right(10));
const invalid = fromEither(3, Either.Left('error'));
```

#### - `fromPredicateOr(errorFn: (v: V) => E, predicate: (v: V) => boolean): (v: V) => Validation<E, V>`

Transforms a boolean predicate into a function that receives a value and returns a Validation. Curried.

```javascript
import { fromPredicateOr } from '@rexform/validation';

const isEven = (x: number) => x % 2 === 0;
const validateEven = fromPredicateOr(v => `${v} is not even.`, isEven);

validateEven(2); // => Valid(2)
validateEven(3); // => Invalid(3, ['3 is not even'])
```

### Dealing with Objects

#### - `property(propertyName: string, object: {}): Validation<string, any>`

Returns a Valid with the object's property value if found and not null, otherwise returns an Invalid with an error message.

```javascript
import { property } from '@rexform/validation';

property('a', { a: 10 }); // => Valid(10)
property('a', {}); // => Invalid(undefined, ['Property "a" not found or null.'])
```

#### - `allProperties(object: {[key: string]: Validation<E, V>}): Validation<E, {[key: string]: V}>`

Transforms an object of Validations into a Validation of the object. Keeps all Valid properties and all errors and discards all Invalid values.

```javascript
import { allProperties } from '@rexform/validation';

const obj = {
  a: valid(10),
  b: valid(undefined),
  c: invalid(10, ['Invalid number']),
  d: invalid(null, ['Property "d" not found']),
};

allProperties(obj); // => Invalid({a: 10, b: undefined}, ['Invalid number', 'Property "d" not found'])
```

#### - `validateProperties(validations: {[key: string]: (v: V) => Validation<E, V>}, object: {[key: string]: V}): {[key: string]: Validation<E, V>}`

Given an object describing the validations for each desired property, checks the presence of that property in an object and validates the value against the validations, returning an object of the resulting validations. Curried.

To check the presence of the property, it uses `Validation.property(propName)`, and then chains the passed validation.

```javascript
import { validateProperties, fromPredicateOr } from '@rexform/validation';

const obj = {
  a: 10,
  b: 'hi',
};

validateProperties(
  {
    a: fromPredicateOr(() => 'Must be below 10', v => v <= 10),
    b: value => invalid(value, 'Must not have b'),
    c: valid,
  },
  obj
);
/* => {
  a: valid(10),
  b: invalid('hi', ['Must not have b']),
  c: invalid(undefined, ['Property "c" not found or null.']),
} */
```

### Dealing with Arrays

#### - `empty(): Validation<any, any[]>`

Returns `valid([])`: A valid Validation of an empty array value.

#### - `concat(listValidation: Validation<E, V[]>, val: Validation<E, V[]>): Validation<E, V[]>`

Concatenates an array validation to another array validation, appending the values only if its valid, otherwise appending the errors.

When called as a method on an existing Validation, if it is not an array, it transforms the value to a one-item array.

```javascript
import { concat } from '@rexform/validation';

concat(valid([1]), valid([2])); // => Valid([1, 2])
concat(valid([1, 2]), valid([3])); // => Valid([1, 2, 3])
concat(valid([2]), invalid([3], ['error'])); // => Invalid([2], ['error'])

valid([2]).concat(valid([1])); // => Valid([1, 2])
```

#### - `sequence(validations: Validation<E, V>[]): Validation<E, V[]>`

Transforms an array of Validations into a Validation of an array using concat, keeping only valid values and all errors. It will return an Invalid unless all Validations are Valid.

```javascript
import { sequence } from '@rexform/validation';

sequence([
  valid(10),
  invalid(-5, ['Too low']),
  invalid(12, ['Too high']),
  valid(8),
]); // => Invalid([10, 8], ['Too low', 'Too high'])
```

### Other methods

All these functions are available both as methods of Valid and Invalid types and also as functions.

Example:

```javascript
import { valid, map } from '@rexform/validation';

// These are the same
valid(42).map(x => x + 1);
map(x => x + 1, valid(42));
```

They are also curried:

```javascript
import { map, chain, valid, invalid, validate } from '@rexform/validation';

pipe(
  map(x => x.trim()),
  chain(x =>
    containsNumbers(x) ? valid(x) : invalid(x, ['Must have numbers'])
  ),
  validate(x => (isEmpty(x) ? Either.Left(['Is empty']) : Either.Right(x)))
)(valid(' Validate me! '));
// => Invalid(['Must have numbers'], 'Validate me!')
```

**The types `Valid<T>` and `Invalid<T, E>` are assumed as the `this` in the following definitions.**

#### - `isValid(): boolean`

Returns true only if validation is Valid.

```javascript
valid('username').isValid(); // => true
invalid('', ['Empty value']).isValid(); // => false
```

#### - `isInvalid(): boolean`

Returns true only if validation is Invalid.

```javascript
valid('username').isInvalid(); // => false
invalid('', ['Empty value']).isInvalid(); // => true
```

#### - `value: T`

Returns the value wrapped in the type. Both Valid and Invalid store the value always, so it can be read safely.

```javascript
valid('username').value; // => 'username'
invalid('', ['Empty value']).value; // => ''
```

#### - `errorsOr(alt: T): E[] | T`

Returns the errors if validation is Invalid, alt otherwise.

```javascript
valid('username').errorsOr([]); // => []
invalid('', ['Empty value']).errorsOr([]); // => ['Empty value']
```

#### - `concat(v: Validation<E, U>): Validation<E, U>`

Returns a validation result of concatenating the errors of both validations, and keeps the value of the second validation (`v`).
Therefore, if both validations are valid, it returns `v`.

Note that the order of parameters is reversed from the other functions, to make it natural with the concatenation order.

```javascript
import { concat } from '@rexform/validation';

valid('discarded value').concat(valid('new value')); // => Valid('new value')
invalid('...', ['hello']).concat(invalid('test', ['world'])); // => Invalid('test', ['hello', 'world'])

// Note that the order is preserved in function style, so that currying feels natural
concat(invalid('test', ['world']), invalid('...', ['hello'])); // => Invalid('test', ['hello', 'world'])

const concatWorld = concat(invalid('test', ['world']));
concatWorld(invalid('...', ['hello'])); // => Invalid('test', ['hello', 'world'])
```

#### - `map(fn: T => U): Validation<E, U>`

Applies a function to the value and returns a new validation equally valid or invalid (with the same errors).

```javascript
valid(42).map(x => x + 1); // => Valid(43)
invalid(42, ['error']).map(x => x + 1); // => Invalid(43, ['error'])
```

#### - `mapErrors(mappingFn: (errors: E[]) => E2 | E2[], validation: Validation<E, V>): Validation<E2, V>`

Applies a function to the errors array and returns a new validation equally valid or invalid with the new errors array.
Casts `mappingFn` result to an array, and throws if it is an empty array.

```javascript
valid(42).mapErrors(e => ['error']); // => Valid(42)
invalid(42, ['error']).mapErrors(e => e.concat(['warning'])); // => Invalid(42, ['error', 'warning'])
invalid(42, ['error']).mapErrors(e => 'hi'); // => Invalid(42, ['hi'])
```

#### - `mapError(mappingFn: (error: E) => E2, validation: Validation<E, V>): Validation<E2, V>`

Applies a function to each of the errors in the validation and returns a new validation equally valid or invalid with the mapped errors.

```javascript
valid(42).mapError(e => 'Error: ' + e); // => Valid(42)
invalid(42, ['Wrong arguments']).mapError(e => 'Error: ' + e)); // => Invalid(42, ['Error: Wrong arguments'])
```

#### - `ap(val: Validation<E, (t: T) => U>): Validation<E, U>`

When passed a validation that contains a function as a value, applies that function to its value and returns a new validation with the concatenated errors of both.

```javascript
invalid(42, ['error']).ap(valid(x => x + 1)); // => Invalid(43, ['error'])
invalid('test', ['hello']).ap(invalid(s => s.length, ['world'])); // => Invalid(4, ['hello', 'world'])
```

#### - `chain(fn: (t: T) => Validation<E, U>): Validation<E, U>`

Applies a validation returning function to the value and returns a new validation with the concatenated errors of both this validation and the returned one, as well as the returned validation's value.

```javascript
valid('test').chain(str => invalid(str.length, ['error'])); // => Invalid(4, ['error'])
invalid('', ['Has no numbers']).chain(str =>
  str.length === 0 ? invalid(str, ['Empty value']) : valid(str)
); // => Invalid('', ['Has no numbers', 'Empty value'])
```

#### - `fold(fnInvalid: (e: E[], t: T, fnValid: (t: T) => U) => U): U`

If it is a Valid value, returns the result of applying fnValid to its value.
If it is an Invalid value, returns the result of applying fnInvalid to its value and errors.

```javascript
invalid('test', ['contain-numbers']).fold(
  (e, v) => `Value "${v}" has failed these validations: ${e}`,
  v => `Value "${v}" is OK!`
); // => 'Value "test" has failed these validations: contain-numbers'
```

It can be used to easily transform a Validation type into other types, like Maybe or Either:

```javascript
const eitherFromValidation = validation.fold(Either.Left, Either.Right); // Either<E[], V>
const maybeValidValue = validation.fold(Maybe.Nothing, Maybe.Just); // Maybe<V>
const maybeErrors = validation.fold(Maybe.Just, Maybe.Nothing); // Maybe<E[]>
```

## Either adapters

**Note:** You can use any type as Either here, as long as it has a function with the following signature:

```javascript
type Either<T, U> = {
    fold: <A>(this: Either<T, U>, left: (t: T) => A, right: (u: U) => A) => A
}
```

#### - `validateEither(either: Either<E[], T>): Validation<E, T>`

If the either is a Left, concatenates the errors to their own.
If the either is a Right, modifies the value.

```javascript
valid(42).validateEither(Right(10)); // => Valid(10)
valid('').validateEither(Left(['Empty value'])); // => Invalid('', ['Empty value'])
```

#### - `validateEitherList(eitherList: Either<E[], T>[]): Validation<E, T>`

Concatenates the errors of all the Left eithers in the list and keeps the value of the last Right either, or the validation value if none.

```javascript
valid('wrong zipcode').validateEitherList([
  Left(['Must have numbers']),
  Right(''),
]); // => Invalid('', ['Must have numbers'])
```

#### - `validate(validator: (t: T) => Either<E[], T>): Validation<E, T>`

Validates the value against the validator. If it returns a Right (passes), the new value is kept. If it returns a Left (fails), the errors are concatenated.

```javascript
const isNotEmpty = str =>
  str.length > 0 ? Either.right(str) : Either.left('Can`t be empty');
const hasNumbers = str =>
  /[0-9]/.test(str) ? Either.right(str) : Either.left('Must have numbers');

valid('wrong zipcode').validate(isNotEmpty); // => Valid('wrong zipcode')
valid('wrong zipcode').validate(hasNumbers); // => Invalid('wrong zipcode', ['Must have numbers'])
```

#### - `validateAll(validators: ((t: T) => Either<E[], T>)[]): Validation<E, T>`

Concatenates the errors returned by all failing validators (returning Left) and keeps the value of the last passing validator (returning Right), or the validation value if none.

```javascript
const trim = str => Either.right(str.trim());
const isNotEmpty = str =>
  str.length > 0 ? Either.right(str) : Either.left('Can`t be empty');
const hasNumbers = str =>
  /[0-9]/.test(str) ? Either.right(str) : Either.left('Must have numbers');

const validators = [trim, isNotEmpty, hasNumbers];

Validation.of('123456').validateAll(validators); // => Valid('123456')
Validation.of('123456 ').validateAll(validators); // => Valid('123456')
Validation.of('wrong zipcode').validateAll(validators); // => Invalid('wrong zipcode', ['Must have numbers'])
Validation.of('   ').validateAll(validators); // => Invalid('', ['Can`t be empty', 'Must have numbers'])
```

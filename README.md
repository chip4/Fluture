# Fluture

A complete [Fantasy Land][1] compatible Future library.

> `npm install --save fluture` <sup>Requires a node 5.0.0 compatible environment
  like modern browsers, transpilers or Node 5+</sup>


## Usage

Using the low level, high performance method API:

```js
const Future = require('fluture');
const program = file =>
  Future.node(done => fs.readFile(file, 'utf8', done))
  .chain(x => Future.try(() => JSON.parse(x)))
  .map(x => x.name)
  .fork(console.error, console.log);
program('package.json');
//> "fluture"
```

Or use the high level, fully curried, functional dispatch API for function
composition using composers like [`S.pipe`][2]:

```js
const {node, chain, try, map, fork} = require('fluture');
const program = S.pipe([
  file => node(fs.readFile.bind(fs, file, 'utf8')),
  chain(x => try(() => JSON.parse(x))),
  map(x => x.name),
  fork(console.error, console.log)
]);
program('package.json')
//> "fluture"
```

## Why?

* Great error messages.
* Async control utilities included.
* High performance Future implementation.
* Decent documentation.

## Documentation

### Constructors

#### Future :: ((a -> Void), (b -> Void) -> Void) -> Future a b

A (monadic) container which represents an eventual value. A lot like Promise but
more principled in that it follows the [Fantasy Land][1] algebraic JavaScript
specification.

```js
const eventualThing = Future((reject, resolve) => {
  setTimeout(resolve, 500, 'world');
});

eventualThing.fork(console.error, thing => console.log(`Hello ${thing}!`));
//> "Hello world!"
```

#### `of :: b -> Future a b`

A constructor that creates a Future containing the given value.

```js
const eventualThing = Future.of('world');
eventualThing.fork(console.error, thing => console.log(`Hello ${thing}!`));
//> "Hello world!"
```

#### `after :: Number -> b -> Future a b`

A constructor that creates a Future containing the given value after n milliseconds.

```js
const eventualThing = Future.after(500, 'world');
eventualThing.fork(console.error, thing => console.log(`Hello ${thing}!`));
//> "Hello world!"
```

#### `try :: (Void -> !a | b) -> Future a b`

A constructor that creates a Future which resolves with the result of calling
the given function, or rejects with the error thrown by the given function.

```js
const data = {foo: 'bar'}
Future.try(() => data.foo.bar.baz).fork(console.error, console.log)
//> [TypeError: Cannot read property 'baz' of undefined]
```

#### `node :: ((a, b -> Void) -> Void) -> Future a b`

A constructor that creates a Future which rejects with the first argument given
to the function, or resolves with the second if the first is not present.

This is a convenience for NodeJS users who wish to easily obtain a Future from
a node style callback API.

```js
Future.node(done => fs.readFile('package.json', 'utf8', done))
.fork(console.error, console.log)
//> "{...}"
```

#### `cache :: Future a b -> Future a b`

Returns a Future which caches the resolution value of the given Future so that
whenever it's forked, it can load the value from cache rather than reexecuting
the chain.

```js
const eventualPackage = Future.cache(
  Future.node(done => {
    console.log('Reading some big data');
    fs.readFile('package.json', 'utf8', done)
  })
);

eventualPackage.fork(console.error, console.log);
//> "Reading some big data"
//> "{...}"

eventualPackage.fork(console.error, console.log);
//> "{...}"
```

### Method API

#### `fork :: Future a b ~> (a -> Void), (b -> Void) -> Void`

Execute the Future (which up until now was merely a container for its
computation), and get at the result or error.

It is the return from Fantasy Land to the real world. This function best shows
the fundamental difference between Promises and Futures.

```js
Future.of('world').fork(
  err => console.log(`Oh no! ${err.message}`),
  thing => console.log(`Hello ${thing}!`)
);
//> "Hello world!"

Future.reject(new Error('It broke!')).fork(
  err => console.log(`Oh no! ${err.message}`),
  thing => console.log(`Hello ${thing}!`)
);
//> "Oh no! It broke!"
```

#### `map :: Future a b ~> (b -> c) -> Future a c`

Map over the value inside the Future. If the Future is rejected, mapping is not
performed.

```js
Future.of(1).map(x => x + 1).fork(console.error, console.log);
//> 2
```

#### `chain :: Future a b ~> (b -> Future a c) -> Future a c`

FlatMap over the value inside the Future. If the Future is rejected, chaining is
not performed.

```js
Future.of(1).chain(x => Future.of(x + 1)).fork(console.error, console.log);
//> 2
```

#### `chainRej :: Future a b ~> (a -> Future a c) -> Future a c`

FlatMap over the **rejection** value inside the Future. If the Future is
resolved, chaining is not performed.

```js
Future.reject(new Error('It broke!')).chainRej(err => {
  console.error(err);
  return Future.of('All is good')
})
.fork(console.error, console.log)
//> "All is good"
```

#### `ap :: Future a (b -> c) ~> Future a b -> Future a c`

Apply the value in the Future to the value in the given Future. If the Future is
rejected, applying is not performed.

```js
Future.of(x => x + 1).ap(Future.of(1)).fork(console.error, console.log);
//> 2
```

#### `race :: Future a b ~> Future a b -> Future a b`

Race two Futures against eachother. Creates a new Future which resolves or
rejects with the resolution or rejection value of the first Future to settle.

```js
Future.after(100, 'hello')
.race(Future.after(50, 'bye'))
.fork(console.error, console.log)
//> "bye"
```

### Dispatcher API

#### `fork :: (a -> Void) -> (b -> Void) -> Future a b -> Void`

Dispatches the first and second arguments to the `fork` method of the third argument.

#### `map :: Functor m => (a -> b) -> m a -> m b`

Dispatches the first argument to the `map` method of the second argument.
Has the same effect as [`R.map`][3].

#### `chain :: Chain m => (a -> m b) -> m a -> m b`

Dispatches the first argument to the `chain` method of the second argument.
Has the same effect as [`R.chain`][4].

#### `chainRej :: (a -> Future a c) -> Future a b -> Future a c`

Dispatches the first argument to the `chainRej` method of the second argument.

#### `ap :: Apply m => m (a -> b) -> m a -> m b`

Dispatches the second argument to the `ap` method of the first argument.
Has the same effect as [`R.ap`][5].

#### `race :: Future a b -> Future a b -> Future a b`

Dispatches the first argument to the `race` method of the second argument.

```js
const first = futures => futures.reduce(Future.race);
first([
  Future.after(100, 'hello'),
  Future.after(50, 'bye'),
  Future(rej => setTimeout(rej, 25, 'nope'))
])
.fork(console.error, console.log)
//> [Error nope]
```

### Futurization

#### `liftNode :: (x..., (a, b -> Void) -> Void) -> x... -> Future a b`

Turn a node continuation-passing-style function into a function which returns a Future.

Takes a function which uses a node-style callback for continuation and returns a
function which returns a Future for continuation.

```js
const readFile = Future.liftNode(fs.readFile);
readFile('README.md', 'utf8')
.map(text => text.split('\n'))
.map(lines => lines[0])
.fork(console.error, console.log);
//> "# Fluture"
```

#### `liftPromise :: (x... -> Promise a b) -> x... -> Future a b`

Turn a function which returns a Promise into a function which returns a Future.

Like liftNode, but for a function which returns a Promise.

## Road map

* [x] Implement Future Monad
* [x] Write tests
* [x] Write benchmarks
* [ ] Implement Traversable?
* [x] Implement Future.cache
* [ ] Implement Future#mapRej
* [x] Implement Future#chainRej
* [x] Implement dispatchers
* [ ] Implement Future#swap
* [ ] Implement Future#and
* [ ] Implement Future#or
* [ ] Implement Future#fold
* [ ] Implement Future#value
* [x] Implement Future#race
* [ ] Implement Future.parallel
* [ ] Implement Future.predicate
* [ ] Implement Future#promise
* [ ] Implement Future.cast
* [ ] Implement Future.encase
* [x] Check `this` on instance methods
* [x] Create documentation
* [ ] Wiki: Comparison between Future libs
* [ ] Wiki: Comparison Future and Promise
* [ ] Add test coverage
* [ ] A transpiled ES5 version if demand arises

## Benchmarks

Simply run `node ./bench/<file>` to see how a specific method compares to
implementations in `data.task`, `ramda-fantasy.Future` and `Promise`*.

\* Promise is not included in all benchmarks because it tends to make the
  process run out of memory.

## The name

A conjunction of the acronym to Fantasy Land (FL) and Future. Also "fluture"
means butterfly in Romanian; A creature you might expect to see in Fantasy Land.

----

[MIT licensed](LICENSE)

<!-- References -->

[1]:  https://github.com/fantasyland/fantasy-land
[2]:  https://github.com/plaid/sanctuary#pipe--a-bb-cm-n---a---n
[3]:  http://ramdajs.com/docs/#map
[4]:  http://ramdajs.com/docs/#chain
[5]:  http://ramdajs.com/docs/#ap

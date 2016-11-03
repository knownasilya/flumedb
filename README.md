# flumedb

a modular database made from moving logs with streams.

## architecture

Flume is a modular database compromised of an Append Only Log,
and then Streaming Views on that log.

The Log compromises the main storage, messages can be appended,
then that data is streamed through _views_. Views are deterministic,
the same data in the same order must give the same query results.

Unlike my previous work, [level-sublevel](https://github.com/dominictarr/level-sublevel)
views can be async.
This means things that where difficult to do with _sublevel_,
such as maintaining a count or sum of all the records, are now easy.

The trick is that each view exposes an [observable](https://github.com/dominictarr/obv)
that represents it's current state. An observable is like an event meets a value.
it's an changing value that you can _observe_, events, promises, or streams are similar,
but not quite the right choice here.

Data goes into the log, then is streamed to the view, view updates may be async,
which means when your log write succeeds the view _might not be consistent yet_.
So, when a read is performed on a view, flume will wait until that view is up to date
with the main log at the time you called the view read, _then_ pass the read to the view.

This means you can append to the log, and then in the callback, read a view,
and you'll get read your writes in the view!

This gives us the freedom to have async views,
which gives us the freedom to have many different sorts of views!

## example

Take one `flumelog-*` module and zero or more `flumeview-*` modules,
and glue them together with `flumedb`. 

``` js
var MemLog = require('flumelog-memory') // just store the log in memory
var Reduce = require('flumeview-reduce') //just a reduce function.
var Flume = require('flumedb')

var db = Flume(MemLog())
  //the api of flumeview-reduce will be mounted at db.sum...
  .use('sum', Reduce(function (acc, item) {
    return (acc || 0) + item.foo
  }))

db.append({foo: 1}, function (err, seq) {
  db.sum.get(function (err, value) {
    if(err) throw err
    console.log(value) // 1
  })
})

```

## api

## flumedb

### flumedb.get (seq, cb)

This exposes `get` from the underlying `flumelog` module.
This takes a `seq` as is the keys in the log,
and returns the value at that sequence or an error.

### flumedb.stream (opts)

This exposes `stream` from the underlying `flumelog` module.
It supports options familiar from leveldb ranges. lt, gt, lte, gte, reverse, live, limit.

### flumedb.since => observable

The [observable](https://github.com/dominictarr/obv) which represents the state of the log.
This will be set to -1 if the view is empty, will monotonically increase as the log is appended
to. The particular format of the number depends on the implementation of the log,
but should usually be a number, for example, a incrementing integer, a timestamp, or byte offset.

### flumedb.append (value, cb(err, seq))

append a value to the database. The encoding of this value will be handled by the `flumelog`,
will callback when value is successfully written to the log (or it errors).
On success, the callback will return the new maximum sequence in the log.

If `value` is an array of values, it will be treated as a batched write.
By the time of the callback, flumedb.since will have been updated.

### flumedb.use(name, createFlumeview) => self

install a `flumeview` module.
This will add all methods that the flumeview describes in it's `modules` property to `flumedb[name]`
If `name` is already a property of `flumedb` then an error will be thrown.
You probably want to add plugins at startup, but given the streaming async nature of flumedb,
it will work completely fine if you add them later!

### flumedb.rebuild (cb)

Destroy all views and rebuild them from scratch. If you have a large database and lots of views
this might take a while, because it must read completely through the whole database again.
`cb` will be called once the views are back up to date with the log. You can continue using
the various read apis and appending to the database while it is being rebuilt,
but view reads will be delayed until those views are up to date.

## flumelog

The log is the heart of flumedb, it's the cannonical store of data, and all state is stored there.
The views are generated completely from the data in the log, so they can be blown away and regenerated easily.

### flumelog.get (seq, cb)

retrive the value at position `seq` in the log.

### flumelog.stream(opts)

return a source stream over the log.
It supports options familiar from leveldb ranges. lt, gt, lte, gte, reverse, live, limit.

### flumelog.since => observable

an observable which represents the state of the log. If the log is uninitialized
it should be set to `undefined`, and if the log is empty it should be `-1` if the log has data,
it should be a number larger or equal to zero.

### flumelog.append (value, cb(err, seq))

append a value (or array of values) to the log, and return the new latest sequence.
`flumelog.since` is updated before calling `cb`.

## flumeview

A `flumeview` provides a read api, and a streaming sink that accepts data from the log
(this will be managed by `flumedb`)

### flumeview.since => observable

the current state of the view (this must be the sequence of the last value processed by the view)
a flumeview _must_ process items from the main log in order, otherwise inconsistencies will occur.

### flumeview.createSink (cb)

return a pull-stream sink that accepts data from the log. The input will be `{seq, value}` pairs.
`cb` will be called when the stream ends (or errors)

### flumeview.destroy (cb)

Wipe out the flumeview's internal (and persisted) state, and reset `flumeview.since` to `-1`

### flumeview.methods = {<key>:'sync'|'async'|'source'}

An object describing the methods exposed by this view.
A view needs to expose at least one method
(otherwise, why is it useful?)

These the corresponding methods will be added at `flumedb[name][key]`.
If they type is `async` or `source` the actual call to the `flumeview[key]` method will
be delayed until `flumeview.since` is in up to date with the log.
`sync` type methods will be called immediately.

### flumeview[name].ready (cb)

a `ready` method is also added to each mounted `flumeview` which takes a callback
which will be called exactly once, when that view is up to date with the log
(from the point where it is called)

## License

MIT



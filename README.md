`lmdb-store` is an interface to the incredibly fast LMDB, and is probably the fastest and most efficient NodeJS key-value/database interface that exists for the storing and retrieving JavaScript structured data in a true persisted, scalable, ACID-compliant, database. It provides a simple interface for interacting with LMDB, as a key-value store, that makes it easy to properly leverage the power, crash-proof design, and efficiency of LMDB using intuitive JavaScript. `lmdb-store` offers several key features that make it idiomatic, highly performant, and easy to use LMDB efficiently:
* High-performance translation of JS values and data structures to/from binary data
* Queueing asynchronous write operations with promise-based API
* Automated database size handling
* Transaction management
* Iterable queries/cursors
* Optional native off-main-thread compression with high-performance LZ4 compression
* Record versioning and optimistic locking

`lmdb-store` is built on the excellent [node-lmdb](https://github.com/Venemo/node-lmdb) package.

## Design

`lmdb-store` is designed to handle translation of JavaScript values, primitives, arrays, and objects, to and from the binary storage of LMDB with highly optimized code using native C++ code for unbelievable performance. It supports multiple types of JS values for keys and values, making it easy to use idiomatic JS for storing and retrieving data.

`lmdb-store` is designed for synchronous reads, and asynchronous writes. In idiomatic NodeJS code, I/O operations are performed asynchronously. `lmdb-store` observes this design pattern; because LMDB is a memory-mapped database, read operations do not use any I/O (other than the slight possibility of a page fault), and can almost always be performed faster than Node's event queue callbacks can even execute, and it is easier to write code for instant synchronous values from reads. On the otherhand, in default mode with sync'ed/flushed transactions, write operations do involve I/O, and furthermore can achieve vastly higher throughput by batching operations. The entire transaction of batch operation are performed in a separate thread. Consequently, `lmdb-store` is designed for writes to go through this asynchronous batching process and return a simple promise that resolves once the write is completed and flushed to disk.

LMDB supports multiple modes of transactions, including disabling of file sync'ing (noSync), which makes transaction commits much faster. We _highly_ discourage turning off sync'ing as it leaves the database prone to data corruption. With the default sync'ing enabled, LMDB has a crash-proof design; a machine can be turned off at any point, and data can only be corrupted if the written data is actually corrupted/changed. This does make transactions slower (although not necessarily less efficient). However, by batching writes, when a database is under load, slower transactions just enable more writes per transaction, and lmdb-store is able to drive LMDB to achieve the same levels of throughput with safe sync'ed transactions as without, while still preserving the durability/safety of sync'ed transactions.

`lmdb-store` supports and encourages the use of conditional writes; this allows for atomic operations that are dependendent on previously read data, and most transactional types of operations can be written with an optimistic-locking based, atomic-conditional-write pattern.

When an `lmdb-store` is created, an LMDB environment/database is created, and starts with a default DB size of 1MB. LMDB itself uses a fixed size, but `lmdb-store` detects whenever the database goes beyond the current size, and automatically increases the size of DB, and re-executes the write operations after resizing. With this, you do not have to make any estimates of database size, the databases automatically grow as needed (as you would expect from a database!)

`lmdb-store` provides optional compression using LZ4 that works in conjunction with the asynchronous writes by performing the compression in the same thread (off the main thread) that performs the writes in a transaction. LZ4 is extremely fast, and decompression can be performed at roughly 5GB/s, so excellent storage efficiency can be achieved with almost negligible performance impact.

## Usage
An lmdb-store instances is created with by using `open` export from the main module:
```
const { open } = require('lmdb-store');
// or
// import { open } from 'lmdb-store';
let myStore = open({
	path: 'my-store',
	// any options go here, we can turn on compression like this:
	compression: true,
});
await myStore.put('greeting', 'Hello, World!')
myStore.get('greeting') // 'Hello, World!'
```
(see store options below for more options)

Once you have created a store, you can store and retrieve values using keys.

### Keys
When using the various APIs, keys can be any JS primitive (string, number, boolean), an array of primitives, or a Buffer. These primitives are translated to binary keys used by LMDB in such a way that consistent ordering is preserved. Numbers are ordered naturally, which come before strings, which are ordered lexically. The keys are stored with type information preserved. The `getRange`operations that return a set of entries will return entries with the original JS primitive values for the keys. If arrays are used as keys, they are ordering by first value in the array, with each subsequent element being a tie-breaker. Numbers are stored as doubles, with reversal of sign bit for proper ordering plus type information, so any JS number can be used as a key. For example, here are the order of some different keys :
```
-10 // negative supported
-1.1 // decimals supported
400
3E10
'Hello'
['Hello', 'World']
'World'
'hello'
['hello', 1, 'world']
['hello', 'world']
```
You can override the default encoding of keys, and cause keys to be returned as node buffers using the `keyIsBuffer` database option (generally slower).

### Values
You can store a wide variety of JavaScript values and data structures in lmdb-store, including objects (with arbitray complexity), arrays, buffers, strings, numbers, etc. in your database. Values are stored and retrieved according the database encoding, which can be set using the `encoding` property on the database options. By default, data is stored using MessagePack, but there are four supported encodings:

* `msgpack` (default) - All values are stored by serializing the value as MessagePack (using the msgpackr package). Values are decoded and parsed on retrieval, so `get` and `getRange` will return the object, array, or other value that you have stored. The msgpackr package is extremely fast (faster than native JSON), and provides the most flexibility in storing different value types. See the Shared Structures section for how to achieve maximum efficiency with this.
* `json` - All values are stored by serializing the value as JSON (using JSON.stringify) and encoded with UTF-8. Values are decoded and parsed on retrieval using JSON.parse. Generally this does not perform as all as msgpack, nor support as many value types.
* `string` - All values should be strings and stored by encoding with UTF-8. Values are returned as strings from `get`.
* `binary` - Values are returned as (Node) buffer objects, representing the raw binary data. Note that creating buffer objects in NodeJS has some overhead and while this is fast and valuable direct storage of binary data, the data encodings provides faster and more optimized process for serializing and deserializing structured data.

Once you have a store, the following methods are available:

### `store.get(key): any`
This will retrieve the value at the specified key. The `key` must be a JS value/primitive as described above, and the return value will be the stored data (dependent on the encoding), or `undefined` if the entry does not exist.

### `store.put(key, value, version?: number, ifVersion?: number): Promise<boolean>`
This will store the provided value/data at the specified key. If the database is using versioning (see options below), the `version` parameter will be used to set the version number of the entry. If the `ifVersion` parameter is set, the put will only occur if the existing entry at the provided key has the version specified by `ifVersion` at the instance the commit occurs (LMDB commits are atomic by default). If the `ifVersion` parameter is not set, the put will occur regardless of the previous value.

This operation will be enqueued to be written in a batch transaction. Any other operations that occur within a certain timeframe (1ms by default) will also occur in the same transaction. This will return a promise for the completion of the put. The promise will resolve once the transaction has finished committing. The resolved value of the promise will be `true` if the `put` was successful, and `false` if the put did not occur due to the `ifVersion` not matching at the time of the commit.

If this is performed inside a transation, the put will be included in the current transaction (synchronously).

### `store.remove(key, ifVersion?: number): Promise<boolean>`
This will delete the entry at the specified key. This functions like `put`, with the same optional conditional version. This is batched along with put operations, and returns a promise indicating the success of the operation.

Again, if this is performed inside a transation, the removal will be included in the current transaction (synchronously).

### `store.putSync(key, value: Buffer, ifVersion?: number): boolean`
This will set the provided value at the specified key, but will do so synchronously. If this is called inside of a synchronous transaction, this put will be added to the current transaction. If not, a transaction will be started, the put will be executed, and the transaction will be committed, and then the function will return. We do not recommend this be used for any high-frequency operations as it can be vastly slower (for the main JS thread) than the `put` operation (often taking multiple milliseconds).

### `store.removeSync(key, ifVersion?: number): boolean`
This will delete the entry at the specified key. This functions like `putSync`, providing synchronous entry deletion.

### `store.transaction(execute: Function)`
This will begin synchronous transaction, execute the provided function, and then commit the transaction. The provided function can perform `get`s, `put`s, and `remove`s within the transaction, and the result will be committed. The execute function can return a promise to indicate an ongoing asynchronous transaction, but generally you want to minimize how long a transaction is open on the main thread, at least if you are potentially operating with multiple processes.

### `getRange(options: { start?, end?, reverse?: boolean, limit?: number}): Iterable<{ key, value: Buffer }>`
This starts a cursor-based query of a range of data in the database, returning an iterable that also has `map`, `filter`, and `forEach` methods. The `start` and `end` indicate the starting and ending key for the range. The `reverse` flag can be used to indicate reverse traversal. The `limit` can limit the number of entries returned. The returned cursor/query is lazy, and retrieves data _as_ iteration takes place, so a large range could specified without forcing all the entries to be read and loaded in memory upfront, and one can exit out of the loop without traversing the whole range in the database. The query is iterable, we can use it directly in a for-of:
```
for (let { key, value } of db.getRange({ start, end })) {
	// for each key-value pair in the given range
}
```
Or we can use the provided methods:
```
db.getRange({ start, end })
	.filter(({ key, value }) => test(key))
	.forEach(({ key, value }) => {
		// for each key-value pair in the given range that matched the filter
	})
```
Note that `map` and `filter` are also lazy, they will only be executed once their returned iterable is iterated or `forEach` is called on it. The `map` and `filter` functions also support async/promise-based functions, and you can create async iterable if the callback functions execute asynchronously (return a promise).

### openDB(database: string|{name:string,...})
LMDB supports multiple databases per environment (an environment is a single memory-mapped file). When you initialize an LMDB store with `open`, the store uses the default root database. However, you can use multiple databases per environment and instantiate a store for each one. If you are going to be opening many databases, make sure you set the `maxDbs` (it defaults to 12). For example, we can open multiple stores for a single environment:
```
const { open } = require('lmdb-store');
let rootStore = open('all-my-data');
let usersStore = myStore.openDB('users');
let groupsStore = myStore.openDB('groups');
let productsStore = myStore.openDB('products');
```
Each of the opened/returned stores has the same API as the default store for the environment. Each of the stores for one environment also share the same batch queue and automated transactions with each other, so immediately writing data from two stores with the same environment will be batched together in the same commit. For example:
```
usersStore.put('some-user', { data: userInfo });
groupsStore.put('some-group', { groupData: moreData });
```
Both these puts will be batched and after 1ms be committed in the same transaction.

### getLastVersion(): number
This returns the version number of the last entry that was retrieved with `get` (assuming it was a versioned database).

## Versioning
Versioning is the preferred method for achieving atomicity with data updates. A version can be stored with an entry, and later the data can be updated, conditional on the version being the expected version. This provides a robust mechanism for concurrent data updates even with multiple processes accessing the same database. To enable versioning, make sure to set the `useVersions` option when opening the database:
```
let myStore = open('my-store', { useVersions: true })
```
You can set a version by using the `version` argument in `put` calls. You can later update data and ensure that the data will only be updated if the version matches the expected version by using the `ifVersion` argument. When retrieving entries, you can access the version number by calling `getLastVersion()`.

## Shared Structures
Shared structures are mechanism for storing the structural information about objects stored in database in a way that can be resused across of the data in database, for much more efficient storage and faster retrieval of data when storing objects that have the same or similar structures (note that this is only available using the default MessagePack encoding, using the msgpackr package). We highly recommend using this when storing structured objects in lmdb-store. When enabled, when data is stored, any structural information (the set of property names) is automatically generated and stored in separate entry to be reused for storing and retrieving all data for the database. To enable this feature, simply specify the key where lmdb-store can store the shared structures. We recommend using a binary key outside of the range of the standard JS values (which start at byte 5):
```
let myStore = open('my-store', {
	sharedStructuresKey: Buffer.from([ 2 ])
})
```
Once shared structures has been enabled, you can store JavaScript objects just as you would normally would, and lmdb-store will automatically generate, increment, and save the structural information in the provided key to improve storage efficiency and performance. You never need to directly access this key, just be aware that that entry is being used by lmdb-store.

This option is recommended when storing with data with similiar object structures in a database (including inside of array).

## Compression
lmdb-store can optionally use off-thread LZ4 compression as part of the asynchronous writes to enable efficient compression with virtually no overhead to the main thread. LZ4 decompression (in `get` and `getRange` calls) is extremely fast and generally has little impact on performance. Compression is turned off by default, but can be turned on by setting the `compression` property when opening a database. The value of compression can be `true` or an object with compression settings, including properties:
* `threshold` - Only entries that are larger than this value (in bytes) will be compressed. This defaults to 1000 (if compression is enabled)
* `dictionary` - This can be buffer to use as a shared dictionary. This is defaults to a shared dictionary in lmdb-store that helps with compressing JSON and English words in small entries. Zstandard provides utilities for creating your own optimized shared dictionary.
For example:
```
let myStore = open('my-store', {
	compression: {
		threshold: 500, // compress any entry larger than 500 bytes
		dictionary: fs.readFileSync('dict.txt') // use your own shared dictionary
	}
})
```
Compression is recommended for large databases that may be larger than available RAM, to improving OS caching.

### Store Options
The open method can be used to create the main database/environment with the following signature:
`open(path, options)` or `open(options)`
Additional databases can be opened within the main database environment with:
`store.openDB(name, options)` or `store.openDB(options)`
If the `path` has an `.` in it, it is treated as a file name, otherwise it is treated as a directory name, where the data will be stored. The `options` argument to either of the functions should be an object, and supports the following properties, all of which are optional:
* `name` - This is the name of the database.
* `encoding` - Sets the encoding for the database, which can be `'msgpack'`, `'json'`, `'string'`, or `'binary'`.
* `sharedStructuresKey` - Enables shared structures and sets the key where the shared structures will be stored.
* `compression` - This enables compression. This can be set a truthy value to enable compression with default settings, or it can be an object with compression settings.
* `useVersions` - Set this to true if you will be setting version numbers on the entries in the database.
* `keyIsBuffer` - This will cause the database to expect and return keys as node buffers.

The following additional option properties are only available when creating the main database environment (`open`):
* `path` - This is the file path to the database environment file you will use.
* `maxDbs` - The maximum number of databases to be able to open (there is some extra overhead if this is set very high).
* `commitDelay` - This is the amount of time to wait (in milliseconds) for batching write operations before committing the writes (in a transaction). This defaults to 1ms. A delay of 0 means more immediate commits, but a longer delay can be more efficient at collecting more writes into a single transaction and reducing I/O load.
* `immediateBatchThreshold` - This parameter defines a limit on the number of batched bytes in write operations that can be pending for a transaction before ldmb-store will schedule the asynchronous commit for the immediate next even turn (with setImmediate). The default is 10,000,000 (bytes).
* `syncBatchThreshold` - This parameter defines a limit on the number of batched bytes in write operations that can be pending for a transaction before ldmb-store will be force an immediate synchronous commit of all pending batched data for the store. This provides a safeguard against too much data being enqueued for asynchronous commit, and excessive memory usage, that can sometimes occur for a large number of continuous `put` calls without waiting for an event turn for the timer to execute. The default is 200,000,000 (bytes).

In addition, the following options map to LMDB's env flags, <a href="http://www.lmdb.tech/doc/group__mdb.html">described here</a> (none of these are recommended, but are available for adjusting performance):
* useWritemap - Use writemaps, this improves performance by reducing malloc calls, but can increase risk of a stray pointer corrupting data.
* noSubdir - Treat `path` as a filename instead of directory (this is the default if the path appears to end with an extension and has '.' in it)
* noSync - Doesn't sync the data to disk. We highly discourage this flag, since it can result in data corruption and lmdb-store mitigates performance issues associated with disk syncs by batching.
* noMetaSync - This isn't as dangerous as `noSync`, but doesn't improve performance much either.
* readOnly - Self-descriptive.
* mapAsync - Not recommended, lmdb-store provides the means to ensure commits are performed in a separate thread (asyncronous to JS), and this prevents accurate notification of when flushes finish.

## Events

The `lmdb-store` instance is an <a href="https://nodejs.org/dist/latest-v11.x/docs/api/events.html#events_class_eventemitter">EventEmitter</a>, allowing application to listen to database events. There is just one event right now:

`beforecommit` - This event is fired before a batched operation begins to start a transaction to write all queued writes to the database. The callback function can perform additional (asynchronous) writes (`put` and `remove`) and they will be included in the transaction about to be performed (this can be useful for updating a global version stamp based on all previous writes, for example).

## License

`lmdb-store` is licensed under the terms of the MIT license.

## Related Projects

lmdb-store is built on top of [node-lmdb](https://github.com/Venemo/node-lmdb)
lmdb-store uses msgpackr for the default serialization of data [msgpackr](https://github.com/kriszyp/msgpackr)
cobase is built on top of lmdb-store: [cobase](https://github.com/DoctorEvidence/cobase)

<a href="https://dev.doctorevidence.com/"><img src="./assets/powers-dre.png" width="203"/></a>

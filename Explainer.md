# IndexedDB putAll

**Authors**

[nums@google.com](mailto:nums@google.com)

**Participate**

Original discussion here: [https://github.com/w3c/IndexedDB/issues/69](https://github.com/w3c/IndexedDB/issues/69).

Tag Review: [https://github.com/w3ctag/design-reviews/issues/536](https://github.com/w3ctag/design-reviews/issues/536).

Please post issues/suggestions in this repository's issue tracker.

**What is this?**

This is a proposal for an enhancement to the IndexedDB API that will provide developers with a way to bulk-insert many records into an IndexedDB object store.

## Why?

Currently, the only way to insert records using IndexedDB is via put(), which allows for the insertion of a single record into an object store. If the object store has a key path then it is said to use _in-line keys_; if the object store does not have a key path it is said to _use out-of-line keys_. To illustrate, here is an example (copied from the w3c IndexedDB API documentation: https://w3c.github.io/IndexedDB/) that inserts “books” into a “library” database.

```javascript
const request = indexedDB.open("library");
.
.
.
const db = request.result;
```


Here is an example of insertion using _in-line_ keys. The key path is set to the author field, so the author of each book record will serve as its key.


```javascript
const store = db.createObjectStore("books", {keyPath: "author"})
let put_req = store.put({title: "Quarry Memories", author: "Fred"})
put_req.onsuccess = () => { // do something }
put_req.onerror = () => { // do something }
```


Here is an example of insertion using _out-of-line_ keys. Since no key path is defined on the object store, a key must be explicitly stated as the second parameter of put(). 


```javascript
const store = db.createObjectStore("books");
let put_req = store.put({title: "Quarry Memories", author: "Fred"}, "key_1")
put_req.onsuccess = () => { // do something }
put_req.onerror = () => { // do something }
```


Currently, if a developer wants to store large numbers of objects, they must loop put().


```javascript
for(let i = 0; i < 100; i++) {
  store.put({ field1: 'value1', field2: 'value2', field3: 'value3'})
}
```


## Benefits

- putAll addresses repeated customer feedback stating the desire to have the ability to bulk-insert large amounts of data into an object store.
- putAll also has the potential for performance improvements.

## Examples

The putAll API endpoints address repeated feedback from customers who have stated the desire to have the ability to bulk-insert large amounts of data into an object store.

putAllEntries(sequence<sequence<Values>>)

```javascript
let entries = new Map();
entries.set("key_1", {title: "Quarry Memories", author: "Fred"})
entries.set("key_2", {title: "Bedrock Nights", author: "Barney"})
let putall_req = store.putAllEntries(entries)
putall_req.onsuccess = () => { // do something }
putall_req.onerror = () => { // do something }
```


putAllValues(Iterable)


```javascript
let value1 = {title: "Quarry Memories", author: "Fred"}
let value2 = {title: "Bedrock Nights", author: "Barney"}
let values = [value1, value2]
let putall_req = store.putAllValues(values)
putall_req.onsuccess = () => { // do something }
putall_req.onerror = () => { // do something }
```


## Exceptions & Edge Case Handling

### Invalid values

There are many ways for a value to be invalid. For example, a value could fail constraints such as



*   not having a key when a key path is specified
*   failing the uniqueness constraint

In the case that one of the values to be inserted is invalid, the whole batch fails.

### Multiple values with the same key

One edge case is if the user specifies multiple values with the same key. In this case the behavior would remain similar to if multiple values with the same key were inserted using put() - only the last value would actually get inserted.

## Alternatives Considered

### Combining putAllEntries and putAllValues into 1 overridden function

I considered combining putAllEntries() and putAllValues() into 1 unified putAll() that would be overridden such that it could handle the insertion of in-line keys or the insertion of out-of-line keys but not both at the same time.

For example, for the insertion of in-line keys:


```javascript
let values = [{title: "some_title"}, {title: "some_other_title"}]
store.putAll(values)
```


and for the insertion of out-of-line keys:


```javascript
let entries = new Map([
    ["key_1", {title: "title_1"}],
    ["key_2", {title: "title_2"}]
])
store.putAll(entries)
```


One advantage with this implementation is that there is only 1 overridden function which is consistent with how put() is implemented. This method, however, isn’t as clear as having 2 unique functions (putAllEntries() and putAllValues()) that distinguish between the insertion for in-line and out-of-line keys. With 1 unified putAll() it is possible that users may try to insert both in-line and out-of-line keys which would result in a failure.

### Using separate key and value arrays instead of maps for _out-of-line key_ insertion

Building on the previous example, 2 separate arrays of values and keys could be used in place of a map:

```javascript
let values = [{title: "some_title"}, {title: "some_other_title"}]
let keys = ["key1", "key2"]
store.putAllEntries(values, keys)
```

This also maintains a syntax similar to put() but the drawback is that the keys and values are far apart.

### Nesting Arrays for _out-of-line_ key insertion

putAll could use nested arrays for the insertion of out-of-line keys instead of maps

```javascript
let entries = [
    ["key_1", {title: "title_1"}],
    ["key_2", {title: "title_2"}]
]
store.putAllEntries(entries)
```

### Using JavaScript Objects instead of Map

I considered using JavaScript objects instead of maps but chose not to since the keys in JavaScript objects can only be strings, whilst the keys in indexedDB are not limited to strings.

### Allowing Insertion of valid records in spite of Failures

Insertion of valid records could be allowed whilst records that fail to be inserted are discarded. With this implementation, the whole batch wouldn't fail just because one record was invalid:

```javascript
const store = db.createObjectStore("books", {keyPath: "title"})
store.putAllValues(
  {title: "some_title"}, // this record would get inserted
  {field: "some_field"}, // this record would not get inserted since it does not have the key
);
```

Though this could be beneficial for very large insertions where a few failures are permissible, this isn't consistent with other parts of the API. For example, 1 unhandled error in a transaction causes the entire transaction to fail. Additionally it's unclear whether or not the failure of one record would trigger an onsuccess or onerror event. 

### Using a non-event based asynchronous design

The asynchronous event-based system of IndexedDB is certainly outdated in comparison to newer non-event based systems like promises. 
> Practically speaking, Indexed DB is predominantly used indirectly by users who instead select libraries, many of which wrap the usage in promises. It's important to make sure that additions to the API surface can be integrated into libraries - i.e. by following the same transaction model. - _Joshua Bell_.

https://github.com/w3ctag/design-reviews/issues/536#issuecomment-661436475

## Future Considerations

### addAll

addAll() could correspond to add() as putAll() does to put(). Additionally, addAll() could essentially have the same syntax as putAll(), except it would throw an error on duplicate key insertion like add():

Inserting with _in-line_ keys

```javascript
let value1 = {title: "Quarry Memories", author: "Fred"}
let value2 = {title: "Bedrock Nights", author: "Barney"}
let values = [value1, value2]
let addall_req = store.addAllValues(values)
addall_req.onsuccess = () => { // do something }
addall_req.onerror = () => { // do something }
```

Inserting with _out-of-line_ keys

```javascript
let entries = new Map();
entries.set("key_1", {title: "Quarry Memories", author: "Fred"})
entries.set("key_2", {title: "Bedrock Nights", author: "Barney"})
let addall_req = store.addAllEntries(entries)
addall_req.onsuccess = () => { // do something }
addall_req.onerror = () => { // do something }
```

Inserting with duplicate keys

```javascript
// assuming the primary key is author
let value1 = {title: "Quarry Memories", author: "Fred"}
let value2 = {title: "Fred's second book", author: "Fred"}
let values = [value1, value2]
let addall_req = store.addAllValues(values)
addall_req.onsuccess = () => { // won't get called }
addall_req.onerror = () => { // this will get called since the key is duplicated }
```

### deleteAll

## References & Acknowledgments

References:

- [w3c/IndexedDB](https://w3c.github.io/IndexedDB/) - IndexedDB spec

Acknowledgements:

Thanks to the following people who helped me create this explainer
- Daniel Murphy <dmurph@chromium.org>
- Joshua Bell <jsbell@chromium.org>
- Victor Costan <pwnall@chromium.org>
- Marijn Kruisselbrink <mek@chromium.org>
- Olivier Yiptong <oyiptong@chromium.org>

# IndexedDB putAll

**Authors**

[nums@google.com](mailto:nums@google.com)

**Participate**

Original discussion here: [https://github.com/w3c/IndexedDB/issues/69](https://github.com/w3c/IndexedDB/issues/69). Please post issues/suggestions in this repository's issue tracker.

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
const store = db.createObjectStore("books", {keyPath: "author"});
store.put({title: "Quarry Memories", author: "Fred"});
store.put({title: "Bedrock Nights", author: "Barney"});
```


Here is an example of insertion using _out-of-line_ keys. Since no key path is defined on the object store, a key must be explicitly stated as the second parameter of put(). 


```javascript
const store = db.createObjectStore("books");
store.put({title: "Quarry Memories", author: "Fred"}, "key_1");
store.put({title: "Bedrock Nights", author: "Barney"}, "key_2");
```


Currently, if a developer wants to store large numbers of objects, they must loop put().


```javascript
for(let i = 0; i < 100; i++) {
  store.put({ field1: 'value1', field2: 'value2', field3: 'value3'});
}
```


## Benefits

“putAll” addresses repeated feedback from customers who have stated the desire to have the ability to bulk-insert large amounts of data into an object store.

## Examples

The putAll API endpoints address repeated feedback from customers who have stated the desire to have the ability to bulk-insert large amounts of data into an object store.


```javascript
let entries = new Map();
entries.set("key_1", {title: "Quarry Memories", author: "Fred"});
entries.set("key_2", {title: "Bedrock Nights", author: "Barney"});
store.putAllEntries(entries);
```


putAllValues(Iterable)


```javascript
let value1 = {title: "Quarry Memories", author: "Fred"};
let value2 = {title: "Bedrock Nights", author: "Barney"};
let values = [value1, value2]
store.putAllValues(values)
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

### Combining putAllEntries and putAllValues into 1 function

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

### Using JavaScript Objects instead of Map

I considered using JavaScript objects instead of maps but chose not to since the keys in JavaScript objects can only be strings, whilst the keys in indexedDB are not limited to strings.

## Future Considerations

addAll

deleteAll

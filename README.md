# record-store

A store for record generation and manipulation. Basically, it takes a set of
named jobs and handles the wiring so that we can get some neat party tricks:

- Job request merging and memoization ensures that we will only ever compute a
  particular job once per-record. For example: this ensures that we only read a
  file once, parse its contents once, or generate source maps once.
- If a file watcher, or similar, invalidates a record, any pending jobs are
  unrolled and passed back with a rejection value that can be introspected and
  handled. This ensures that we don't compute expensive jobs that are no longer
  relevant.


## Install

`npm install --save record-store`


## Example

```js
import fs from 'fs';
import promisify from 'promisify-node';
import createRecordStore from 'record-store';

const readFile = promisify(fs.readFile);

const store = createRecordStore({
  readText(ref, store) {
    return readFile(ref.name, 'utf8');
  },
  reverse(ref, store) {
    return store.readText(ref).then(text => {
      return text.split('').reverse().join('');
    });
  },
  splitWords(ref, store) {
    return store.readText(ref).then(text => {
      return text.split(' ');
    });
  }
});


// Assuming a file containing the text 'foo bar woz'
store.create('/some/file');

Promise.all([
  store.reverse('/some/file'),
  store.splitWords('/some/file')
]).then(([reversed, lines]) => {
  console.log(reversed); // 'zow rab oof'
  console.log(lines); // ['foo', 'bar', 'woz']
});
```


## Jobs

Jobs are functions that are specified via the object passed to the record
store during its creation. Jobs are passed two arguments, the first is a
reference to a record, the second is an interface to the store itself that
enables the job to call other jobs.

Record references are immutable objects that expose the following
properties:

- `name`: the string value used when the record was created
- `ext`: a convenience hook for treating files based on their extension

Store interfaces expose functions that correspond to all named jobs.

Note: When calling a job via the interface, you must provide the record's
reference object.


## Intercepts

If your build pipeline is half-way through processing some data when an
associated file suddenly changes, you would expect all pending jobs to respond
to the change and exit early.

While this seems simple enough, it ends up requiring your subsystems to
periodically check the state of other subsystems. This introduces more code,
more complex control flow and more tightly couples the various parts of your
system together.

To avoid these issues, some abstraction layers are placed between jobs such
that they can be intercepted and rejected either before they start or once
they have completed.

If a job was intercepted and rejected, you can handle the intercept further
along the promise chain. You can also interrogate the intercept via the
store's API, if you want to handle things more gracefully.


## Handling intercept rejections

```js
import createRecordStore from 'record-store';

const recordStore = createRecordStore({
  foo(ref, store) {
    // ...
  }
});

recordStore.create('/some/file');

recordStore.foo('/some/file')
  .catch(err => {
    if (recordStore.isIntercept(err)) {
      // The job was intercepted during processing

      if (recordStore.isRecordRemovedIntercept(err)) {
        // The associated record was removed
      }

      if (recordStore.isRecordInvalidIntercept(err)) {
        // The associated record was removed,
        // then re-created
      }
    }

    // It was an actual error, so we pass it back up
    // the chain
    return Promise.reject(err);
  });
```
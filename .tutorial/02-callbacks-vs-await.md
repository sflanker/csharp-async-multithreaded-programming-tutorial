# Callbacks vs. Await

## The Before Times

It's important to be aware of the way things were to appreciate the way they are now.

In olden days, asynchronous programming was a bleak landscape of callbacks, handles, and notifications. There were no async and await keywords, no Promises or Tasks. Makinging your code asynchronous meant making it ugly and unreadable.

Specifically, a typical mechanism to perform an IO operation asynchronously involved calling the target function, but instead of actually using some value it returned, you would pass it a _callback function_ which would be invoked later when the operation completed.

```javascript
myfs.read('filename', (data, err) => {
  // Any code that needs to run after data has been read from the file must be written here
  myfs.read('anotherFile', (data, err) => {
    // Effectively "sequential" IO operations are written as deeply nested callbacks
  });
  // And don't forget to handle error conditions
});

// code that is written here would run before data was actually read from the file
```

In Node.js this paradigm lead to unfortunate code structures consisting of callbacks functions declared within callback functions and so on. Libraries and data structures were created to deal with this problem but frequently these introduced there own problems and still represented a deviation from typical procedural program flow.

Other programming languages faired little better, with their own clunky alternatives, such as the `Begin...` and `End...` functions of the .Net Framework (see: [IAsyncResult](https://learn.microsoft.com/en-us/dotnet/api/system.iasyncresult?view=net-7.0)).

## The Long _Awaited_ Solution

First introduced in F# (although possibly influenced by paradigms from other functional programming languages), and eventually spreading to numerous other programming languages (C#, Python, TypeScript, JavaScript, and more). The `await` keyword brought an end to the callback hell that had long plagued asynchronous programming.

In essence `await` makes the callback implicit in procedural code. Any code that comes after an `await`, up to the end of the enclosing function, is encapsualted as an anonymous function that is only invoked when the `awaited` operation is completed. Any code that uses `await` is itself asynchronous. Languages with an `await` keyword typically have a type that defines what it means to be "awaitable" and any asynchronous function will return that type.

### Example

```javascript
// some initial code
const result = await asyncfs.read('filename');
// code that does something with the result
console.log(result);
```

Would be essentially equivalent to:

```javascript
// some initial code
myfs.read('filename', result => {
  // code that does something with the result
  console.log(result);
});
```

However, this contrived example doesn't do justice to the `await` keyword. It can be used in contexts were restructuring the code to use a callback would present a substantial difficulty:

```javascript
let result = '';
let errors = [];
for (const file of fileList) {
  if (file.exists) {
    try {
      result += file.name + ': ';
      result += await asyncfs.read(file.name);
      result += '\n';
    } catch (e) {
      errors.push(e);
    }
  } else {
    result += file.name + ' Not Found\n';
  }
}
```

By using the await keyword, the compiler is able to take care of all the local state that needs to be captured and used after each IO operation completes (i.e. the `fileList` iterator, the `results` string, and the `errors` array).
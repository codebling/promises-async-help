From https://stackoverflow.com/questions/60456059/mongoose-pass-data-out-of-withtransaction-helper/60480108#60480108


It looks like there is some confusion here as to how to correctly use Promises, on several levels. 

### Callback and Promise are being used incorrectly

If the function is supposed to accept a callback, don't return a Promise. If the function is supposed to return a Promise, use the callback given by the Promise:

```
const transactionSession = await mongoose.startSession()
await transactionSession.withTransaction( (tSession) => {
    return new Promise( (resolve, reject) => {
        //using Node-style callback
        doSomethingAsync( (err, testData) => {
            if(err) {
                reject(err);
            } else {
                resolve(testData); //this is the equivalent of cb(null, "Any test data")
            }
        });
    })
```

Let's look at this in more detail: 

`return new Promise( (resolve, reject) => {` This creates a new Promise, and the Promise is giving you two callbacks to use. `resolve` is a callback to indicate success. You pass it the object you'd like to return. Note that I've removed the `async` keyword (more on this later). 

For example:
```
const a = new Promise( (resolve, reject) => resolve(5) );
a.then( (result) => result == 5 ); //true
```

`(err, testData) => {` This function is used to map the Node-style `cb(err, result)` to the Promise's callbacks. 


### Try/catch are being used incorrectly. 

Try/catch can only be used for synchronous statements. Let's compare a synchronous call, a Node-style (i.e. `cb(err, result)`) asynchronous callback, a Promise, and using await:

* Synchronous:
```
try {
    let a = doSomethingSync();
} catch(err) {
    handle(err);
}
```
* Async:
```
doSomethingAsync( (err, result) => {
    if (err) {
        handle(err);
    } else {
        let a = result;
    }
});
```
* Promise:
```
doSomethingPromisified()
    .then( (result) => { 
        let a = result; 
    })
    .catch( (err) => {
        handle(err);
    });
```
* Await. Await can be used with any function that returns a Promise, and lets you handle the code as if it were synchronous:
```
try {
    let a = await doSomethingPromisified();
} catch(err) {
    handle(err);
}
```


---
## Additional Info

### `Promise.resolve()` 
**`Promise.resolve()`** creates a new Promise and resolves that Promise with an undefined value. This is shorthand for:
```
new Promise( (resolve, reject) => resolve(undefined) );
```
The callback equivalent of this would be:
```
cb(err, undefined);
```

### `async`
**`async`** goes with **`await`**. If you are using `await` in a function, that function must be declared to be `async`. 

Just as `await` unwraps a Promise (`resolve` into a value, and `reject` into an exception), `async` *wraps* code into a Promise. A `return value` statement gets translated into `Promise.resolve(value)`, and a thrown exception `throw e` gets translated into `Promise.reject(e)`. 

Consider the following code
```
async () => {
    return doSomethingSync();
}
```
The code above is equivalent to this:
```
() => {
    const p = new Promise(resolve, reject);
    try {
        const value = doSomethingSync();
        p.resolve(value);
    } catch(e) {
        p.reject(e);
    }
    return p;
}
```

If you call either of the above functions without `await`, you will get back a Promise. If you `await` either of them, you will be returned a value, or an exception will be thrown.

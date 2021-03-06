# asynquence

A lightweight (**~1.9k** minzipped) micro-lib for asynchronous flow-control using sequences and gates.

## Explanation

### TL;DR: By Example

* [Sequences & gates](https://gist.github.com/getify/5959149), at a glance
* Refactoring ["callback hell"-style to asynquence](https://gist.github.com/getify/8459026#file-gistfile3-js-L27-L39)
* Example/explanation of [promise-style sequences](https://gist.github.com/jakearchibald/0e652d95c07442f205ce#comment-977119)
* More advanced example of ["nested" composition of sequences](https://gist.github.com/getify/10273f3de07dda27ebce)
* [Iterable sequences](#iterable-sequences): [sync loop](https://gist.github.com/getify/8211148#file-ex1-sync-iteration-js) and [async loop](https://gist.github.com/getify/8211148#file-ex2-async-iteration-js) and [async batch iteration of list](https://gist.github.com/getify/8464917)
* API [Usage Examples](#usage-examples)

### Sequences
Say you want to perform two or more asynchronous tasks one after the other (like animation delays, XHR calls, file I/O, etc). You need to set up an ordered series of tasks and make sure the previous one finishes before the next one is processed. You need a **sequence**.

You create a sequence by calling `ASQ(...)`. **Each time you call `ASQ()`, you create a new, separate sequence.**

To create a new step, simply call `then(...)` with a function. That function will be executed when that step is ready to be processed, and it will be passed as its first parameter the completion trigger. Subsequent parameters, if any, will be any messages passsed on from the immediately previous step.

The completion trigger that your step function(s) receive can be called directly to indicate success, or you can add the `fail` flag (see examples below) to indicate failure of that step. In either case, you can pass one or more messages onto the next step (or the next failure handler) by simply adding them as parameters to the call.

If you register a step using `then(...)` on a sequence which is already currently complete, that step will be processed at the next opportunity. Otherwise, calls to `then(...)` will be queued up until that step is ready for processing.

You can register multiple steps, and multiple failure handlers. However, messages from a previous step (success or failure completion) will only be passed to the immediately next registered step (or the next failure handler). If you want to propagate along a message through multiple steps, you must do so yourself by making sure you re-pass the received message at each step completion.

To listen for any step failing, call `or(...)` on your sequence to register a failure callback. You can call `or()` as many times as you would like. If you call `or()` on a sequence that has already been flagged as failed, the callback you specify will just be executed at the next opportunity.

### Gates
If you have two or more tasks to perform at the same time, but want to wait for them all to complete before moving on, you need a **gate**.

Calling `gate(..)` with two or more functions creates a step that is a parallel gate across those functions, such that the single step in question isn't complete until all segments of the parallel gate are **successfully** complete.

For parallel gate steps, each segment of that gate will receive a copy of the message(s) passed from the previous step. Also, all messages from the segments of this gate will be passed along to the next step (or the next failure handler, in the case of a gate segment indicating a failure).

### Conveniences
There are a few convenience methods on the API, as well:

* `pipe(..)` takes one or more completion triggers from other sequences, treating each one as a separate step in the sequence in question. These completion triggers will, in turn, be piped both the success and failure streams from the sequence.

    `Sq.pipe(done)` is sugar short-hand for `Sq.then(done).or(done.fail)`.

* `seq(..)` takes one or more functions, treating each one as a separate step in the sequence in question. These functions are expected to return new sequences, from which, in turn, both the success and failure streams will be piped back to the sequence in question.

    `seq(Fn)` is sugar short-hand for `then(function(done){ Fn.apply(null,[].slice.call(arguments,1)).pipe(done); })`.

    This method will also accept *asynquence* sequence instances directly. `seq(Sq)` is sugar short-hand for `then(function(done){ Sq.pipe(done); })`.

    Additionally, this method can accept, either directly or through function-call, an [Iterable Sequence](#iterable-sequences). `seq(iSq)` is (sort-of) sugar short-hand for `then(function(done){ iSq.then(done).or(done.fail); })`.

* `val(..)` takes one or more functions, treating each one as a separate step in the sequence in question. These functions can each optionally return a value, each value of which will, in turn, be passed as the completion value for that sequence step.

    `val(Fn)` is sugar short-hand for `then(function(done){ done(Fn.apply(null,[].slice.call(arguments,1))); })`.

    This method will also accept non-function values as sequence value-messages. `val(Va)` is sugar short-hand for `then(function(done){ done(Va); })`.

* `promise(..)` takes one or more [standard Promises/A+ compliant](http://promisesaplus.com/) promises, and subsumes them into the sequence. See [Promises/A+ Compliance](#promisesa-compliance) below for more information.

    `promise(Pr)` is sugar short-hand for `then(function(done){ Pr.then(done,done.fail); })`.

    This method will also accept function(s) which return promises. `promise(Fn)` is sugar short-hand for `then(function(done){ Fn.apply(null,[].slice.call(arguments,1)).then(done,done.fail); })`.

* `fork()` creates a new sequence that forks off of the main sequence. Success or Error message(s) stream along to the forked sequence as expected, but the main sequence continues as its own sequence beyond the fork point, and neither sequence will have any further effect on the other.

    This API method is primarily useful to create multiple "listeners" at the same point of a sequence. For example: `sq = ASQ()...; sq2 = sq.fork().then(..); sq3 = sq.fork().then(..); sq.then(..)`. In that snippet, there'd be 3 `then(..)` listeners that would be equally and simultaneously triggered when the main `sq` sequence reached that point.

    **Note:** Unlike most other API methods, `fork()` returns a new sequence instance, so chaining after `fork()` would not be chaining off of the main sequence but off of the forked sequence.

    `sq.fork()` is (sort-of) sugar short-hand for `ASQ().seq(sq)`.

* `duplicate()` creates a separate copy of the current sequence (as it is at that moment). The duplicated sequence is "paused", meaning it won't automatically run, even if the original sequence is already running.

    To unpause the paused sequence-copy, call `unpause()` on it. The other option is to call the helper `ASQ.unpause(..)` and pass in a sequence. If the sequence is paused, it will be unpaused (and if not, just passes through safely).

    **Note:** Technically, `unpause()` schedules the sequence to be unpaused as the next "tick", so it doesn't really unpause *immediately* (synchronously). This is consistent with all other calls to the API (`ASQ()`, `then()`, `gate()`, etc), which all schedule procession of the sequence on the next "tick".

    The instance form of `unpause(..)` (not `ASQ.unpause(..)`) will accept any arguments sent to it and pass them along as messages to the first step of the sequence, each time it's invoked. This allows you to setup different templated (duplicated) sequences with distinct initial message states, if necessary.

    `unpause()` is only present on a sequence API in this initial paused state after it was duplicated from another sequence. It is removed as soon as that next "tick" actually unpauses the sequence. It is safe to call multiple times until that next "tick", though that's not recommended. The `ASQ.unpause(..)` helper is always present, and it first checks for an `unpause()` on the specified sequence instance before calling it, so that's safer.

* `errfcb` is a flag on the triggers that are passed into `then(..)` steps and `gate(..)` segments. If you're using methods which expect an "error-first" style (aka, "node-style") callback, `{trigger}.errfcb` provides a properly formatted callback for the occasion.

    If the "error-first" callback is then invoked with the first ("error") parameter set, the main sequence is flagged for error as usual. Otherwise, the main sequence proceeds as success. Messages sent to the callback are passed through to the main sequence as success/error as expected.

    `ASQ(function(done){ somethingAsync(done.errfcb); })` is sugar short-hand for `ASQ(function(done){ somethingAsync(function(err){ if (err) done.fail(err); else done.apply(null,[].slice.call(arguments,1))}); })`.

You can also `abort()` a sequence at any time, which will prevent any further actions from occurring on that sequence (all callbacks will be ignored). The call to `abort()` can happen on the sequence API itself, or using the `abort` flag on a completion trigger in any step (see example below).

`ASQ.messages(..)` wraps a set of values as a ASQ-branded array, making it easier to pass multiple messages at once, and also to make it easier to distinguish a normal array (a value) from a value-messages container array, using `ASQ.isMessageWrapper(..)`.

If you want to test if any arbitrary object is an *asynquence* sequence instance, use `ASQ.isSequence(..)`.

`ASQ.iterable(..)` is added by the `iterable-sequence` contrib plugin. See [Iterable Sequences](#iterable-sequences) below for more information.

`ASQ.unpause(..)` is a helper for dealing with "paused" (aka, *just* duplicated) sequences (see `duplicate()` above).

`ASQ.noConflict()` rolls back the global `ASQ` identifier and returns the current API instance to you. This can be used to keep your global namespace clean, or it can be used to have multiple simultaneous libraries (including separate versions/copies of *asynquence*!) in the same program without conflicts over the `ASQ` global identifier.

### Plugin Extensions
`ASQ.extend( {name}, {build} )` allows you to specify an API extension, giving it a `name` and a `build` function callback that should return the implementation of your API extension. The `build` callback is provided two parameters, the sequence `api` instance, and an `internals(..)` method, which lets you get or set values of various internal properties (generally, don't use this if you can avoid it).

Example:

```js
// "foobar" plugin, which injects message "foobar!"
// into the sequence stream
ASQ.extend("foobar",function __build__(api,internals){
    return function __foobar__() {
        api.val(function __val__(){
            return "foobar!";
        });

        return api;
    };
});

ASQ()
.foobar() // our custom plugin!
.val(function(msg){
    console.log(msg); // foobar!
});
```

See the `/contrib/*` plugins for more complex examples of how to extend the *asynquence* API.

The `/contrib/*` plugins provide a variety of [optional contrib plugins](https://github.com/getify/asynquence/blob/master/contrib/README.md) as helpers for async flow-controls.

For browser usage, simply include the `asq.js` library file and then the `contrib.js` file. For node.js, these contrib plugins are available as a separate npm module `asynquence-contrib`.

#### Iterable Sequences
One of the contrib plugins provided is `iterable-sequence`. Unlike other plugins, which add methods onto the sequence instance API, this plugin adds a new method directly onto the main module API: `ASQ.iterable(..)`. Calling `ASQ.iterable(..)` creates a special iterable sequence, as compared to calling `ASQ(..)` to create a normal *asynquence* sequence.

An iterable sequence works similarly to normal *asynquence* sequences, but a bit different. `then(..)` still registers steps on the sequence, but it's basically just an alias of `val(..)`, because the most important difference is that steps of an iterable sequence **are not passed completion triggers**.

Instead, an iterable sequence instance API has a `next(..)` method on it, which will allow the sequence to be externally iterated, one step at a time. Whatever is passed to `next(..)` is sent as step messages to the current step in the sequence. `next(..)` always returns an object like:

```js
{
    value: ...          // return messages
    done: true|false    // sequence iteration complete?
}
```

`value` is any return message(s) from the `next(..)` invocation (`undefined` otherwise). `done` is `true` if the previously iterated step was (so far) the last registered step in the iterable sequence, or `false` if there's still more sequence steps queued up.

Just like with normal *asynquence* sequences, register one or more error listeners on the iterable sequence by calling `or(..)`. If a step results in some error (either accidentally or manually via `throw ..`), the iterable sequence is flagged in the error state, and any error messages are passed to the registered `or(..)` listeners.

Also, just like `next(..)` externally controls the normal iteration flow of the sequence, `throw(..)` externally "throws" an error into the iterable sequence, triggering the `or(..)` flow as above. Iterable sequences can be `abort()`d just as normal *asynquence* sequences.

Iterable sequences are a special subset of sequences, and as such, some of the normal *asynquence* API variations do not exist, such as `gate(..)`, `seq(..)`, and `promise(..)`.

```js
function step(num) {
    return "Step " + num;
}

var sq = ASQ.iterable()
    .then(step)
    .then(step)
    .then(step);

for (var i=0, ret;
    !(ret && ret.done) && (ret = sq.next(i+1));
    i++
) {
    console.log(ret.value);
}
// Step 1
// Step 2
// Step 3
```

This example shows sync iteration with a `for` loop, but of course, `next(..)` can be called in various [async fashions to iterate](https://gist.github.com/getify/8211148#file-ex2-async-iteration-js) the sequence over time.

Just like regular sequences, iterable sequences have a `duplicate()` method (see ASQ's instance API above) which makes a copy of the sequence *at that moment*. However, iterable sequences are already "paused" at each step anyway, so unlike regular sequences, there's no `unpause()` (nor is there any reason to use the `ASQ.unpause(..)` helper!), because it's unnecessary. You just call `next()` on an iterable sequence (even if it's a copy of another) when you want to advance it one step.

### Multiple parameters
API methods take one or more functions as their parameters. `gate(..)` treats multiple functions as segments in the same gate. The other API methods (`then(..)`, `or(..)`, `pipe(..)`, `seq(..)`, and `val(..)`) treat multiple parameters as just separate subsequent steps in the respective sequence. These methods don't accept arrays of functions (that you might build up programatically), but since they take multiple parameters, you can use `.apply(..)` to spread those out.

### Promises/A+ Compliance
**The goal of *asynquence* is that you should be able to use it as your primary async flow-control library, without the need for other Promises implementations.**

This lib is intentionally designed to hide/abstract the idea of Promises, such that you can do quick and easy async flow-control programming without creating Promises directly.

As such, the *asynquence* API itself is *not [Promises/A+](http://promisesaplus.com/) compliant*, nor *should* it be, because the "promises" used are hidden underneath *asynquence*'s API. **Note:** the implementation promises behave predictably like standard Promises.

If you are also using other Promises implementations alongside *asynquence*, you *can* quite easily receive and consume a regular Promise value from some other method into the signal/control flow for an *asynquence* sequence.

For example, if using both the [Q promises library](https://github.com/kriskowal/q) and *asynquence*:

```js
// Using *Q*, make a standard Promise out
// of jQuery's Ajax "promise"
var p = Q( $.ajax(..) );

// Now, asynquence flow-control including a
// standard Promise
ASQ()
.then(function(done){
    setTimeout(done,100);
})
// subsume a standard Promise into the sequence
.promise(p)
.val(function(ajaxResp){
    console.log(ajaxResp);
});
```

**Despite API similarities** (like the presence of `then(..)` on the API), an *asynquence* instance is **not** designed to be used *as a Promise value* linked/passed to another standard Promise.

Trying to do so will likely cause unexpected behavior, because Promises/A+ insists on problematic (read: "dangerous") duck-typing for objects that have a `then()` method, as *asynquence* instances do.

## Browser, node.js (CommonJS), AMD: ready!

The *asynquence* library is packaged with a light variation of the [UMD (universal module definition)](https://github.com/umdjs/umd) pattern, which means the same file is suitable for inclusion either as a normal browser .js file, as a node.js module, or as an AMD module. Can't get any simpler than that, can it?

For browser usage, simply include the `asq.js` library file. For node.js usage, install the `asynquence` package via npm, then `require(..)` the module:

```js
var ASQ = require("asynquence");
```

**Note:** The `ASQ.noConflict()` method really only makes sense when used in a normal browser global namespace environment. It **should not** be used when the node.js or AMD style modules are your method of inclusion.

## Usage Examples

Using the following example setup:

```js
function fn1(done) {
    alert("Step 1");
    setTimeout(done,1000);
}

function fn2(done) {
    alert("Step 2");
    setTimeout(done,1000);
}

function yay() {
    alert("Done!");
}
```

Execute `fn1`, then `fn2`, then finally `yay`:

```js
ASQ(fn1)
.then(fn2)
.then(yay);
```

Pass messages from step to step:

```js
ASQ(function(done){
    setTimeout(function(){
        done("hello");
    },1000);
})
.then(function(done,msg1){
    setTimeout(function(){
        done(msg1,"world");
    },1000);
})
.then(function(_,msg1,msg2){ // basically ignoring this step's completion trigger (`_`)
    alert("Greeting: " + msg1 + " " + msg2);
    // 'Greeting: hello world'
});
```

Handle step failure:

```js
ASQ(function(done){
    setTimeout(function(){
        done("hello");
    },1000);
})
.then(function(done,msg1){
    setTimeout(function(){
        // note the `fail` flag here!!
        done.fail(msg1,"world");
    },1000);
})
.then(function(){
    // sequence fails, won't ever get called
})
.or(function(msg1,msg2){
    alert("Failure: " + msg1 + " " + msg2);
    // 'Failure: hello world'
});
```

Create a step that's a parallel gate:

```js
ASQ()
// normal async step
.then(function(done){
    setTimeout(function(){
        done("hello");
    },1000);
})
// parallel gate step (segments run in parallel)
.gate(
    function(done,greeting){ // gate segment
        setTimeout(function(){
            // 2 gate messages!
            done(greeting,"world");
        },500);
    },
    function(done,greeting){ // gate segment
        setTimeout(function(){
            // only 1 gate message!
            done(greeting + " mikey");
        },100);
        // this segment finishes first, but message
        // still kept "in order"
    }
)
.then(function(_,msg1,msg2){
    // msg1 is an array of the 2 gate messages
    // from the first segment
    // msg2 is the single message (not an array)
    // from the second segment

    alert("Greeting: " + msg1[0] + " " + msg1[1]);
    // 'Greeting: hello world'
    alert("Greeting: " + msg2);
    // 'Greeting: hello mikey'
});
```

Use `pipe(..)`, `seq(..)`, and `val(..)` helpers:

```js
var seq = ASQ()
.then(function(done){
    ASQ()
    .then(function(done){
        setTimeout(function(){
            done("Hello World");
        },100);
    })
    .pipe(done); // pipe sequence output to `done` completion trigger
})
.val(function(msg){ // NOTE: no completion trigger passed in!
    return msg.toUpperCase(); // map return value as step output
})
.seq(function(msg){ // NOTE: no completion trigger passed in!
    var seq = ASQ();

    seq
    .then(function(done){
        setTimeout(function(){
            done(msg.split(" ")[0]);
        },100);
    });

    return seq; // pipe this sub-sequence back into the main sequence
})
.then(function(_,msg){
    alert(msg); // "HELLO"
});
```

Abort a sequence in progress:

```js
var seq = ASQ()
.then(fn1)
.then(fn2)
.then(yay);

setTimeout(function(){
    // will stop the sequence before running
    // steps `fn2` and `yay`
    seq.abort();
},100);

// same as above
ASQ()
.then(fn1)
.then(function(done){
    setTimeout(function(){
        // `abort` flag will stop the sequence
        // before running steps `fn2` and `yay`
        done.abort();
    },100);
})
.then(fn2)
.then(yay);
```

## Builds

The core library file can be built (minified) with an included utility:

```
./build-core.js
```

However, the recommended way to invoke this utility is via npm:

```
npm run-script build-core
```

## License

The code and all the documentation are released under the MIT license.

http://getify.mit-license.org/

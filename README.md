## Bloodhound
### Tracked Promises in JavaScript

## Why?

Promises are fantastic. They encapsulate a long-running operation into a simple object that
invokes callbacks based on the operation's success or failure.

But more than that, promises can be chained together into complex trees, where the root's
success or failure will depend on the success or failure of all its children.

This tree-like structure accurately represents much of the real-world transactional logic of
modern web applications.

Unfortunately, because these operations can overlap, current performance tracking libraries
have no way to distinguish each operation -- they simply assume all transactions on a page
shoudl be groupd together. Or in a single-page application, they assume that everything in
the current view should be timed together.

But modern single-page applications can have micro-content loading alongside primary content.
The user can be in any part of an application -- like a slideout drawer showing user messages
while the main area shows an edit-record view. Current tracking libraries have no way to
handle this situation.

Enter Bloodhound.

## How?

Bloodhound promises work just like regular promises (in fact, it fully implements the
(https://promisesaplus.com/)[A+ spec]), with a lot of the syntactic sugar that other promise
implementations have popularized.

But unlike other promise libraries, Bloodhound also adds the following instance methods:

`Promise#trackAs(name [, passive]) : Promise`
Marks the promise instance as a tracked promise, and then returns that instance for chaining.
Optionally, you can say the promise is passively tracked. What that means will be explained
below.

`Promise#done() : Promise`
Not all promise libraries have a done method, but it's vital to using promises correctly.
Basically, the golden rule of promises is:

    If you don't return the promise, you must call done.

Calling done is what tells Bloodhound that it should attempt to gather up any timing data
in the tree and persist it to any registered collectors. It also throws any unhandled
rejections / exceptions so you know there's an error in your application. (Otherwise, your
app could end up in an inconsistent state.)

Let's look at an example that combines the two new methods:

    function loadMessages() {
        return new Promise(function(resolve, reject) {
            // do real stuff here and
            // call resolve(...) when done
        }).trackAs('load messages', true);
    }
    
    function loadAppData() {
        return new Promise(function(resolve, reject) {
            // do real stuff here and
            // call resolve(...) when done
        }).trackAs('load app data', true);
    }
    
    Promise.all([
        loadMessages(),
        loadAppData()
    ]).trackAs('loading').done();

What does the code do? Both `loadMessages` and `loadAppData` return a promise that would
be resolved once their data calls completed. But before the promise is returned, it is
passively tracked with an appropriate name. Because the promise is being returned, we do
not call `done()` -- someone else will be consuming our promise, so we can't throw any
unhandled errors just yet.

    **What does it mean to be passively tracked?**
    Basically, a passively tracked promise is just a promise that has been given the
    specified name but will not be persisted to any registered collectors when `done`
    is called...*unless* it is part of a larger tree of promises that contains at
    least one actively tracked promise and that tree has completely settled.
    
    **Why use passive tracking at all?**
    Sometimes you want to control how your promise will appear in timing data but don't
    actually want it persisted. For example, let's say you're routing all remote HTTP
    calls through a custom data layer. Your data layer could return a promise that is
    passively tracked using the remote URL as the name. That way, any reports you run
    on the generated timing data show clearly what endpoint was taking the longest time
    to run. But that timing data would not be logged EVERY time a remote call was made,
    just when a call was made as part of a larger, actively tracked promise tree.

Finally, we wrap both promises in a call to `Promise.all`, a static method that returns
a promise which is only resolved if all child promises resolve. But whether that promise
resolves or reject, we actively track it as 'loading'. You can tell it's actively tracked
because we didn't specify `true` for the optional `passive` parameter of `trackAs`. Then
we call `done()`, which waits until the promise is either resolved or rejected to check
for unhandled exceptions and also persist the timing data to any registered collectors.

In this case, the timing data might look like the following:

    {
      "name": "loading",
      "data": [
        [
          "sample",
          "messages"
        ],
        [
          "sample",
          "app",
          "data"
        ]
      ],
      "start": 1425943275662,
      "stop": 1425943275786,
      "duration": 124,
      "children": [
        {
          "name": "load messages",
          "data": [
            "sample",
            "messages"
          ],
          "start": 1425943275661,
          "stop": 1425943275716,
          "duration": 55,
          "children": []
        },
        {
          "name": "load app data",
          "data": [
            "sample",
            "app",
            "data"
          ],
          "start": 1425943275662,
          "stop": 1425943275776,
          "duration": 114,
          "children": []
        },
        {
          "name": "anonymous",
          "start": 1425943275663,
          "children": []
        }
      ]
    }

This is already excellent data -- we can quickly see that our application
is taking a long time loading application data. Before, we would simply see
that the initial load took 124 seconds; but figuring out why would be a lot
more difficult.

If you look closely, you'll notice that the timing data for 'load messages'
and 'load app data' both start *before* the parent and end *after*. Why?
Because that's when they started. We kicked off those calls *and then*
passed the promises to `Promise.all`.

Some people don't like seeing data like this; they prefer more consistent
timing data, where parent promises always start on or before their children
and always end on or after the last child settles. If this is needed for
your particular situation, not to worry. Simply configure Bloodhound using
the following command:

`Promise.config.timing.useSaneTimings();`

This adjusts timing data so parents start on or before their children and
end on or after their last child settles. With sane timings enabled, the
above timing data might look like the following:

    {
      "name": "loading",
      "data": [
        [
          "sample",
          "messages"
        ],
        [
          "sample",
          "app",
          "data"
        ]
      ],
      "start": 1425943993069,
      "stop": 1425943993193,
      "duration": 124,
      "children": [
        {
          "name": "load messages",
          "data": [
            "sample",
            "messages"
          ],
          "start": 1425943993069,
          "stop": 1425943993129,
          "duration": 60,
          "children": []
        },
        {
          "name": "load app data",
          "data": [
            "sample",
            "app",
            "data"
          ],
          "start": 1425943993069,
          "stop": 1425943993183,
          "duration": 114,
          "children": []
        }
      ]
    }

## Configuration

Bloodhound provides a number of ways to configure timings, collectors, and how
asynchronous operations are invoked.

### Scheduling Asynchronous Operations

`Promise.config.setScheduler(mySchedulerFunction)`
Use the specified function to invoke an asynchronous operation. By default, Bloodhound
uses `window.setTimeout` to execute an operation on the next tick of the clock. In
node environments, you may wish to set the scheduler to `async`. In Angular environments
you may wish to set the scheduler to `$rootScope.$digest.bind($rootScope)`. If you're
looking for the fastest possible async execution in modern browsers, you could set the
scheduler to use a predefined MutationObserver callback.

### Timing Configuration

`Promise.config.timing.enable()`
Enables the persistence of timing data to any registered collectors. This is the default
state of Bloodhound.

`Promise.config.timing.disable()`
Disables the persistence of timing data to registered collectors. You can re-enable
persistence by calling `Promise.config.timing.enable()`.

`Promise.config.timing.useSaneTimings()`
Ensures parent promises are persisted as starting on or before their child promises and
ending on or after their last child promise settles.

### Timing Collectors

`Promise.config.collectors.add(collector) : Function`
Adds the specified collector to the list of registered collectors, and returns a function
you can invoke to remove the collector again.

    var collector = {
        collect: function(timingData) {
            console.log(JSON.stringify(timingData, null, 2));
        }
    };
    
    var remove = Promise.config.collectors.add(collector);

A collector is simply an object with a method called `collect` that will accept a single
timingData instance. Each timing data instance will have the following properties:

    name {String} the tracked name of the promise, or 'anonymouse'
    data {*} either the resolved value or rejection reason
    start {Number} when the promise was created, as the number of milliseconds since
        midnight, January 1, 1970 UTC
    stop {Number} when the promise was finally settled, as the number of milliseconds
        since midnight, January 1, 1970 UTC
    duration {Number} the difference between start and stop
    children {Array} an array of child timings
    
`Promise.config.collectors.remove(collector)`
Removes the specified collector from the list of registered collectors.

## Full API

### Promise constructor

You create a new instance of a Bloodhound promise by calling the constructor and
passing your 'resolver' function -- the method that will be run asynchronously
and either resolve or reject the promise:

    new Promise(function(resolve, reject, notify) {
        // this method will be invoked asynchronously;
        // when it completes, call resolve or reject;
        // if it throws an exception, reject will be
        // called automatically; if you want to notify
        // any listeners of progress updates, call
        // notify with any data you want your listeners
        // to receive (such as a progress percentage)
    });
    
### Static Methods

#### all
#### any
#### some
#### race
#### settle

#### call
#### apply

#### cast
#### defer
#### delay

#### isPromise

### Instance Methods

#### then
#### tap
#### catch
#### notified
#### finally
#### done

#### spread
#### trackAs

#### isRejected
#### isResolved
#### isSettled
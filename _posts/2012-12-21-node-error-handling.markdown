--- 
layout: post
title: Node.js error handling
---

We use node.js at [Bugsnag](https://bugsnag.com) and also support error tracking for [node](https://bugsnag.com/docs/notifiers/node). While building out these projects I've had a bunch of experience with the state of error handling and thought I'd pass on what I've found. I'm going to discuss the various ways I've seen error handling done, and talk a little about what the future holds.

## Exceptions

Exceptions are supported in javascript, and you may well be tempted to use them to communicate an error, but due to asyncronous callbacks, it can be a bad idea. Throwing them from within asyncronous code means that a surrounding catch block isn't actually in the stack at that point. Consider the following example,

{% highlight javascript %}function async(callback) {
    process.nextTick(function(){
        throw new Error("Something went wrong");
        callback();
    });
}

try {
    async(function(){
        console.log("It worked!");
    });
} catch(error) {
    console.log("This is never printed.");
}{% endhighlight %}

You may think that the error would be caught by the catch block, but due to the asynchronous nature of the async function, it isnt actually in the stack at the time the exception is thrown. This will generate an uncaught exception in your app and by default it will crash. It does emit an [uncaughtException](http://nodejs.org/api/process.html#process_event_uncaughtexception) event and you can use services like [Bugsnag](http://bugsnag.com) to notify you when this happens, but your app is still crashing which should obviously be avoided.

## Event Emitters

You can use [EventEmitters](http://nodejs.org/api/events.html#events_class_events_eventemitter) in node to "emit" errors. For example,

{% highlight javascript %}function async() {
    var emitter = new (require('events').EventEmitter)();
    process.nextTick(function(){
        // This emits the "error" event
        emitter.emit("error", new Error("Something went wrong"));
    });
    return emitter;
}
var event = async();
event.on("error", function(error) {
    // When the "error" event is emitted, this is called
    console.error(error);
});{% endhighlight %}

This allows you to split the success and error functionality so there is a clean split between the two. You can even call the same callback from multiple event emitters so you can have common error handlers. These event emitters come with native support for 'error' events. From the docs,

<em>When an EventEmitter instance experiences an error, the typical action is to emit an 'error' event. Error events are treated as a special case in node. If there is no listener for it, then the default action is to print a stack trace and exit the program.</em>

So ensure that any event emitters that emit the error event are listened to, otherwise your app will crash.

## Error Objects

The most common form of error handling will make C coders feel at home. Like a return value in C, you pass an Error object as the first argument of an asynchronous callback, like so,

{% highlight javascript %}function async(callback) {
    callback(new Error("Something went wrong"));
}

// Call the async function    
async(function(error){
    if(error) {
        console.error(error);
        return;
    }
    
    //Do the callback work here
    console.log("It worked!");
});{% endhighlight %}

This way you can trivially handle errors in any callbacks without having to nest try catch statements. However you still have to remember to check the error in multiple places and you may have very similar error handling code litered throughout the application logic.

## Managing Errors

I've seen all three methods being used to inform a caller about an error. This gets especially bad when all three methods are in use within the same few lines of code in your app. There are some [techniques](http://dc-syntropy.blogspot.com/2012/03/error-handling-in-nodejs.html) for trying to manage this, but unfortunately we have to manually manage it. My preference, is to always use error objects that get passed to a callback as the error code path is clearer and more maintainable, but when using third party libraries and even core node code, you get what you are given.

This makes error handling in node often feel clunky, as you have to have multiple error handlers for different types of errors. When a single standard interface to detailing errors is introduced, things will get alot nicer.

## Future

Luckily, in node 0.8, there has been some work on trying to rectify this with the introduction of [domains](http://nodejs.org/api/domain.html). These are very cool so I recommend checking them out and seeing what you think. 

From a high level they allow you to deal with exceptions and errors in a single place, similar to how try catch works in synchronous code. It even allows you to intercept errors passed as the first argument to a callback as well as event emitters that emit "error" events. For example,

{% highlight javascript %}var mainDomain = domain.create();
mainDomain.run(function() {
    fs.readFile(filename, 'utf8', d.intercept(function(data) {
        // note, the first argument is never passed to the
        // callback since it is assumed to be the 'Error' argument
        // and thus intercepted by the domain.

        // if this throws, it will also be passed to the domain
        // so the error-handling logic can be moved to the 'error'
        // event on the domain instead of being repeated throughout
        // the program.
        return cb(null, JSON.parse(data));
      }));
});
mainDomain.on("error", function(error){
    console.error(error);
}).{% endhighlight %}

This goes some way to trying to create a standard interface for error handling in node and I like the principles behind it. There is still a little too much boilerplate code to be a perfect solution, but I'm hopeful they will iterate towards a nicer interface.

The maintainers of node are very clear that this is a work in progress, but they are soliciting feedback, so I'd encourage everyone to check them out and let them know what you think.

<em>This feature is new in Node version 0.8. It is a first pass, and is expected to change significantly in future versions. Please use it and provide feedback.</em>
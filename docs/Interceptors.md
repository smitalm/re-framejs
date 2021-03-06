## re-frame Interceptors

This tutorial explains re-frame Interceptors. By the end, you'll much
better understand the mechanics of event handling. 

As you read this, refer back to the 3rd panel of the 
[Infographic](EventHandlingInfographic.md).


<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
## Table Of Contents

- [Why Interceptors?](#why-interceptors)
- [What Do Interceptors Do?](#what-do-interceptors-do)
- [Wait, I know That Pattern!](#wait-i-know-that-pattern)
- [What's In The Pipeline?](#whats-in-the-pipeline)
- [Show Me](#show-me)
- [Handlers Are Interceptors Too](#handlers-are-interceptors-too)
- [Executing A Chain](#executing-a-chain)
  - [The Links Of The Chain](#the-links-of-the-chain)
  - [What Is Context?](#what-is-context)
  - [Self Modifying](#self-modifying)
  - [Credit](#credit)
  - [Write An Interceptor](#write-an-interceptor)
  - [Wrapping Handlers](#wrapping-handlers)
- [Summary](#summary)
- [Appendix](#appendix)
  - [The Built-in Interceptors](#the-built-in-interceptors)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->


## Why Interceptors?

Two reasons.

__First__, we want simple event handlers.  

Interceptors can look after "cross-cutting" concerns like undo, tracing and validation. 
They help us to factor out commonality, hide complexity and introduce further steps into the "Derived Data, 
Flowing" story promoted by re-frame. 

So, you'll want to use Interceptors because they solve problems, and help you to write nice code.  

__Second__, under the covers, Interceptors provide the mechanism by which 
event handlers are executed (when you `dispatch`). They are a central concept.

## What Do Interceptors Do?

They wrap. 

Specifically, they wrap event handlers. 

Imagine your event handler is like a piece of ham. An interceptor would be
like bread on either side of your ham, which makes a sandwich.

And two Interceptors, in a chain, would be like you put another 
pair of bread slices around the outside of the existing sandwich to make 
a sandwich of the sandwich. Now it is a very thick sandwich. 

Interceptors wrap on both sides of a handler, layer after layer.

## Wait, I know That Pattern!

Interceptors implement middleware. But differently.

Traditional middleware - often seen in web servers - creates a data 
processing pipeline via the nested composition of higher order functions. 
The result is a "stack" of functions. Data flows through this pipeline, 
first forwards from one end to the other, and then backwards.

Interceptors achieve the same outcome by assembling functions, as data, 
in a collection (a chain, rather than a stack). Data can then be iteratively
pipelined, first forwards through the functions in the chain, 
and then backwards along the same chain. 

Because the interceptor pipeline is composed via data, rather than 
higher order functions, it is a more flexible arrangement.  

## What's In The Pipeline?

Data. It flows through the pipeline being progressively transformed. 

Fine. But what data?

With a web server, the middleware "stack" progressively 
transforms a `request` in one direction, and, then in the backwards 
sweep, it progressively produces a `response`. 

In re-frame, the forwards sweep progressively creates the `coeffects` 
(inputs to the handler), while the backwards sweep processes the `effects`
(outputs from the handler).  

I'll pause while you read that sentence again. That's the key 
concept, right there. 

## Show Me

At the time when you register an event handler, you can provide a chain of interceptors too. 
 
Using a 3-arity registration function:
```javascript
regEventDb(
    'some-id',
    [in1, in2],                     // <--- a chain of 2 interceptors
    (db, v) => {                    // <-- the handler here, as before
        
    }
)
```

> Each Event Handler can have its own tailored interceptor chain, provided at registration-time. 

## Handlers Are Interceptors Too

You might see that registration above as associating `some-id` with 
two things: (1) a chain of 2 interceptors `[in1 in2]`
and (2) a handler.  
 
Except, the handler is turned into an interceptor too (we'll see how shortly).
 
So `some-id` is only associated with one thing: a 3-chain of interceptors, 
with the handler wrapped in an interceptor, called say `h`, and put on the 
end of the other two: `[in1, in2, h]`.  
 
Except, the registration function itself, `regEventDb`, actually takes this 3-chain 
and inserts its own standard interceptors, called say `std1` and `std2`
(which do useful things, more soon) at the front,
so **ACTUALLY**, there's about 5 interceptors in the chain: `[std1, std2, in1, in2, h]`

So, ultimately, that event registration associates the event id `some-id` 
with __just__ a chain of interceptors. Nothing more.
 
Later, when a `dispatch(['some-id', ...])` happens, that 5-chain of 
interceptors will be "executed".  And that's how an event gets handled. 


## Executing A Chain

### The Links Of The Chain

Each interceptor has this form:
```javascript
{
    id: 'something',                // decorative only
    before(ctx) {                   // returns possibly modified context
        ...
    },
    after(ctx) {                    // `identity` would be a noop
        
    }
} 
```

That's essentially a map of two functions. Now imagine a vector of these maps - that's an interceptor chain.  

Above we imagined an interceptor chain of `[std1, std2, in1, in2, h]`. Now we know that this is really
a vector of 5 maps: `[{...}, {...}, {...}, {...}, {...}]`  where each of the 5 maps have 
a `before` and `after` fn.

Sometimes, the `before` and `after` fns are noops (think `identity`). 

To "execute" an interceptor chain:
  1. create a `context` (a map, described below)
  2. iterate forwards over the chain, calling the `before` function on each interceptor
  3. iterate over the chain in the opposite direction calling the `after` function on each interceptor

Remember that the last interceptor in the chain is the handler itself (wrapped up to be the `before`).

That's it. That's how an event gets handled.


### What Is Context?

Some data called a `context` is threaded through all the calls. 

This value is passed as the argument to every `before` and `after` 
function and it is returned by each function, possibly modified. 

A `context` is a map with this structure: 
```javascript
{
    coeffects: {
        event: ['some-id', 'some-param'],
        db: <original contenxt of appDb>
    },
    effects: {
        db: <new value for app-db>,
        dispatch: ['an-event-id', 'param1']
    },
    queue: <a collection of further interceptors>,
    stack: <a collection of interceptors already walked>
}
```

`context` has `coeffects` and `effects` which, if this was a web
server, would be somewhat analogous to `request` and `response`
respectively.

`coeffects` will contain the inputs required by the event handler
(sitting presumably on the end of the chain). So that's 
data like the `event` being processed, and the initial state of `db`.

The handler-returned side effects are put into `effects` including, 
but not limited to, new values for `db`.

The first few interceptors in a chain (inserted by `regEventDb`) 
have `before` functions which __prime__ the `coeffects`
by adding in `event`, and `db`.  Of course, other interceptors can
add further to `coeffects`.  Perhaps the event handler needs
data from localstore, or a random number, or a 
DataScript connection. Interceptors can build up `coeffects`, via their 
`before`. 

Equally, some interceptors in the chain will have `after` functions
which process the side effects accumulated into `effects`
including, but not limited to, updates to app-db.

### Self Modifying

Through both stages (before and after), `context` contains a `:queue`
of interceptors yet to be processed, and a `:stack` of interceptors
already done.

In advanced cases, these values can be modified by the Interceptor
functions through which the `context` is threaded. 

What I'm saying is that interceptors can be dynamically added 
and removed from the `queue` by existing Interceptors. 

### Credit

> All truths are easy to understand once they are discovered <br>
>   -- Galileo Galilei

> Things always become obvious after the fact <br>
>   -- Nassim Nicholas Taleb

This elegant and flexible arrangement was originally 
designed by the talented 
[Pedestal Team](https://github.com/pedestal/pedestal/blob/master/guides/documentation/service-interceptors.md). Thanks! 

### Write An Interceptor

Dunno about you, but I'm easily offended by underscores.

If we had a view  which did this: 
```javascript
dispatch(['delete-item', 42])
```

We'd have to write this handler: 
```javascript
regEventDb(
    'delete-item',
    (db, [_, keyToDelete]) =>               // <---- Arrgggghhh underscore 
        db.delete(keyToDelete)
)
```

Do you see it there? In the event destructuring!!! Almost mocking us with that
passive aggressive, understated thing it has going on!! Co-workers
have said I'm "being overly sensitive", perhaps even pixel-ist, but 
you can see it, right? Of course you can.

What a relief it would be to not have it there, but how? We'll write an interceptor: `trim-v`

Once we have written `trim-v`, our registration will change to look like this:
```javascript
regEventDb(
    'delete-item',
    [reframe.trimv],                        // <--- interceptor added
    (db, [keyToDelete]) =>                  // <---yaaah!  no leading underscore
        db.delete(keyToDelete)
)
```

`trim-event` will need to change the `coeffects` map (within `context`).  Specifically, it will be 
changing the `event` value within the `coeffects`. 

`event` will start off as `[delete-item 42]`, but will end up `[42]`.  `trim-event`  will remove that 
leading `delete-item` because, by the time the event is 
being processed, we already know what id it has.

And, here it is: 
```javascript
const trimv = reframe.toInterceptor({
    id: 'trim-event',
    before(ctx) {
        return ctx.updateIn(
            ['coeffects', 'event'], 
            (event) => event.slice(1));
    }
})
```

As you read this, look back to what a `context` looks like.   

Notes:
  1. We use `toInterceptor` to create an interceptor (which is just a map)
  2. Our interceptor only has a `before` function 
  3. Our `before` is given `context`.  It modifies it and returns it.
  4. There is no `after` for this Interceptor. It has nothing to do 
     with the backwards processing flow of `effects`. It is concerned only 
     with `coeffects` in the forward flow.
      
### Wrapping Handlers

We're going well. Let's do an advanced wrapping. 

Earlier, in the "Handlers Are Interceptors Too" section, I explained that `event handlers` 
are wrapped in an Interceptor and placed on the end of an Interceptor chain.  Remember the 
whole `[std1, std2, in1, in2, h]` thing?  

We'll now look at the `h` bit. How does an event handler get wrapped to be an Interceptor?

Reminder - there's two kinds of handler:
   - the `-db` variety registered by `regEventDb`
   - the `-fx` variety registered by `regEventFx`

I'll now show how to wrap the `-db` variety. 

Reminder: here's what a `-db` handler looks like:
```javascript
function (db, event) {              // takes two params
    return db.set('flag', true);    // returns a new db
}
```

So, we'll be writing a function which takes a `-db` handler and returns an 
Interceptor which wraps that handler:
```javascript
function dbHandlerToInterceptor(dbHandlerFn) {
    return toInterceptor({                                      // a utility function supplied by re-frame
        id: 'db-handler',                                       // ids are decorative only
        before(ctx) {
            const 
                db = ctx.getIn(['coeffects', 'db']),            // extract db and event from coeffects
                event = ctx.getIn(['coeffects', 'event']),
                newDb = dbHandlerFn(db, event);                 // call the handler 
            return ctx.setIn(['effects', 'db'], newDb);         // put db back into :effects
        }
    });
}
```

Notes:
  1.  Notice how this wrapper extracts data from the `context's` `coeffects` 
      and then calls the handler with that data  (a handler must be called with `db` and `event`)
  2.  Equally notice how this wrapping takes the return value from the `-db` 
      handler and puts it into `context's` `effects`
  3.  The modified `context` (it has a new `effects`) is returned
  3.  This is all done in `before`.  There is no `after` (it is a noop).  But this 
      could have been reversed with the work happening in `after` and `before` a noop. Shrug.
      Remember that this Interceptor will be on the end of a chain. 

Feeling confident?  Try writing the wrapper for `-fx` handlers - it is just a small variation.  

## Summary

In this tutorial, we've learned: 
 
__1.__ When you register an event handler, you can supply a collection of interceptors:
```javascript
regEventDb(
    'some-id',
    [in1, in2],                 // <--- a chain of 2 interceptors
    (db, v) => {                // <-- real handler here
        ...
    }
)
```

__2.__ When you are registering an event handler, you are associating an event id with a chain of interceptors including:
  - the ones you supply (optional)
  - an extra one on the end, which wraps the handler itself 
  - a couple at the beginning of the chain, put there by the `regEventDb` or `regEventFx`. 
  
__3.__ An Interceptor Chain is executed in two stages. First a forwards sweep in which 
  all `before` functions are called, and then second, a backwards sweep in which the 
  `after` functions are called. A `context` will be threaded through all these calls.

__4.__ Interceptors do interesting things:
   - add to coeffects  (data inputs to the handler)
   - process side effects (returned by a handler)
   - produce logs 
   - further process    

In the next Tutorial, we'll look at (side) Effects in more depth.  Later again, we'll look at Coeffects.   
 
## Appendix

### The Built-in Interceptors

re-frame comes with some built-in Interceptors:
  - __debug__: log each event as it is processed. Shows incremental [`clojure.data/diff`](https://clojuredocs.org/clojure.data/diff) reports.
  - __trimv__:  a convenience. More readable handlers.  

And some Interceptor factories (functions that return Interceptors):
  - __enrich__:  perform additional computations (validations?), after the handler has run. More derived data flowing.
  - __after__: perform side effects, after a handler has run.  Eg: use it to report if the data in `appDb` matches a schema.   
  - __path__:  a convenience. Simplifies our handlers. Acts almost like `updateIn`.
  
In addition, [a Library like re-frame-undo](https://github.com/Day8/re-frame-undo) provides an Interceptor 
factory called `undoable` which checkpoints app state.
 
<!--
To use them, first require them:
```Clojure
(ns my.core
  (:require
    [re-frame.core :refer [debug path]])
```
-->

***

Previous:  [Effectful Handlers](EffectfulHandlers.md)&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
Up:  [Index](README.md)&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
Next:  [Effects](Effects.md)  

+++
title = "Cooperative Concurrency in OCaml: A Core.Std.Async example"
date = "2014-07-09"
slug = "2014/07/09/core-dot-async-example"
Categories = []
+++

I've been working on an OCaml [Mqtt](http://mqtt.org/) client for eventual use with [MirageOS](http://mirageos.org/). Mqtt is a lightweight publish/subscribe messaging transport protocol that's aimed at embedded systems and IoT applications. Without going into much detail about the protocol, implementing it requires some form of concurrency as there is a keep-alive ping that needs to be sent from the client to the broker at regular intervals to keep the connection alive. The client can also be receiving messages from the broker and needs to be able to queue up those messages for processing. 

When considering concurrency options for OCaml, I thought I'd give Core.Std.Async (from here on, I'll just refer to it as Async) a try as it's covered in pretty good detail in [chapter 18 of Real World OCaml](https://realworldocaml.org/v1/en/html/concurrent-programming-with-async.html). Of course, I didn't see exactly what I needed in the examples there, so I had to play with a toy producer/consumer example for a bit. The code appears below, hopefully you'll find it useful if you want to try out Async:

title = "core dot async example"
{{<highlight ocaml "linenostart=1" >}}
(* compile with:
   corebuild -pkg async,unix  producer_consumer.native
*)
open Sys
open Async.Std
open Core
 
let (r,w) = Pipe.create () 
 
(* producer *)
let countup hi w =
  let rec loop i = 
    printf "i=%d\n" i ;
    if (i < hi &&( not (Pipe.is_closed w))) then 
       Pipe.write w i >>>
       fun () -> loop (i+1)
     else Pipe.close w
  in 
  loop 0 ;;
 
(* consumer *)
let rec readloop r = 
  Pipe.read r >>=
  function
  | `Eof -> return ()
  | `Ok v -> return (printf "Got %d\n" v) >>=
             fun () -> after (Time.Span.of_sec 0.5) >>=
             fun () -> readloop r  ;;
 
 
let () =
  Pipe.set_size_budget r 256  ;
  countup 10 w;
  ignore(readloop r);
  Core.Never_returns.never_returns (Scheduler.go ()) 

{{< / highlight >}}

Async falls under the category of cooperative concurrency along with other frameworks that have an event loop like Python's Twisted and Node.js. Preemptive concurrency is on the other side of the concurrency divide, and generally involves threads and mutexes. Cooperative concurrency is usually done by passing callbacks to functions that get called after some call in the function has returned data. Node.js is famous for this: you need to read data from some other website, for example, so you call your function for doing this and pass a callback to it that will actually return the data after it becomes available. Meanwhile instead of blocking waiting for the data to come back your program has gone on to call other functions in the event loop. This works fairly nicely (for some value of *nicely*) in Javascript where there are very few blocking calls... but note that [callback hell](http://callbackhell.com/) is a thing as callbacks get nested into callbacks sometimes several layers deep.

Async works a bit differently, at least on the surface, from the asynchronous style of Javascript used in Node.js and this seems to help avoid the nested callback problem. Async is very monadic and that seems to help. Well-behaved Async functions don't block, but instead return a value of type *Deferred.t* (Async.Deferred.t). A *'a Deferred.t* value is kind of like a *promise* in some other languages, a promise that the value you seek will be computed sometime in the future. There's a scheduler that schedules these deferred events. 

In addition, Async also has these very handy *Pipe* thingys that are basically FIFOs that connect different parts of your program together in a way that is reminiscent of [Kahn Process Networks](http://en.wikipedia.org/wiki/Kahn_process_networks). (Async Pipes don't have anything to do with Unix pipes other than a very surfacy resemblence)

You'll notice that a *Pipe* is created at line 5 of the code above where *r* is the reader end of the pipe and *w* is the writer end. The *countup* function is our producer and simply counts up to a value passed in. On every iteration of the *loop* the new count value gets writen to the pipe at line 12 ( *Pipe.write w i* ). Notice that there's an *>>>* operator at the end of that line. This is also called *upon* and could also have been written as: *upon (Pipe.write w i) (fun () -> loop (i+1))*. *upon* has the following type: *'a Deferred.t -> ('a -> unit) -> unit*.  When the deferred value that's passed into *upon* is determined, the callback will be called, in this case we call *loop (i+1)* and recurse. 

The readloop function (starting at line 19) uses the bind operator ( >>= ) which has the type: *'a Deferred.t -> ('a -> 'b Deferred.t) -> 'b Deferred.t*. When the deferred value of *Pipe.read r*  is determined, the result is passed into the callback function which will print what was received from the pipe (on success), then wait a half second before going on to call *readloop* again. For more details about the sequencing operators >>= (*bind*), >>> (*upon*) and >>| (*map*) have a look at the *Real World OCaml* link above. Notice that we use *return* to wrap a value in a 
*Deferred.t* (the type of *return* is *'a -> 'a Deferred.t*) in order to make *readloop* type-check and compile as *readloop*'s type is: *int Pipe.Reader.t -> unit Deferred.t*.

So *countup* writes values to the *Pipe* while *readloop* reads values from it at half second intervals. If you compile this program and run it you'll see:

    i=0
    i=1
    Got 0
    i=2
    i=3
    i=4
    i=5
    i=6
    i=7
    i=8
    i=9
    i=10
    Got 1
    Got 2
    Got 3
    Got 4
    Got 5
    Got 6
    Got 7
    Got 8
    Got 9

The first lines up until 'i=10' come out pretty much instantaneously, while the next lines ('Got 1' through 'Got 9') came out every half second.

However... Notice line 32 of the code above: *Pipe.set_size_budget r 256  ;*
Initially I thought (based on the explanation in RWO) that *Pipe*s were queues of infinite length. I didn't know you had to *set_size_budget* on the Pipe. So earlier attempts omitted line 32 above and I got the following result:

    i=0
    i=1
    Got 0
    i=2
    Got 1
    i=3
    Got 2
    i=4
    Got 3
    i=5
    Got 4
    i=6
    Got 5
    i=7
    Got 6
    i=8
    Got 7
    i=9
    Got 8
    i=10
    Got 9

(With a half second delay between each 'Got' message)

The *Pipe.write* seemed to block waiting for a *Pipe.read* which didn't seem right given the description of *Pipe* in RWO. But as it turns out the pipe size defaults to 0 which means that the pipe fills up immediately. To get a deeper pipe, you need to call *set_budget_size* on the pipe (either end, apparently) as shown on line 32 of the code above. (Aside: It seems like there should be an optional parameter *budget_size* to the *Pipe.create* function.)

**Conclusions**

While in essense we're still using callbacks in our event loop (very Javascript/Node.js-ish), the monadic sequencing operators >>=, >>| and >>> do seem to make the code easier to read and reason about. Add in the handy *Pipe*s and it's not a bad way to do concurrency, really, once you get the hang of dealing with the monads. It should be noted, however, that as in most cooperative concurrency schemes, you're only going to be using one core of your processor.

Finally... I'm pretty new to Core and Async so if I've made some mistakes here in the explanation or if the code could be more elegant please don't hesitate to let me know in the comments.


 




  

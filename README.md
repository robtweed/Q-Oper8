# Q-Oper8

# IMPORTANT NOTICE

# THIS MODULE HAS NOW BEEN DEPRECATED - USE ewd-qoper8 INSTEAD

See [https://github.com/robtweed/ewd-qoper8.git](https://github.com/robtweed/ewd-qoper8.git)

# ==========================================

Simple multi-process manager for Node.js

Rob Tweed <rtweed@mgateway.com>  
30 August 2011, M/Gateway Developments Ltd [http://www.mgateway.com](http://www.mgateway.com)  

Twitter: @rtweed

## Installing Q-Oper8

       npm install qoper8


Note: Q-Oper8 will now work on Node.js version 0.4.x

##  What is Q-Oper8?

*Q-Oper8* is an easy-to-use module for enabling and managing a scalable but high-performance multi-process 
environment in Node.js.  The primary incentive for writing it was to create a hybrid environment where 
the many benefits of synchronous coding could be acheived without the risk of blocking the main server process,
but, as a side effect of its architecture, it also returns many other significant benefits for users of Node.js.

It consists of four components:

- a master Node server process that is 100% asynchronous and non-blocking
- a pool of *persistent* child Node processes that **only ever handle a single action/request at a time**
- a queue of pending actions/requests
- a queue processor that attempts to allocate requests/actions on the queue to available child Node processes

The key aspect of the Q-Oper8 architecture is that you pre-determine the size of your child Node process pool and 
these are started before anything else happens.  Subsequently, your child Node processes persist throughout the lifetime of 
the master Node server process, so there is no setup/teardown 
overhead or latency when handling requests/actions: the child processes that are flagged as being in the 
*available process pool* are instantaneously available for use by the master process.

It's the fact that each Child Node Processes only ever handle a single request at a time that makes it 
possible for them to support synchronous coding, since they don't need to worry about blocking anyone else.

Note: *Q-Oper8* is completely event-driven with no polling overheads or delays.  Processing of the queue 
is automatically triggered by two main events:

- adding a request to the queue
- a child Node process becoming available

You have complete control over the behaviour and configuration of *Q-Oper8*.  You can:

- pre-determine the number of child Node processes that constitute the worker pool, based on your application's workload and 
  available system resources
- write the child process logic that you require for handling your actions/requests, which may be synchronous, asynchronous or any 
  combination of sync and async
- define a handler that runs on the master server to process the completed responses sent back from the child processes, eg 
  to return the response as a web page to the originating client.  It can even put more requests/actions back on the queue if
  necessary

Clearly, the larger the pool of Node.js processes, the less likely it is that the request/action queue will build up.  On the 
other hand, each child Node process uses about 10Mb memory according to the Node.js documentation.  Additionally, the quicker 
your child process can handle a request, the sooner it will become available again to the pool to handle a queued request/action.

In basic tests, optimum throughput appears to occur when the number of child processes is equal to, or one less than the 
number of CPU cores.  However other factors may affect the optimum number of child processes you use.  See 
the benchmark figures later in this documentation.

The architecture of Q-Oper8 was modified in build 8 to transmit batches of messages between the master process 
and the child processes.  This has improved throughput to nearly 5 times that of the original versions.  
Optimum throughput occurred on the test machine when the message buffer 
length was set to 8192 bytes, and so this is the default.  However you may find that altering the 
*maxMsgLength* startup parameter will improve throughput.

##  Benefits of the Q-Oper8 module

The *Q-Oper8* module addresses many of the key potential drawbacks of Node.js, including:

- allowing safe use of synchronous logic with Node.js, avoiding the need for cumbersome, non-intuitive and difficult to maintain 
  nested callback structures.  Synchronous logic won't block the main server process
- allowing the safe use of in-process synchronous database APIs, as exemplified by the [Globals](http://globalsdb.org) database
- providing protection to your main Node server process by isolating it from problems that might occur when handling particular 
  requests/actions.
- isolating the main server process from processing that requires significant amounts of computation that would otherwise 
  slow down the performance of the main server process for all concurrent users
- distributing load across multiple Node processes, allowing Node to exploit multiple-core CPUs.
- providing a highly scalable architecture that can be easily tailored to suit your traffic and processing demands.

##  Using the Q-Oper8 module

The /examples directory contains a simple worked example of a master server process and a child process to handle web requests.

The following is a simple example of how to use the *Q-Oper8* module:

      var qoper8 = require('qoper8');
	  
      qoper8.start('', function() {
        console.log("Q-Oper8 started!!!");
		// start processing!
      });

An action is defined as a standard Javascript object.  Its structure and content is up to you, but note that the object that 
is passed to a child Node process cannot contain functions.  

You simply add your action to thread's queue and let it do the rest, eg:

       var requestObj = {action: {x:1,y:2}, response: response, otherStuff: 'etc...'};
       qoper8.addToQueue(requestObj, responseHandler);

      
The only part of the requestObj object that is sent to and handled by your child process is the *action* property.  
You can add any other properties to the requestObj object: these are retained within the master Node process 
for use by *responseHandler*: the master Node process response handler that you must also define (see later).

So, in the example above, if we're using the scheduler with Node's HTTP server module, we're adding the response object to the 
requestObj object so that the master Node process has the correct handle to allow it to ultimately return the response to the correct client.

##  Startup parameters

The parameters that you can specify for the *Q-Oper8* *start()* function are as follows:

- poolSize = the number of Node child processes to fire up (default = 4)
- maxMsgLength = the length in bytes of the message buffer used to transfer action/requests from the queue 
 to a child process (default = 8192)
- childProcessPath = the filepath of the Node child process Javascript file (default = __dirname + '/qoper8ChildProcess.js')
- monitorInterval = no of milliseconds delay between displaying process usage in console (default = 30000)
- silentStart = true if you don't want any message to the console when *Q-Oper8* starts (default = false)

For example:

      var qoper8 = require('qoper8');
	  
      var params = {poolSize: 20, childProcessPath: '/home/user/node/myChildProc.js', maxMsgLength: 4096};
      qoper8.start(params, function() {
        console.log("Q-Oper8 started!!!");
          // start processing!
      });

##  Defining a child Node process

The child process should be designed to handle any instance of your action/requests that get sent to it by the master
Node process.  Note that a child Node process will only handle one single action/request at a time, so you are at liberty to 
use as much synchronous coding as you like, since there will be no other users to block.

Here's a simple example:

       var childProcess = require('qoper8').childProcess;
       
       var actionMethod = function(action) {
         console.log("Action method: Process " + process.pid + ": action = " + JSON.stringify(action));
         var result = "method completed for " + process.pid + " at " + new Date().toLocaleTimeString();
         return result;
       };

       childProcess.handler(actionMethod);

So you first define your *actionMethod* which will process the contents of the action object that you originally placed 
on the queue.  This method can do anything you like, and must return a value or object.  This returnValue will be automatically sent back to 
the master Node process which will look after what is done with it, eg sending it back to a user as a web page.

Then just add the last line exactly as shown above.  That's it!  *Q-Oper8* will do the rest.

By default, Q-Oper8 will assume that your child process *actionMethod()* function is synchronous and that it 
will return the result as shown above.  However, you'll often require the use of asynchronous logic in 
your *actionMethod()* function, in which case the result must be returned via a callback function.  To cater for 
this, the childProcess.handler() function can take a second argument: 

       isAsync // true|false

Here's the previous example, written as an asynchronous version:

       var childProcess = require('qoper8').childProcess;
       
       var actionMethod = function(action, callback) {
         console.log("Action method: Process " + process.pid + ": action = " + JSON.stringify(action));
         var result = "method completed for " + process.pid + " at " + new Date().toLocaleTimeString();
         callback(result)
       };

       childProcess.handler(actionMethod, true);



## Defining the master Node Results Handler method

You define this in the master Node process.  Here's an example that will package up the resonses from the child Processes
as web pages:

      var responseHandler = function(requestObj, results) {
        //console.log("This is the response handler: ");
        //console.log("** action: " + JSON.stringify(requestObj.action));
        //console.log("results = " + JSON.stringify(results));

        var response = requestObj.response;
        var html = "<html>";
        html = html + "<head><title>Q-Oper8 action response</title></head>";
        html = html + "<body>";
        html = html + "<p>Action was processed !</p><p>Results: " + results + "</p>";
        html = html + "</body>";
        html = html + "</html>";

        response.writeHead(200, {"Content-Type": "text/html"});  
        response.write(html);  
        response.end();  
      };

- requestObj will be picked up automatically by *Q-Oper8* and is the original requestObj you placed on the queue
- results is the returnValue you returned from your childProcesses.

You add a reference to this handler whenever you add a request/action to the *Q-Oper8* queue, eg:

       qoper8.addToQueue(requestObj, responseHandler);

By default, Q-Oper8 will release a child process back to the available pool as soon as master Node
Results Handler method has executed.  This is OK if the handler method does not contain asynchronous 
logic.  If it does, there's a risk that the *requestObj* object gets overwritten by the next action before 
it is fully processed by the first action handler method.  There are two ways to avoid this:

- extract any values from the *requestObj* object into Javascript variables at the start of the handler method, 
and use these values within any callback functions rather than properties of the requestObj object.

- manually take control by using:

         qoper8.asyncHandler();

This instructs Q-Oper8 not to automatically release the child process after the handler method completes.  
Instead, you manually release the child process using:

         qoper8.makeProcessAvailable(pid);

You can obtain the value of pid by specifying it as a third argument of your handler method.  So, the 
example above could be rewritten as follows:

      qoper8.asyncHandler();

      var responseHandler = function(requestObj, results, pid) {
        //console.log("This is the response handler: ");
        //console.log("** action: " + JSON.stringify(requestObj.action));
        //console.log("results = " + JSON.stringify(results));

        var response = requestObj.response;
        var html = "<html>";
        html = html + "<head><title>Q-Oper8 action response</title></head>";
        html = html + "<body>";
        html = html + "<p>Action was processed !</p><p>Results: " + results + "</p>";
        html = html + "</body>";
        html = html + "</html>";

        response.writeHead(200, {"Content-Type": "text/html"});  
        response.write(html);  
        response.end();
        qoper8.makeProcessAvailable(pid);
      };

	   
## That's it!

You now have the best of all worlds: a non-blocked scalable Node server process, with a pool of child Node processes in which you 
can use synchronous logic without any concern about blocking other users.  You can also make use of your multi-core processor into 
the bargain!

## Running the Example

- Find the simple worked example in the */examples* directory of this repository.

- Copy *qoper8ChildProcess.js* to the same directory/path used by npm when you installed the *Q-Oper8* module, 
   eg *~/node/node_modules/qoper8/lib* (Note: you can change the name and location of the childProcess file by 
   using the *childProcessPath* startup parameter for *Q-Oper8*)

- Copy *webQOper8.js* and *qoper8Test.js* to the directory/path where you normally run your Node.js applications, eg *~/node*

- Start the web example: **node webQOper8.js**

You'll now have a web server running on port 8080 (edit *webQOper8.js* if you want to use a different port)

Now start a browser and point it at the Node application's IP address and port.  Make sure you use a URL that includes */test/*, eg:

       http://192.168.1.100:8080/test/it
	   
You should get a response that is something like:

       Action was processed!
	   
	   Results: method completed for 10372 at 09:40:07
	   
You should also have seen a flurry of activity in the Node.js console, because the *trace* flag in *Q-Oper8* has been set to *true*.

The key lines that made it all burst into life are in *webQOper8.js*:

       // *********Example use of threaded action ******** 

       if (uri.indexOf('/test/') !== -1) {
         var action = {query: urlObj.query};
         var requestObj = {action: action, request: request, response: response, urlObj: urlObj};
         qoper8.addToQueue(requestObj, handler);
       }

       // **************************************************

ie the request was added to the *Q-Oper8* queue when you sent the URL.  The request was then processed by the *actionMethod* in 
one of the running instances of *qoper8ChildProcess.js* and the HTML response was generated by the *handler* function in *webQOper8.js*

Now try out the second example: *qoper8Test*, which contains a loop that adds 5000 simple requests onto the queue.  It will show you 
how long it takes to process all the requests.  Try altering the poolSize and number of requests to see how these affect the 
total processing time.  To run the second example:

       node qoper8Test.js
	   
## Benchmark Tests

Benchmark tests on a relatively 
low-powered, single CPU server showed a maximum sustained throughput of over 90,000 requests/actions per second.  
This is the rate at which requests can be added to the queue without the queue growing or being exhausted.

On a 4 X CPU AMD Opteron-based (2.1 GHz) server with 8Gb memory, running Ubuntu 10.10 server 
and Node.js 0.4.0:

      1 child process:   19,580 requests/sec
      2 child processes: 40,540 requests/sec
      3 child processes: 59,920 requests/sec
      4 child processes: 90,630 requests/sec
      5 child processes: 88,120 requests/sec
      6 child processes: 85,570 requests/sec
      7 child processes: 65,460 requests/sec
      8 child processes: 63,200 requests/sec
      9 child processes: 61,670 requests/sec
     10 child processes: 60,730 requests/sec

You can try out the benchmark tests for yourself: see *benchmark.js* which you will find in the */examples* directory of this
 repository.  You'll also find the *benchmarkChildProc.js* file which will run in your child processes. Make 
sure you change the *childProcessPath* startup parameter in *benchmark.js* to the appropriate file path.

You can adjust the queue-creation rate by changing the maximum queue batch size and creation loop interval parameters.  
 I will be grateful for any results for your particular configuration.

   
## License

(The MIT License)

Copyright (c) 2011 M/Gateway Developments Ltd,
Reigate, Surrey UK.
All rights reserved.

http://www.mgateway.com
Email: rtweed@mgateway.com

Permission is hereby granted, free of charge, to any person obtaining a copy of this software and 
associated documentation files (the 'Software'), to deal in the Software without restriction, including 
without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell 
copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the 
following conditions:

The above copyright notice and this permission notice shall be included in all copies or substantial 
portions of the Software.

THE SOFTWARE IS PROVIDED 'AS IS', WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT 
LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. 
IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, 
WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE 
SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.


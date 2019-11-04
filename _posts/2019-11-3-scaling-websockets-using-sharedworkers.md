---
layout: post
title: Scaling WebSocket Connections using Shared Workers
comments: true
---

You can find the code for this post on [SharedWorker WebSocket example](https://github.com/ayushgp/shared-worker-socket-example).

## Web Sockets
Web Sockets allow real-time communication between the client browser and a server. They are different from HTTP because they not only allow client to request data from the server but also allow server to push data from the server. 

![image](https://user-images.githubusercontent.com/7992943/68089345-efe33700-fe8d-11e9-8953-ba1b38b51d81.png)

### The Problem
But in order to allow this each client needs to open a connection with the server and keep it alive till the time client closes the tab/goes offline. They create a persistent connection. This makes the interaction stateful, leading both client and server to store at least some data in memory on the WebSocket server for each open client connection. 

So if a client has 15 tabs open, they'll have 15 open connections to the server. This post is an attempted solution to try and reduce this load from a single client. 

![image](https://user-images.githubusercontent.com/7992943/68089325-b27ea980-fe8d-11e9-9527-9a4c09cc3d1c.png)

## `WebWorkers`, `SharedWorkers` and `BroadcastChannels` to the rescue
**[Web Workers]((https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API/Using_web_workers))** are a simple means for web content to run scripts in background threads. The worker thread can perform tasks without interfering with the user interface. Once created, a worker can send messages to the JavaScript code that created it by posting messages to an event handler specified by that code (and vice versa).

**[Shared Workers](https://developer.mozilla.org/en-US/docs/Web/API/SharedWorker)** are a type of web workers that can be accessed from several browsing contexts, such as several windows, iframes or even workers.

**[Broadcast Channels](https://developer.mozilla.org/en-US/docs/Web/API/Broadcast_Channel_API)**  allows simple communication between [browsing contexts](https://developer.mozilla.org/en-US/docs/Glossary/browsing_context "browsing contexts: A browsing context is the environment in which a browser displays a Document (normally a tab nowadays, but possibly also a window or a frame within a page).") (that is _windows_, _tabs_, _frames_, or _iframes_) with the same [origin](https://developer.mozilla.org/en-US/docs/Glossary/origin "origin: Web content's origin is defined by the scheme (protocol), host (domain), and port of the URL used to access it. Two objects have the same origin only when the scheme, host, and port all match.").

_All the above definitions are from MDN._

### Reducing the server load using SharedWorkers
We can use `SharedWorker` for solving this problem of a single client having multiple connections open from the same browser. Instead of opening a connection from each tab/browser window, we can instead use a `SharedWorker` to open the connection to the server. 

This connection will be open until all the tabs to the website are closed. And the single connection can be used by all the open tabs to communicate with and receive messages from the server. 

We'll use the broadcast channels API to broadcast state change of the web socket to all the contexts (tabs).

## Setting up a basic Web Socket Server
Let us now jump in the code. For the purpose of this post, we'll set up a very simple web server that supports socket connections using the `ws` npm module. Initialize a npm project using:

```bash
$ npm init
```

Run through the steps, once you have a `package.json` file, add the `ws` module and `express` for a basic http server:

```bash
$ npm install --save ws express
```

Once you have this, create a index.js file with the following code to set up your static server serving files from `public` directory at port 3000 and running a `ws` server at port 3001:

```js
const  express  =  require("express");
const  path  =  require("path");
const  WebSocket  =  require("ws");
const  app  =  express();

// Use the public directory for static file requests
app.use(express.static("public"));

// Start our WS server at 3001
const wss = new WebSocket.Server({ port: 3001 });

wss.on("connection", ws => {
  console.log('A new client connected!');
  ws.on("message", data => {
    console.log(`Message from client: ${data}`);

    // Modify the input and return the same.
    const  parsed  =  JSON.parse(data);
    ws.send(
      JSON.stringify({
        ...parsed.data,
        // Additional field set from the server using the from field.
        // We'll see how this is set in the next section.
        messageFromServer: `Hello tab id: ${parsed.data.from}`
      })
    );
  });
  ws.on("close", () => {
    console.log("Sad to see you go :(");
  });
});

// Listen for requests for static pages at 3000
const  server  =  app.listen(3000, function() {
  console.log("The server is running on http://localhost:"  +  3000);
});
```

## Creating a `SharedWorker`
To create any type of a `Worker` in JavaScript, you need to create a separate file that defines what the worker will do.

Within the worker file, you need to define what to do when this worker is initialized. This code will only be called once when the `SharedWorker` is initialized. After that until the last tab connecting to this worker is not closed/ends connection with this worker, this code cannot be rerun.

We can define a `onconnect` event handler to handle each tab connecting to this `SharedWorker`. Let us look at the `worker.js` file. 

```js
// Open a connection. This is a common
// connection. This will be opened only once.
const ws = new WebSocket("ws://localhost:3001");

// Create a broadcast channel to notify about state changes
const broadcastChannel = new BroadcastChannel("WebSocketChannel");

// Mapping to keep track of ports. You can think of ports as
// mediums through we can communicate to and from tabs.
// This is a map from a uuid assigned to each context(tab)
// to its Port. This is needed because Port API does not have
// any identifier we can use to identify messages coming from it.
const  idToPortMap  = {};

// Let all connected contexts(tabs) know about state cahnges
ws.onopen = () => broadcastChannel.postMessage({ type: "WSState", state: ws.readyState });
ws.onclose = () => broadcastChannel.postMessage({ type: "WSState", state: ws.readyState });

// When we receive data from the server.
ws.onmessage  = ({ data }) => {
  console.log(data);
  // Construct object to be passed to handlers
  const parsedData = { data:  JSON.parse(data), type:  "message" }
  if (!parsedData.data.from) {
    // Broadcast to all contexts(tabs). This is because 
    // no particular id was set on the from field here. 
    // We're using this field to identify which tab sent
    // the message
    broadcastChannel.postMessage(parsedData);
  } else {
    // Get the port to post to using the uuid, ie send to
    // expected tab only.
    idToPortMap[parsedData.data.from].postMessage(parsedData);
  }
};

// Event handler called when a tab tries to connect to this worker.
onconnect = e => {
  // Get the MessagePort from the event. This will be the
  // communication channel between SharedWorker and the Tab
  const  port  =  e.ports[0];
  port.onmessage  =  msg  => {
    // Collect port information in the map
    idToPortMap[msg.data.from] =  port;
    
    // Forward this message to the ws connection.
    ws.send(JSON.stringify({ data:  msg.data }));
  };

  // We need this to notify the newly connected context to know
  // the current state of WS connection.
  port.postMessage({ state: ws.readyState, type: "WSState"});
};
```

There are a few things we've done here that may not be clear from the start. As you read through the post, these things will become clear as to why we did those. Still some points I want to clarify on:

* We're using the Broadcast Channel API to broadcast the state change of the socket.
* We're using `postMessage` to the port on connection to set the initial state of the context(tab). 
* We're using the `from` field coming from the context(tabs) themselves to identify where to redirect the response. 
* In case we don't have a `from` field set from the message coming from the server, we'll just broadcast it to everyone!

**Note**: `console.log` statements here won't work in your tab's console. You need to open the SharedWorker console to be able to see those logs. To open the dev tools for SharedWorkers, head over to chrome://inspect.

## Consuming a `SharedWorker`
Let us first create an HTML page to house our script that'll consume the `SharedWorker`.

```html
<!DOCTYPE  html>
<html  lang="en">
<head>
  <meta  charset="UTF-8"  />
  <title>Web Sockets</title>
</head>
<body>
  <script  src="https://cdnjs.cloudflare.com/ajax/libs/node-uuid/1.4.8/uuid.min.js"></script>
  <script  src="main.js"></script>
</body>
</html>
```

So we've defined our worker in `worker.js` file and set up a HTML Page. Now let us look at how we can use this shared web socket connection from any context(tab). Create a `main.js` file with the following contents:

```js
// Create a SharedWorker Instance using the worker.js file. 
// You need this to be present in all JS files that want access to the socket
const worker = new SharedWorker("worker.js");

// Create a unique identifier using the uuid lib. This will help us
// in identifying the tab from which a message was sent. And if a 
// response is sent from server for this tab, we can redirect it using
// this id.
const id = uuid.v4();

// Set initial web socket state to connecting. We'll modify this based
// on events.
let  webSocketState  =  WebSocket.CONNECTING;
console.log(`Initializing the web worker for user: ${id}`);

// Connect to the shared worker
worker.port.start();

// Set an event listener that either sets state of the web socket
// Or handles data coming in for ONLY this tab.
worker.port.onmessage = event => {
  switch (event.data.type) {
    case "WSState":
      webSocketState = event.data.state;
      break;
    case "message":
      handleMessageFromPort(event.data);
      break;
  }
};

// Set up the broadcast channel to listen to web socket events.
// This is also similar to above handler. But the handler here is
// for events being broadcasted to all the tabs.
const broadcastChannel = new BroadcastChannel("WebSocketChannel");
broadcastChannel.addEventListener("message", event => {
  switch (event.data.type) {
    case  "WSState":
      webSocketState  =  event.data.state;
      break;
    case  "message":
      handleBroadcast(event.data);
      break;
  }
});

// Listen to broadcasts from server
function  handleBroadcast(data) {
  console.log("This message is meant for everyone!");
  console.log(data);
}

// Handle event only meant for this tab
function  handleMessageFromPort(data) {
  console.log(`This message is meant only for user with id: ${id}`);
  console.log(data);
}

// Use this method to send data to the server.
function  postMessageToWSServer(input) {
  if (webSocketState  ===  WebSocket.CONNECTING) {
    console.log("Still connecting to the server, try again later!");
  } else  if (
    webSocketState  ===  WebSocket.CLOSING  ||
    webSocketState  ===  WebSocket.CLOSED
  ) {
    console.log("Connection Closed!");
  } else {
    worker.port.postMessage({
      // Include the sender information as a uuid to get back the response
      from:  id,
      data:  input
    });
  }
}

// Sent a message to server after approx 2.5 sec. This will 
// give enough time to web socket connection to be created.
setTimeout(() =>  postMessageToWSServer("Initial message"), 2500);```
```

### Sending Messages to `SharedWorker`
As we've seen above, you can send messages to this `SharedWorker` using `worker.port.postMessage()`. You can pass any JS object/array/primitive value here. 

A good practice here can be passing an object that specifies from what context the message is coming so that the worker can take action accordingly. So for example, if we have a chat application and one of the tabs wants to send a message, we can use something like:

```js
{
    // Define the type and the 
  type: 'message',
  from: 'Tab1'
  value: {
    text: 'Hello',
    createdAt: new Date()
  }
}
```
If we have a file sharing application, on deleting a file, the same structure can be used with a different type and value: 

```js
{
  type: 'deleteFile',
  from: 'Tab2'
  value: {
    fileName: 'a.txt',
    deletedBy: 'testUser'
  }
}
```

This will allow the Worker to decide what to do with it. 

### Listening to messages from the worker
We had set up a map in the beginning to keep track of `MessagePorts` of different tabs. We then set up a `worker.port.onmessage` event handler to handle events coming from the `SharedWorker` directly to the tab. 

In cases where the server doesn't set a from field, we just broadcast the message to all tabs using a broadcast channel. All tabs will have a message listener for the `WebSocketChannel` which will handle all message broadcasts. 

This type of a set up can be used in following 2 scenarios:
* Let's say you're playing a game from a tab. You only want the messages to come to this tab. Other tabs won't be needing this information. This is where you can use the first case.
* Now, if you were playing this game on facebook, and got a text message. This information should be broadcasted across all tabs as the notification count in the title would need to be updated. 

## Final Diagrammatic Representation
We've used SharedWorkers to optimize our use of Web Sockets. Here is the final diagrammatic representation of how this can be used:

![image](https://user-images.githubusercontent.com/7992943/68090110-0e4d3080-fe96-11e9-9cf1-6ba0b4914c00.png)

### Note
This is just an experiment I wanted to try to share the same socket connection across multiple browsing contexts. I think this can help reduce the number of connections needed per client. There are still a lot of rough edges around this. Let me know what you think about this solution to a scaling problem with realtime application. Repository containing the code: [SharedWorker WebSocket example](https://github.com/ayushgp/shared-worker-socket-example).

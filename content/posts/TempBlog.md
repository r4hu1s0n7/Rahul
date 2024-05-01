---
title: "Diving into Sockets"
date: "2022-04-12"
draft: false
---
This discussion is based on  learning about sockets from scratch to trying to build an application.

Basic Google definition is: *A socket is a communication endpoint in a computer network that allows two computers to communicate with each other.*

So, on a daily basis, for simplicity, we use WebSockets, or more accurately, we use a library for WebSockets. The WebSocket protocol, which is a browser-focused extension of sockets, uses TCP sockets behind the scenes.

There are also UDP sockets but we will learn about TCP ones.

### TCP Sockets

TCP Socket or Raw Socket, or maybe vanilla socket, has one job: to transfer data from one endpoint to another, that's it. Nothing like maintaining a buffer or so, because this just uses the basic protocol and provides us the most core thing, which is a bridge or pathway of communication.

We just get the bytes; their handling and parsing also need to be taken care of separately by users, which could be some high-level code or anything else.

It just gets the job done at the lowest possible level, like sending and receiving data to a specific address. Since the core functionality is provided, Now we can add our own wrappers around it and make it robust and reliable.

#### Connections

Before a client can connect to a server, the server must first bind to address and listen on a port to open it up for connections.

The Network can handle multiple connections on the same port. So how does it identify a device uniquely? Well, with a pair of source IP and port, and destination IP and port, we can uniquely identify a connection. So my server listening on port 80 could be listening to multiple clients.

(10.20.30.40:80) -> (100.0.0.0:80) and (50.60.70.80:80) -> (100.0.0.0:80)
are 2 different clients, pointing to same server ip. this would make 2 connections

### Web Sockets

So we have established communication, and now we're adding more functionality. We have the WebSocket protocol now. We can refer to its implementation in the WebSocket() API that the browser provides.

```
    var ws = new WebSocket('wss://example.com/socket');

    ws.onerror = function (error) { ... }
    ws.onclose = function () { ... }

    ws.onopen = function () {
    ws.send("Connection established. Hello server!");
    }

    ws.onmessage = function(msg) {
    if(msg.data instanceof Blob) {
        processBlob(msg.data);
    } else {
        processText(msg.data);
    }
}
```

The WebSocket() API communication consists of messages, and the application code and user do not need to worry about buffering, parsing, and reconstructing received data. For example, if the server sends a 1 MB payload, the application's onmessage callback will be called only when the entire message is available on the client.

TCP Socket, depending on payload limit, once the entire data is received, alerts this is callback and it is executed on higher level. To call the rest of application code.

#### Steps involved in communication
1. It starts with a three-way handshake using TCP: SYN → SYN/ACK → ACK. We can read more about this in the Cloudflare [blog on SYN and ACK queues](https://blog.cloudflare.com/syn-packet-handling-in-the-wild). When you go in-depth, and analyse the message passing, we can see the source and destination IP switch in each packet of this handshake.

2. Once the connection is established, we send the HTTP Request, which WebSocket uses to upgrade the connection from HTTP to WebSocket:
   - Client sends: `Connection: Upgrade` request
   
3. The server responds with an acknowledgment of switching protocols.

4. Now the client sends a message, and the server acknowledges (ACKs) it. The same happens if the server sends something; the client acknowledges (ACKs) it. This is where the whole communication happens and this is repeated on every message passed. It is not in sequence like Request-Response cycle, rather anyone can initiate message and other has to ACK.

5. Finally, the close is initiated or timedout.

Now if we need additional headers like supported protocols, there are no default headers here, so the client and server would need to do some custom work around it.

This whole setup is still limited, we need a library over it to actually make it useful in our application, otherwise our application has to handle lots of cases.


## Building WebSocket Application

Okay, now let's proceed towards building an actual app using WebSocket, and we will be learning SignalR, which is a WebSocket library.

Why a library? Well, it's cool because it provides features like reconnection, grouping or channels broadcasts, and maintaining connection IDs.

The SignalR Hubs makes it feel like remote procedure calls (RPCs) from a server to connected clients and from clients to the server. In server side code, you define methods that can be called by clients, and you call methods that run on the client. In client code, you define methods that can be called from the server, and call them. SignalR takes care of all the client-to-server plumbing for us.

We just need to create a class with set of methods that will be called by client and inherit the Hub class, and that will make server side Hub, which then client will invoke.

We will be working on a small Stock application and creating the backend for it.

There would be a Stock class and a StockManager class. The StockManager class has all the stock-related operations like adding stock, updating price, and maintaining a list of stocks.The Stock class only represents a single stock, and it only has the necessary required parameters.

[github link](https://github.com/r4hu1s0n7/signalr-sockets-server) refer for code

```
public class Stock
    {
        public int Price { get; set; }
        public string Name { get; set; }

        public Stock(string name, int price)
        {
            this.Name = name;
            this.Price = price;
        }
    }
```

```
public class StockManager
{
    private static StockManager _instance;
    List<Stock> stocks = new List<Stock>();
    Dictionary<string,List<int>> priceHistory = new Dictionary<string,List<int>>();

    public static StockManager Instance(){}
    public void AddStock(string name, int price){}
    public List<Stock> GetAllStocks(){}
    public void UpdateStockPrices(){}
    public List<int> GetStockPriceHistory(string name){}
    private int UpdatePrice(int price){}
    
}
```

Now we will add Hubs and Controller.

```
public interface SocketHubInterface
{

    Task GetLiveUpdatesAll(List<Stock> stocks);
    Task BucketAddAck(string message);
    Task BroadcastStockPriceHistory(Dictionary<string, List<int>> stockPriceHistory);

}
```

So first, I have added an interface that would define the methods that we will be using. To have strong type checking among RPCs, we have added this interface. So all the RPCs we will be defining in our Hubs class need to be declared here.

Then, we will create a SocketHub class and inherit from the Hub class. This is from the SignalR library, and every Hub class has to inherit it.

```
public class SocketsHub : Hub<SocketHubInterface>
{
	public async Task GetLiveUpdatesAll(List<Stock> stocks)
	{
	     Clients.All.GetLiveUpdatesAll(stocks);
	}
}
```

So this function receives a list of type Stock, and it basically broadcasts to all the connected clients, that are susbcried to “GetLiveUpdatesAll(stocks)”. How do you subscribe? Well client side code has to add listener on this method.

(client subscriber)[https://learn.microsoft.com/en-us/aspnet/core/signalr/hubs?view=aspnetcore-8.0#typescript-client]

same when client has to call any method on server side, they use invoke("server-side-rpc")

```
// several examples
// this will broadcast to all clients that are listening on 'GetLiveUpdatesAll'
Clients.All.GetLiveUpdatesAll(stocks);
// will broadcast to caller, assuming they have implemented this method
Clients.Caller.GetLiveUpdatesAll(stocks);
// when we want to send data to specific client, we can use connectionId provided by Context Property
Clients.Client(Context.ConnectionId).
```

lets further implement our hub class and write other functions we defined in our interface
```
    public class SocketsHub : Hub<SocketHubInterface>
    {
        public async Task GetLiveUpdatesAll(List<Stock> stocks)
        {
            await Clients.All.GetLiveUpdatesAll(stocks);
        }


        public async Task AddMemberToStock(string stock  )
        {
            string connId = Context.ConnectionId;
            await Groups.AddToGroupAsync(connId, stock);
            await Clients.Client(connId).BucketAddAck("Added in Bucket");
        }

        public async Task BroadcastStockPriceHistory(Dictionary<string, List<int>> stockPriceHistory)
        {
            Clients.Group(stockPriceHistory.Keys.First()).BroadcastStockPriceHistory(stockPriceHistory);
            Clients.Groups(Context.ConnectionId);
        }

    }
```
So what I want to do here is I want to create a virtual bucket, that could mean anything like stocks in your portfolio, stocks in your watchlist or any mutual fund, anything.

And any client ID that needs a specific stock to be updated, their subscription will work as being added to that Stock's group, and then a broadcast for that group will be initiated on a timer basis, so this is a simple workflow as of now.

I have not used await in BroadcastStockPriceHistory since I don't want to wait for this execution to finish and then move the code to the next line, that might cause unnecessary delay.

Using await to wait until a client method finishes before the next line of code executes does not mean that clients will actually receive the message before the next line of code executes. "Completion" of a client method call only means that SignalR has done everything necessary to send the message.

We can write a custom workaround for this, but I don't feel it's necessary for it as of now.

Refer to the main repo; it has information about the API controller that allows the execution of these Hubs to be triggered by the API.

##### reference links
https://www.baeldung.com/cs/raw-sockets

https://blog.cloudflare.com/syn-packet-handling-in-the-wild

https://stackoverflow.com/questions/57528914/will-websockets-over-http2-also-be-multiplexed-in-streams 

https://www.keil.com/pack/doc/mw6/Network/html/using_network_sockets_tcp.html

https://www.linkedin.com/posts/interview-ready_how-many-socket-connections-can-one-server-activity-7108811787973619712-chPT?utm_source=share&utm_medium=member_desktop

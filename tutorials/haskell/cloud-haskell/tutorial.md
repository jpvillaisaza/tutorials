---
title: Using Cloud Haskell to write a type-safe distributed chat
published: 2018-04-10
ghc: 8.0.2
lts: 9.17
tags: haskell, distributed-computing, erlang
libraries: distributed-process
language: haskell
author-name: Sebastian Pulido Gomez
github-profile: sebashack
description: In this tutorial we will implement a distributed chat with Erlang’s style and Haskell’s type safety using Cloud Haskell.
---


## What is Cloud Haskell?

[Cloud Haskell](http://haskell-distributed.github.io/documentation.html) is a set of libraries that combines the power of Haskell’s type system with
[Erlang’s](https://www.erlang.org/) style of concurrency and distributed programming. This brings a lot of power in order to implement networking
applications that are fault tolerant, can work on several network transports, and have an expressive model of communication. By expressive we mean
that processes communicate explicitly by sending messages to each other rather than by sharing resources in memory. This also implies that
processes -running in the same local node or in a remote node- pattern match over the specific messages they can handle, which fits very well with
Haskell’s capabilities of modeling messages with algebraic data types.


## Why Cloud Haskell?

Programming concurrent and distributed applications is way too hard. On top of the several challenges that you face when designing software, you
will also struggle with race conditions, deadlocks, bad communication protocols, network problems that are hard to detect and to recover from, and
code that is hard to debug and to maintain. That’s why Erlang was invented: Erlang is a language for manipulating distributed systems that focuses
on recovery from failure.

Additionally, Erlang brought the possibility to write distributed programs in a functional style. All this is pretty
interesting. However, it still lacks type-level guarantees since Erlang is a dynamically typed language, and we cannot model our concurrent
programs as type-safe and predictable communication protocols. Cloud Haskell fills this gap by providing Erlang’s powerful distributed model
shielded by Haskell’s powerful type system, so you can write your distributed programs with the robustness of Haskell and the error recovery from
Erlang.


## Overview

As an overview, let’s see how Cloud Haskell makes use of Erlang’s model by analyzing a very simple example.  First, Cloud Haskell’s most
fundamental entity is a process. *Processes* are isolated and lightweight threads of execution which run in a node, and the only way they can
interact is by passing messages between each other. This is why processes are highly isolated since they do not share resources, which is the main
cause of deadlocks and race conditions in distributed and concurrent systems. Keeping this in mind, sending a message to a process is as easy as
creating a node for the process to reside in and sending a message to it with its unique *process-id*:


```Haskell
import Network.Transport.TCP (createTransport, defaultTCPParameters)
import Control.Distributed.Process
import Control.Distributed.Process.Node


main :: IO ()
main = do
  Right transport <- createTransport "127.0.0.1" "4001" defaultTCPParameters
  node <- newLocalNode transport initRemoteTable
  _ <- runProcess node $ do
    -- get the id of this process
    self <- getSelfPid
    send self "Talking to myself"
    message <- expect :: Process String
    liftIO $ putStrLn message
  return ()
```

Even though the example above is not very interesting, it exposes some cool concepts about this platform: on the one hand, the process logic is
decoupled from the network layer so that you can inject any transport backend you like. In our example, we are using [createTransport](http://hackage.haskell.org/package/network-transport-tcp-0.6.0/docs/Network-Transport-TCP.html#v:createTransport) to create a TCP/IP transport layer, and then we pass it to [newLocalNode](https://hackage.haskell.org/package/distributed-process-0.7.3/docs/Control-Distributed-Process-Node.html#t:LocalNode) to create the node where our process will reside. Nevertheless, you could create a node with a different protocol like SSH, UDP, Unix Sockets, etc. On the other hand, as long as you have the unique identifier for a process, you can send a message to it. Thus, we can use [getSelfPid](https://hackage.haskell.org/package/distributed-process-0.7.3/docs/Control-Distributed-Process.html#v:getSelfPid) to get the id of our process and then make it [send](https://hackage.haskell.org/package/distributed-process-0.7.3/docs/Control-Distributed-Process.html#v:send) the message "Talking to myself" to itself. After that, the process [expects](https://hackage.haskell.org/package/distributed-process-0.7.3/docs/Control-Distributed-Process.html#v:expect) a message of type String which is queued in its mailbox. If no message of the expected type is in the mailbox, then the process will block
until one arrives. In this case, the process is immediately receiving the "Talking to myself" string.

In this tutorial we will approach more interesting and advanced concepts of Cloud Haskell by implementing a simple chat server and client which supports the following features:

  * Launching a chat server room in a specific endpoint address that can be found by chat clients.
  * Launching chat clients that can search a chat server room in a specific endpoint address and connect to it.
  * Command line interface for writing messages in the chat room.
  * Broadcasting messages to all clients who are connected to a chat server room.
  * Handling the disconnection of clients from a chat server room.


## First steps: The chat types

As mentioned above, one of the goals of Cloud Haskell is to set up an expressive messaging model between processes. By expressive we mean that we
can specify which messages we want our process to handle, similar to specifying a communication protocol. This is achieved by pattern matching
over the messages a process must handle and specifying some policy for unhandled messages which do not match any of the handlers. Thus, the first
step in our implementation consists of defining the data types that will be signaled between our chat server and clients.

Initially, our chat server will have to handle two types of messages coming from clients, namely, 1) a message that lets a new client join the
chat and 2) messages to be broadcasted to all the clients on the chat.

The first of these messages is straightforward:


```Haskell
{-# LANGUAGE DeriveDataTypeable     #-}
{-# LANGUAGE DeriveGeneric          #-}

module Types where

import GHC.Generics
import Data.Binary
import Data.Typeable.Internal
import Data.Map (Map)
import Control.Distributed.Process (SendPort)

type NickName = String

newtype JoinChatMessage = JoinChatMessage {
    clientName :: NickName
  } deriving (Generic, Typeable, Show)

instance Binary JoinChatMessage
```

Whenever a client wants to join a chat server, it provides a unique nickname that identifies him/her. Note that we are using Generics to derive instances for [Binary](https://hackage.haskell.org/package/binary-0.9.0.0/docs/Data-Binary.html) and [Typeable](https://hackage.haskell.org/package/base-4.10.1.0/docs/Data-Typeable.html). This is necessary so that our type can automatically be an instance of [Serializable](https://hackage.haskell.org/package/distributed-process-0.7.3/docs/Control-Distributed-Process-Serializable.html#t:Serializable) which is about “objects that can be sent across the network” or, in other words, objects that can be encoded to raw bytestrings and decoded again to their original form.

Our second type represents any message which is broadcasted to the clients connected to a chat:


```Haskell
data Sender = Server | Client NickName
  deriving (Generic, Typeable, Eq, Show)

data ChatMessage = ChatMessage {
    from :: Sender
  , message :: String
  } deriving (Generic, Typeable, Show)


instance Binary ChatMessage
```

Note that we capture the fact that the sender of that message can be either a client or the server itself -For example, when a client connects to
the chat, the server broadcasts a message to the other clients announcing that a new member has joined.

Finally, with Cloud Haskell we can define processes which can update their state after handling a message. Thus, we can define the type of the state
that the chat server process will update after a client joins:

```Haskell
...

import Control.Distributed.Process (SendPort)

...

type ClientPortMap = Map NickName (SendPort ChatMessage)
```

The state of our chat server process consists of a map from a client’s nickname (or identifier) to a send port. We’ll give more details about the
*SendPort* data type when we talk about channels. Meanwhile, we can think about this type as an inventory of the clients that join the chat and a
port through which we can send messages to them.



## The server


As we mentioned above, the most fundamental entity in cloud Haskell is a process and that is why we can naturally define our chat server as a process. Nevertheless, in order to define server processes we can take advantage of the [ProcessDefinition](https://hackage.haskell.org/package/distributed-process-client-server-0.1.3.1/candidate/docs/Control-Distributed-Process-ManagedProcess.html#t:ProcessDefinition) data type defined in the [distributed-process-client-server](https://hackage.haskell.org/package/distributed-process-client-server-0.1.3.1) package. A *ProcessDefinition* has several components which determine how different kinds of messages must be handled,  but we will be focusing only on the two used in our chat server’s definition:


```Haskell
launchChatServer :: Process ProcessId
launchChatServer =
  let server = defaultProcess {
          apiHandlers =  [ handleRpcChan joinChatHandler
                         , handleCast messageHandler
                         ]
        , infoHandlers = [ handleInfo disconnectHandler ]
        , unhandledMessagePolicy = Log
        }
  in spawnLocal $ serve () (const (return $ InitOk M.empty Infinity)) server
```

Thus, a server process is basically a definition which specifies different kinds of handlers that match different types of messages. That’s why we
have several kinds of handler lists in the *ProcessDefinition*, each one containing dispatchers that try to match a message queued in the process
mailbox.

You can notice that our server process has two kinds of handlers, namely:

  *  *apiHandlers* which are in charge of handling the core messages of the application, that is, messages from clients who want to join the chat,
     and messages which have to be broadcast to all the clients connected to the server.
  *  *infoHandlers* which are useful for handling messages that clients are not explicitly sending to the server (e.g. when a client disconnects) and
     that have extra information about the *SendPort* which must be deregistered when a client disconnects.

Finally, we are specifying an *unhandledMessagePolicy* which makes the server log any of the messages which match none of the handlers defined above.

In order to have a better understanding of how our chat server will handle messages coming from the clients, let’s analyze the implementation of
the handlers referenced in our process definition.


## The server’s api handlers

Our chat server has a state represented by the *ClientPortMap* type which may be updated by a handler whenever this matches a specific message.
Thus, besides matching specific messages, handlers also get access to the current state of the server which they can update according to the flow
of the application. In our case, one of the handlers which must update the state of the application is the one in charge of registering the clients
who connect to the chat server:


```Haskell
joinChatHandler :: ChannelHandler ClientPortMap JoinChatMessage ChatMessage
joinChatHandler sendPort = handler
  where
    handler :: ActionHandler ClientPortMap JoinChatMessage
    handler clients JoinChatMessage{..} =
      if clientName `M.member` clients
      then replyChan sendPort (ChatMessage Server "Nickname already in use ... ") >> continue clients
        else do
          void $ monitorPort sendPort
          let clients' = M.insert clientName sendPort clients
              msg = clientName ++ " has joined the chat ..."
          logStr msg
          broadcastMessage clients $ ChatMessage Server msg
          continue clients'
```

*ChannelHandler* is a type synonym with the following definition:


``` Haskell
type ChannelHandler state msg1 msg2 = SendPort msg2 -> (state -> msg1 -> Action state)
```

Which instantiated to the specific type parameters of the *joinChartHandler* definition, it would be:

  * *state*: The state of the server which is of type *ClientPortMap*
  * *msg2*: The type of message which can be sent through the SendPort, namely, a *ChatMessage*.
  * *msg1*: The type of message our handler is expecting from a client and that will be matched in the process' mailbox, that is, *JoinChatMessage*.

This definition expresses that it handles a [channel](https://hackage.haskell.org/package/distributed-process-client-server-0.2.3/docs/Control-Distributed-Process-ManagedProcess.html#t:ChannelHandler) by having as argument the [SendPort](https://hackage.haskell.org/package/distributed-process-0.6.6/docs/Control-Distributed-Process-Internal-Types.html#t:SendPort) of the chat client that is communicating to the server. This handler only matches messages of type *JoinChatMessage* and it replies to the clients with a message of type *ChatMessage*. A *SendPort* is one end of a tuple of communication ports whose other end is a [ReceivePort](https://hackage.haskell.org/package/distributed-process-0.6.6/docs/Control-Distributed-Process-Internal-Types.html#t:ReceivePort). Together they compose an abstraction  named channel which is useful for communicating two processes in a type-safe fashion. For example, in our handler, our chat server can only send messages of type ChatMessage to our clients through a port of type `SendPort ChatMessage` while the clients can only accept messages from this handler through a port of type `ReceivePort ChatMessage`.

With the concept of channel in mind, it is now clear that our handler basically does two things: if the client that is attempting to join the chat
server wants to use a nickname that has already been taken by another client, the server will reply through its specific *SendPort* with a message
notifying that the nickname is already in use ("Nickname already in use ..."). On the other hand, if the nickname is available, the server will
broadcast a message to the currently connected clients notifying that a new user has joined the chat. The definition of the broadcast function is
the following:

```Haskell
broadcastMessage :: ClientPortMap -> ChatMessage -> Process ()
broadcastMessage clientPorts msg =
  forM_ clientPorts (flip replyChan msg)
```

Which simply iterates over the *ReceivePorts* of the clients stored in the current server's state sending a *ChatMessage* with the [replyChan](https://hackage.haskell.org/package/distributed-process-client-server-0.2.3/docs/Control-Distributed-Process-ManagedProcess-Server.html#v:replyChan) function.

Finally, notice how this handler updates the state of the server: in case a new client joins the chat, our process will [continue](https://hackage.haskell.org/package/distributed-process-client-server-0.2.3/docs/Control-Distributed-Process-ManagedProcess-Server.html#v:continue) its execution
with a ClientPortMap that includes the new `(nickname, port)` pair, that is, it is performing an [Action](https://hackage.haskell.org/package/distributed-process-client-server-0.2.3/docs/Control-Distributed-Process-ManagedProcess-Internal-Types.html#t:Action) that updates the state of the server process.

The *messageHandler*, in charge of broadcasting the messages in the chat room among all the clients, is even simpler:


```Haskell
messageHandler :: CastHandler ClientPortMap ChatMessage
messageHandler = handler
  where
    handler :: ActionHandler ClientPortMap ChatMessage
    handler clients msg = do
      broadcastMessage clients msg
      continue clients
```

It only matches messages of type *ChatMessage* and broadcasts them to the other clients by using the *broadcastMessage* function defined above.
Notice that here our process continues its execution without updating the state of the server.


## The server's info handlers

Perhaps in the definition of the *joinChatHandler* you noticed the following mysterious line of code:


```Haskell
void $ monitorPort sendPort
```

This means that we are attaching a *monitoring process* to every client that is connecting to the chat server. In other words, we want our server to
receive a signal whenever a client disconnects (for, example, by typing `ctrl + c` in the terminal). In this way, we can define a handler to match
those signals and take an appropriate action on client disconnection:


```Haskell
disconnectHandler :: ActionHandler ClientPortMap PortMonitorNotification
disconnectHandler clients (PortMonitorNotification _ spId reason) = do
  let search = M.filter (\v -> sendPortId v == spId) clients
  case (null search, reason) of
    (False, DiedDisconnect)-> do
      let (clientName, _) = M.elemAt 0 search
          clients' = M.delete clientName clients
      broadcastMessage clients' (ChatMessage Server $ clientName ++ " has left the chat ... ")
      continue clients'
    _ -> continue clients
```

Whenever a client disconnects it sends a message of type *PortMonitorNotification* which carries the id of its *SendPort*. This way, we can perform a
search to find which client disconnected, which is used to notify the remaining clients, and finally the server’s process continues execution with
a new state which does not include the `(nickname, port)` pair of the disconnected client.

Next, let’s see how to implement the client that will interact with our server!


## The chat client

Implementing a client that can connect to a specific chat-server is even easier. Basically we have to do 5 things:

  * Create a node for our client process to reside.
  * Search our remote chat server and get its [ProcessId](http://hackage.haskell.org/package/distributed-process-0.7.3/docs/Control-Distributed-Process-Internal-Types.html#t:ProcessId).
  * Send a message to the remote chat server signaling that we want to join the chat.
  * Fork a separate process to log the messages coming from other clients connected to the server.
  * Fork a separate process that constantly waits for user input and that broadcasts it as a message to the other clients connected to the remote chat server.

```Haskell
searchChatServer :: ChatName -> ServerAddress -> Process ProcessId
searchChatServer name serverAddr = do
  let addr = EndPointAddress (BS.pack serverAddr)
      srvId = NodeId addr
  whereisRemoteAsync srvId name
  reply <- expectTimeout 1000
  case reply of
    Just (WhereIsReply _ (Just sid)) -> return sid
    _ -> searchChatServer name serverAddr
```

The heart of this definition is the [whereisRemoteAsync](http://hackage.haskell.org/package/distributed-process-0.7.3/docs/Control-Distributed-Process-Internal-Primitives.html#v:whereisRemoteAsync) function provided by the *distributed-process* package. It asynchronously queries a remote node for a process that is in its local registry. That is why we pass to it the id of the remote node ([NodeId](http://hackage.haskell.org/package/distributed-process-0.7.3/docs/Control-Distributed-Process-Internal-Types.html#t:NodeId)) and the name of the chat server room. Notice that we recursively invoke this function until we get a [WhereIsReply](http://hackage.haskell.org/package/distributed-process-0.7.3/docs/Control-Distributed-Process-Internal-Types.html#t:WhereIsReply) with the id of the remote process belonging to the chat server.

With this in mind we can cover almost all the implementation:


```Haskell
...

node <- newLocalNode transport initRemoteTable
      runChatLogger node
      runProcess node $ do
        serverPid <- searchChatServer name serverAddr
        link serverPid
        logStr "Joining chat server ... "
        logStr "Please, provide your nickname ... "
        nickName <- liftIO getLine

...

```

First, we are creating the node to host our client and run its process. Then, after getting the server’s id we have done some extra things for linking our client
to the remote server, and logging a string in console indicating  that you are joining that chat and must provide your nickname in the standard input. [Linking](http://hackage.haskell.org/package/distributed-process-0.7.3/docs/Control-Distributed-Process.html#v:link) our client means that if the remote server unexpectedly dies, our process will be killed.

The third step is straightforward:

```Haskell
...

rp <- callChan serverPid (JoinChatMessage nickName) :: Process (ReceivePort ChatMessage)
logStr "You have joined the chat ... "

...
```

We are only sending a message to the server of type *JoinChatMessage*  that it will handle with the *joinChatHandler* explained above. This means that the server will
add your client to the *ClientPortMap* defined in the application’s types. Moreover, we are getting the *ReceivePort* for the server which will be useful to get the
messages that are broadcast.

Finally, we need to fork two loops, namely, one for constantly receiving messages from the chat server and another to constantly wait for user input. The first
one is implemented as follows:


```Haskell
...

void $ spawnLocal $ forever $ do
  msg <- receiveChan rp
  logChatMessage msg

...
```

This means that we are spawning (forking) another process which will be listening (forever!) to messages coming from the server and it will log those messages to
the standard output. For this we pass the *ReceivePort* from the server to [receiveChan](http://hackage.haskell.org/package/distributed-process-0.7.3/docs/Control-Distributed-Process-Internal-Primitives.html#v:receiveChan) which is a function that waits for messages being sent through a channel.


The second loop has the following implementation:

```Haskell
...

forever $ do
  chatInput <- liftIO getLine
  cast serverPid (ChatMessage (Client nickName) chatInput)

...
```

Which makes use of the standard input to get text from the user that will be cast to the server in the form of a *ChatMessage*. The function [cast](https://hackage.haskell.org/package/distributed-process-client-server-0.2.3/docs/Control-Distributed-Process-ManagedProcess-Client.html#v:cast) allows us to send a message to a
remote process without expecting a reply from it, which suits the kind of messages that our chat server dispatches with the *messageHandler* explained above.

And ... that’s it! We have covered all the tasks the chat client needs to perform. The full code looks like this:


```Haskell
launchChatClient :: ServerAddress -> Host -> Int -> ChatName -> IO ()
launchChatClient serverAddr clientHost port name  = do
  mt <- createTransport clientHost (show port) defaultTCPParameters
  case mt of
    Left err -> putStrLn (show err)
    Right transport -> do
      node <- newLocalNode transport initRemoteTable
      runChatLogger node
      runProcess node $ do
        serverPid <- searchChatServer name serverAddr
        link serverPid
        logStr "Joining chat server ... "
        logStr "Please, provide your nickname ... "
        nickName <- liftIO getLine
        rp <- callChan serverPid (JoinChatMessage nickName) :: Process (ReceivePort ChatMessage)
        logStr "You have joined the chat ... "
        void $ spawnLocal $ forever $ do
          msg <- receiveChan rp
          logChatMessage msg
        forever $ do
          chatInput <- liftIO getLine
          cast serverPid (ChatMessage (Client nickName) chatInput)
          liftIO $ threadDelay 500000
```

## Final remarks

You can check the repository for this tutorial’s source code [here](https://github.com/stackbuilders/cloud-haskell-chat) together with a *README* that explains how to launch both the chat server and the client so that you can try it out. Now that you have learned the essentials of cloud Haskell, do not hesitate to fork this repo and add your experiments or further refinements!

In case you need additional documents or readings you can visit the following links:

  * [Get-started tutorial for cloud haskell.](http://haskell-distributed.github.io/tutorials/1ch.html)
  * [Distributed process library.](https://hackage.haskell.org/package/distributed-process)
  * [Implementing a chat with the low level Network.Socket library.](https://wiki.haskell.org/Implement_a_chat_server)
  * [Some basic Erlang concepts.](http://learnyousomeerlang.com/what-is-otp)

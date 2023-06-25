---
title: "RESP protocol spec"
linkTitle: "Protocol spec"
weight: 4
description: Redis serialization protocol (RESP) specification  Redis序列化协议规范
aliases:
    - /topics/protocol
---

Redis clients use a protocol called **RESP** (REdis Serialization Protocol) to communicate with the Redis server. 
While the protocol was designed specifically for Redis, it can be used for other client-server software projects.
Redis客户端使用名为RESP（Redis序列化协议）的协议与Redis服务器通信。
虽然本协议是专门为Redis设计的，但它可以用于其他客户端-服务器的软件项目。

RESP is a compromise between the following things:
RESP是以下内容之间的折衷方案：

* Simple to implement. 易于实现
* Fast to parse. 快速解析
* Human readable. 可读性强

RESP can serialize different data types like integers, strings, and arrays. There is also a specific type for errors. 
Requests are sent from the client to the Redis server as arrays of strings that represent the arguments of the command to execute. 
Redis replies with a command-specific data type.
RESP可以序列化不同的数据类型，如整数、字符串和数组。还有一种特定类型的错误。
请求以字符串数组的形式从客户端发送到Redis服务器，这些字符串表示要执行的命令的参数。
Redis使用特定于命令的数据类型进行回复。

RESP is binary-safe and does not require processing of bulk data transferred from one process to another 
because it uses prefixed-length to transfer bulk data.
RESP是二进制安全的，它使用前缀长度来传输批量数据。

Note: the protocol outlined here is only used for client-server communication. 
Redis Cluster uses a different binary protocol in order to exchange messages between nodes.
注意：本协议仅用于客户端-服务器通信。

## Network layer  网络层

A client connects to a Redis server by creating a TCP connection to the port 6379.
客户端通过创建TCP连接来连接到Redis服务器。

While RESP is technically non-TCP specific, the protocol is only used with TCP connections 
(or equivalent stream-oriented connections like Unix sockets) in the context of Redis.
虽然RESP在技术上是非TCP特定的，但在Redis的上下文中，本协议仅与TCP连接（或类似Unix套接字的面向流的等效连接）一起使用。

## Request-Response model  请求-响应模型

Redis accepts commands composed of different arguments.
Once a command is received, it is processed and a reply is sent back to the client.
Redis接收由不同参数组成的命令。一旦接收到命令，就会对其进行处理，并将回复发送回客户端。

This is the simplest model possible; however, there are two exceptions:
这是最简单的模型；但是，有两个例外：

* Redis supports pipelining (covered later in this document). 
  So it is possible for clients to send multiple commands at once and wait for replies later.
  Redis支持流水线（本文稍后将介绍）。因此，客户端可以同时发送多个命令，然后等待稍后的回复。
* When a Redis client subscribes to a Pub/Sub channel, the protocol changes semantics and becomes a *push* protocol. 
  The client no longer requires sending commands because the server will automatically send new messages to the client 
  (for the channels the client is subscribed to) as soon as they are received.
  当Redis客户端订阅Pub/Sub通道时，该协议会更改语义并成为推送协议。
  客户端不再需要发送命令，因为服务器将在收到新消息后立即自动向客户端发送新消息（针对客户端订阅的频道）。

Excluding these two exceptions, the Redis protocol is a simple request-response protocol.
排除这两个例外，Redis协议是一个简单的请求-响应协议。

## RESP protocol description  RESP协议描述

The RESP protocol was introduced in Redis 1.2, but it became the
standard way for talking with the Redis server in Redis 2.0.
This is the protocol you should implement in your Redis client.
RESP协议在Redis 1.2中引入，但它在Redis 2.0中成为与Redis服务器进行通信的标准方式。
这是您应该在Redis客户端中实现的协议。

RESP is actually a serialization protocol that supports the following
data types: Simple Strings, Errors, Integers, Bulk Strings, and Arrays.
RESP实际上是一个支持以下数据类型的序列化协议：简单字符串、错误、整数、批量字符串、数组。

Redis uses RESP as a request-response protocol in the
following way:
Redis通过以下方式使用RESP作为请求-响应协议：

* Clients send commands to a Redis server as a RESP Array of Bulk Strings.
  客户端将命令作为批量字符串的RESP数组发送到Redis服务器。
* The server replies with one of the RESP types according to the command implementation.
  服务器根据命令实现使用其中一种RESP类型进行回复。

In RESP, the first byte determines the data type:
在RESP中，第一个字节决定数据类型：

* For **Simple Strings**, the first byte of the reply is "+"  +简单字符串
* For **Errors**, the first byte of the reply is "-"  -错误
* For **Integers**, the first byte of the reply is ":"  :整数
* For **Bulk Strings**, the first byte of the reply is "$"  $批量字符串
* For **Arrays**, the first byte of the reply is "`*`"  `*`数组

RESP can represent a Null value using a special variation of Bulk Strings or Array as specified later.
RESP可以使用稍后指定的批量字符串或数组的特殊变体来表示Null值。

In RESP, different parts of the protocol are always terminated with "\r\n" (CRLF).
在RESP中，协议的不同部分总是"\r\n" (CRLF)换行符结尾。

<a name="simple-string-reply"></a>

## RESP Simple Strings  简单字符串

Simple Strings are encoded as follows: a plus character, followed by a string that cannot contain a CR or LF character (no newlines are allowed), and terminated by CRLF (that is "\r\n").
简单字符串的编码如下：一个加号，后面跟着一个不能包含CR或LF字符的字符串(不允许使用换行符)，并以CRLF ("\r\n")换行符结尾。

Simple Strings are used to transmit non binary-safe strings with minimal overhead. For example, many Redis commands reply with just "OK" on success. 
The RESP Simple String is encoded with the following 5 bytes:
简单字符串用于以最小的开销传输非二进制安全字符串。例如，许多Redis命令在成功时只回复"OK"。
RESP简单字符串由以下5个字节编码：

    "+OK\r\n"

In order to send binary-safe strings, use RESP Bulk Strings instead.
为了发送二进制安全字符串，请改用RESP批量字符串。

When Redis replies with a Simple String, a client library should respond with a string composed of the first character after the '+'
up to the end of the string, excluding the final CRLF bytes.
当Redis使用一个简单字符串回复时，客户端库应该用一个字符串响应，该字符串由'+'之后的第一个字符组成，直到字符串的末尾，不包括最后的CRLF字节。

<a name="error-reply"></a>

## RESP Errors  错误

RESP has a specific data type for errors. They are similar to
RESP Simple Strings, but the first character is a minus '-' character instead
of a plus. The real difference between Simple Strings and Errors in RESP is that clients treat errors
as exceptions, and the string that composes
the Error type is the error message itself.
RESP具有特定的错误数据类型。它们类似于RESP简单字符串，但第一个字符是减号'-'字符，而不是加号。
RESP中的简单字符串和错误之间的真正区别在于，客户端将错误视为异常，而组成错误类型的字符串就是错误消息本身。

The basic format is:
基本格式为：

    "-Error message\r\n"

Error replies are only sent when something goes wrong, for instance if
you try to perform an operation against the wrong data type, or if the command
does not exist. The client should raise an exception when it receives an Error reply.
只有当出现错误时，才会发送错误回复。
例如，如果您试图对错误的数据类型执行操作，或者如果命令不存在。客户端在收到错误回复时应引发异常。

The following are examples of error replies:
以下是错误回复的示例：

    -ERR unknown command 'helloworld'
    -WRONGTYPE Operation against a key holding the wrong kind of value

The first word after the "-", up to the first space or newline, represents
the kind of error returned. This is just a convention used by Redis and is not
part of the RESP Error format.
"-"之后的第一个单词，直到第一个空格或换行符，表示返回的错误类型。
这只是Redis使用的约定，不是RESP错误格式的一部分。

For example, `ERR` is the generic error, while `WRONGTYPE` is a more specific
error that implies that the client tried to perform an operation against the
wrong data type. This is called an **Error Prefix** and is a way to allow
the client to understand the kind of error returned by the server without checking the exact error message.
例如，`ERR`是一般错误，而`WRONGTYPE`是更具体的错误，意味着客户端试图对错误的数据类型执行操作。
这被称为错误前缀，是一种允许客户端在不检查确切错误消息的情况下了解服务器返回的错误类型的方法。

A client implementation may return different types of exceptions for different
errors or provide a generic way to trap errors by directly providing
the error name to the caller as a string.
客户端实现可以针对不同的错误返回不同类型的异常，或者通过将错误名称作为字符串直接提供给调用方来提供捕获错误的通用方法。

However, such a feature should not be considered vital as it is rarely useful, and a limited client implementation may simply return a generic error condition, such as `false`.
然而，这样的功能不应该被认为是至关重要的，因为它很少有用，并且有限的客户端实现可能只是返回一个通用的错误条件，例如`false`。

<a name="integer-reply"></a>

## RESP Integers  整数

This type is just a CRLF-terminated string that represents an integer,
prefixed by a ":" byte. For example, ":0\r\n" and ":1000\r\n" are integer replies.
这种类型只是一个以CRLF结尾的字符串，表示一个以":"字节为前缀的整数。例如":0\r\n"和":1000\r\n"都是整数答复。

Many Redis commands return RESP Integers, like `INCR`, `LLEN`, and `LASTSAVE`.
许多Redis命令返回RESP整数，如`INCR`、`LLEN`和`LASTSAVE`。

There is no special meaning for the returned integer. It is just an
incremental number for `INCR`, a UNIX time for `LASTSAVE`, and so forth. However,
the returned integer is guaranteed to be in the range of a signed 64-bit integer.
返回的整数没有特殊含义。它只是`INCR`的一个增量，`LASTSAVE`的一个UNIX时间，等待。
但是，返回的整数保证在有符号64位整数的范围内。

Integer replies are also used in order to return true or false.
For instance, commands like `EXISTS` or `SISMEMBER` will return 1 for true
and 0 for false.
整数答复也用于返回的true或false。例如，`EXISTS`或`SISMEMBER`等命令将返回1表示true，返回0表示false。

Other commands like `SADD`, `SREM`, and `SETNX` will return 1 if the operation
was actually performed and 0 otherwise.
如果实际执行了操作，则`SADD`、`SREM`、`SETNX`等其他命令将返回1，否则返回0。

The following commands will reply with an integer: `SETNX`, `DEL`,
`EXISTS`, `INCR`, `INCRBY`, `DECR`, `DECRBY`, `DBSIZE`, `LASTSAVE`,
`RENAMENX`, `MOVE`, `LLEN`, `SADD`, `SREM`, `SISMEMBER`, `SCARD`.
以下命令将使用整数进行回复：`SETNX`、`DEL`、
`EXISTS`、`INCR`、`INCRBY`、`DECR`、`DECRBY`、`DBSIZE`、`LASTSAVE`、
`RENAMENX`、`MOVE`、`LLEN`、`SADD`、`SREM`、`SISMEMBER`、`SCARD`。

<a name="nil-reply"></a>
<a name="bulk-string-reply"></a>

## RESP Bulk Strings  批量字符串

Bulk Strings are used in order to represent a single binary-safe
string up to 512 MB in length.
批量字符串用于表示长度高达512 MB的单个二进制安全的字符串。

Bulk Strings are encoded in the following way:
批量字符串以以下方式编码：

* A "$" byte followed by the number of bytes composing the string (a prefixed length), terminated by CRLF.
  一个"$"字节，后面跟着组成字符串的字节数(带前缀的长度)，以CRLF结尾。
* The actual string data.
  实际的字符串数据。
* A final CRLF.
  最后的CRLF。

So the string "hello" is encoded as follows:
因此，字符串"hello"的编码如下：

    "$5\r\nhello\r\n"

An empty string is encoded as:
空字符串编码为：

    "$0\r\n\r\n"

RESP Bulk Strings can also be used in order to signal non-existence of a value
using a special format to represent a Null value. In this
format, the length is -1, and there is no data. Null is represented as:
RESP批量字符串也可以用于使用特殊格式表示Null值的值不存在的信号。
在这种格式中，长度为-1，并且没有数据。Null表示为：

    "$-1\r\n"

This is called a **Null Bulk String**.
这被称为**Null批量字符串**。

The client library API should not return an empty string, but a nil object,
when the server replies with a Null Bulk String.
For example, a Ruby library should return 'nil' while a C library should
return NULL (or set a special flag in the reply object).
当服务器使用Null批量字符串回复时，客户端库API不应返回空字符串，而应返回nil对象。

<a name="array-reply"></a>

## RESP Arrays  数组

Clients send commands to the Redis server using RESP Arrays. Similarly,
certain Redis commands, that return collections of elements to the client,
use RESP Arrays as their replies. An example is the `LRANGE` command that
returns elements of a list.
客户端使用RESP数组向Redis服务器发送命令。类似地，某些向客户端返回元素集合的Redis命令使用RESP数组作为其回复。
一个例子是返回列表元素的`LRANGE`命令。

RESP Arrays are sent using the following format:
RESP数组使用以下格式发送：

* A `*` character as the first byte, followed by the number of elements in the array as a decimal number, followed by CRLF.
  一个`*`字符作为第一个字节，后面跟着数组中的元素数量作为十进制数，后面跟着CRLF。
* An additional RESP type for every element of the Array.
  数组中每个元素的额外RESP类型。

So an empty Array is just the following:
因此，空数组如下所示：

    "*0\r\n"

While an array of two RESP Bulk Strings "hello" and "world" is encoded as:
由两个RESP批量字符串"hello"和"world"组成的数组编码为：

    "*2\r\n$5\r\nhello\r\n$5\r\nworld\r\n"

As you can see after the `*<count>CRLF` part prefixing the array, the other
data types composing the array are just concatenated one after the other.
For example, an Array of three integers is encoded as follows:
正如您在前缀为数组的`*<count>CRLF`部分之后看到的那样，组成数组的其他数据类型只是一个接一个地连接在一起。
例如，一个由三个整数组成的数组编码如下：

    "*3\r\n:1\r\n:2\r\n:3\r\n"

Arrays can contain mixed types, so it's not necessary for the
elements to be of the same type. For instance, a list of four
integers and a bulk string can be encoded as follows:
数组可以包含混合类型，因此元素不必是同一类型。
例如，一个由四个整数组成的列表和一个批量字符串，可以按如下方式编码：

    *5\r\n
    :1\r\n
    :2\r\n
    :3\r\n
    :4\r\n
    $5\r\n
    hello\r\n

(The reply was split into multiple lines for clarity).
(为了清楚起见，答复分为多行)

The first line the server sent is `*5\r\n` in order to specify that five
replies will follow. Then every reply constituting the items of the
Multi Bulk reply are transmitted.
服务器发送的第一行是`*5\r\n`，为了指定后面将有五个答复。然后，传输构成多个批量回复项的每个回复。

Null Arrays exist as well and are an alternative way to
specify a Null value (usually the Null Bulk String is used, but for historical
reasons we have two formats).
Null数组也存在，并且是指定Null值的替代方法(通常使用Null批量字符串，但由于历史原因，我们有两种格式)。

For instance, when the `BLPOP` command times out, it returns a Null Array
that has a count of `-1` as in the following example:
例如，当`BLPOP`命令超时时，它返回一个计数为`-1`的Null数组，如下面示例：

    "*-1\r\n"

A client library API should return a null object and not an empty Array when
Redis replies with a Null Array. This is necessary to distinguish
between an empty list and a different condition (for instance the timeout
condition of the `BLPOP` command).
当Redis使用Null数组回复时，客户端库API应该返回null对象，而不是空数组。
这对于区分空列表和不同条件(例如`BLPOP`命令的超时条件)是必要的。

Nested arrays are possible in RESP. For example a nested array of two arrays
is encoded as follows:
RESP中可以使用嵌套数组。例如，两个数组的嵌套数组的编码如下：

    *2\r\n
    *3\r\n
    :1\r\n
    :2\r\n
    :3\r\n
    *2\r\n
    +Hello\r\n
    -World\r\n

(The format was split into multiple lines to make it easier to read).
(为了便于阅读，将格式拆分为多行)

The above RESP data type encodes a two-element Array consisting of an Array that contains three Integers (1, 2, 3) and an array of a Simple String and an Error.
上述RESP数据类型编码一个两个元素的数组，该数组由一个包含三个整数(1、2、3)的数组和一个简单字符串和一个错误的数组组成。

## Null elements in Arrays  数组中的Null元素

Single elements of an Array may be Null. This is used in Redis replies to signal that these elements are missing and not empty strings. This
can happen with the SORT command when used with the GET _pattern_ option
if the specified key is missing. Example of an Array reply containing a
Null element:

    *3\r\n
    $5\r\n
    hello\r\n
    $-1\r\n
    $5\r\n
    world\r\n

The second element is a Null. The client library should return something
like this:

    ["hello",nil,"world"]

Note that this is not an exception to what was said in the previous sections, but 
an example to further specify the protocol.

## Send commands to a Redis server  向Redis服务器发送命令

Now that you are familiar with the RESP serialization format, you can use it to help write a Redis client library. We can further specify
how the interaction between the client and the server works:
现在您已经熟悉了RESP序列化格式，可以使用它来帮助编写Redis客户端库。
我们可以进一步指定客户端和服务器之间的交互是如何工作的：

* A client sends the Redis server a RESP Array consisting of only Bulk Strings.
  客户端向Redis服务器发送仅由批量字符串组成的RESP数组。
* A Redis server replies to clients, sending any valid RESP data type as a reply.
  Redis服务器回复客户端，发送任何有效的RESP数据类型作为回复。

So for example a typical interaction could be the following.
例如，一个典型的交互可以是如下。

The client sends the command **LLEN mylist** in order to get the length of the list stored at key *mylist*. 
Then the server replies with an Integer reply as in the following example (C: is the client, S: the server).
客户端发送命令**LLEN mylist**，以便获取存储在键*mylist*处的列表的长度。
然后，服务器以整数回复，如下面示例所示(C:是客户端，S:是服务器)。

    C: *2\r\n
    C: $4\r\n
    C: LLEN\r\n
    C: $6\r\n
    C: mylist\r\n

    S: :48293\r\n

As usual, we separate different parts of the protocol with newlines for simplicity, 
but the actual interaction is the client sending `*2\r\n$4\r\nLLEN\r\n$6\r\nmylist\r\n` as a whole.
像往常一样，为了简单起见，我们使用换行符分隔协议的不同部分，但实际的交互是客户端发送`*2\r\n$4\r\nLLEN\r\n$6\r\nmylist\r\n`作为一个整体。

## Multiple commands and pipelining  多个命令和流水线

A client can use the same connection in order to issue multiple commands.
Pipelining is supported so multiple commands can be sent with a single
write operation by the client, without the need to read the server reply
of the previous command before issuing the next one.
All the replies can be read at the end.
客户端可以使用相同的连接来发出多个命令。
支持流水线操作，因此客户端可以通过一次写入操作发送多个命令，而无需在发出下一个命令之前读取上一个命令的服务器回复。
最后可以阅读所有回复。

For more information, see [Pipelining](/topics/pipelining).

## Inline commands  内联命令

Sometimes you may need to send a command
to the Redis server but only have `telnet` available. While the Redis protocol is simple to implement, it is
not ideal to use in interactive sessions, and `redis-cli` may not always be
available. For this reason, Redis also accepts commands in the **inline command** format.

The following is an example of a server/client chat using an inline command
(the server chat starts with S:, the client chat with C:)

    C: PING
    S: +PONG

The following is an example of an inline command that returns an integer:

    C: EXISTS somekey
    S: :0

Basically, you write space-separated arguments in a telnet session.
Since no command starts with `*` that is instead used in the unified request
protocol, Redis is able to detect this condition and parse your command.

## High performance parser for the Redis protocol  Redis协议的高性能解析器

While the Redis protocol is human readable and easy to implement, it can
be implemented with a performance similar to that of a binary protocol.
虽然Redis协议是人类可读的并且易于实现，但它可以以类似于二进制协议的性能来实现。

RESP uses prefixed lengths to transfer bulk data, so there is
never a need to scan the payload for special characters, like with JSON, nor to quote the payload that needs to be sent to the
server.
RESP使用带前缀的长度来传输批量数据，因此永远不需要像JSON那样扫描有效载荷中的特殊字符，也不需要引用需要发送到服务器的有效载荷。

The Bulk and Multi Bulk lengths can be processed with code that performs
a single operation per character while at the same time scanning for the
CR character, like the following C code:
批量和多个批量长度可以使用每个字符执行单个操作的代码进行处理，同时扫描CR字符。

```
#include <stdio.h>

int main(void) {
    unsigned char *p = "$123\r\n";
    int len = 0;

    p++;
    while(*p != '\r') {
        len = (len*10)+(*p - '0');
        p++;
    }

    /* Now p points at '\r', and the len is in bulk_len. */
    printf("%d\n", len);
    return 0;
}
```

After the first CR is identified, it can be skipped along with the following
LF without any processing. Then the bulk data can be read using a single
read operation that does not inspect the payload in any way. Finally,
the remaining CR and LF characters are discarded without any processing.

While comparable in performance to a binary protocol, the Redis protocol is
significantly simpler to implement in most high-level languages,
reducing the number of bugs in client software.

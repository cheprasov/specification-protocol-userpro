SPECIFICATION of UNIVERSAL SERIALIZATION PROTOCOL (USERPRO)
===
The specification describes how clients communicate with some a server using a protocol called **USERPRO** (Universal Serialization Protocol). While the protocol was designed specifically for client-server software projects and microservices.

USERPRO is a compromise between the following things:
* Versatility.
* Stream processing.
* Simple to implement.
* Fast to parse.
* Human readable.
* Support data structure.

While comparable in performance to a binary protocol the protocol is significantly simpler to implement in most very high level languages, reducing the number of bugs in client software.

USERPRO can serialize different data types like integers, strings, arrays, maps, booleans, errors, bulks and other.Requests are sent from the client to the server as any type of data (array, string and other), representing the arguments of the command to execute, based on the server implementation. The server replies with a command-specific structure of data.

USERPRO is binary-safe and does not require processing of bulk data transferred from one process to another, because it uses prefixed-length to transfer bulk data.
Note: the protocol outlined here is only used for client-server communication.

### Networking layer

A client connects to a server creating a TCP connection to some port.
While USERPRO is technically non-TCP specific, in the context of the protocol is only used with TCP connections (or equivalent stream oriented connections like Unix sockets).

The protocol is a simple request-response protocol.

### USERPRO protocol description

USERPRO is actually a serialization protocol that supports the following data types: Simple Strings, Integers, Floats, Booleans, Arrays, Maps, Errors, Bulk Strings and other.

The way USERPRO as a request-response protocol is the following:
* Clients send commands to a server as a USERPRO Data.
* The server replies with one of the USERPRO types according to the command implementation.

In USERPRO, the type of some data depends on the first byte:
* For **Integers** the first byte of the reply is "**i**"
* For **Floats** the first byte of the reply is "**f**"
* For **Booleans** the first byte of the reply is "**b**"
* For **Simple Strings (Lines)** (without newlines symbols "\r" or "\n") the first byte of the reply is "**l**"
* For **Bulk or Multilines Strings** the first byte of the reply is "**s**"
* For **Arrays** the first byte of the reply is "**a**"
* For **Maps** the first byte of the reply is "**m**"
* For **Predefined Constants** the first byte of the reply is "**c**"
* For **Errors** the first byte of the reply is "**e**"

Additionally USERPRO is able to represent a Null, NaN, or Infinity values using a special variation of Predefined Constants as specified later.

In USERPRO different parts of the protocol are always terminated with "\n" (LF).

### USERPRO Integers

This type is just a LF terminated string representing an integer, prefixed by a "**i**" byte. For example "`i0\n`", "`i-33\n`" or "`i42\n`" are integer replies.

### USERPRO Floats

This type is just a LF terminated string representing an float, prefixed by a "**f**" byte. For example "`f0.0\n`", "`f-3.3\n`" or "`f4.2\n`" are float replies.

### USERPRO Booleans

This type is just a LF terminated string representing an booleab, prefixed by a "**b**" byte. For example "`b0\n`" (false), "`b1\n`" (true) are boolean replies.

### USERPRO Simple Strings (Lines)

Simple Strings are encoded in the following way: "**l**" byte, followed by a string that cannot contain a CR or LF character (no newlines are allowed), terminated by LF (that is "\n").

Simple Strings are used to transmit non binary safe strings with minimal overhead. For example some reply with just "OK" on success, that as a USERPRO Simple String is encoded with the following 4 bytes:
"`lOK\n`"

In order to send binary-safe strings, USERPRO Bulk Strings are used instead.

When server replies with a Simple String, a client library should return to the caller a string composed of the first character after the "**l**" up to the end of the string, excluding the final **LF** byte.

### USERPRO Bulk or Multilines Strings

Bulk or Multilines Strings are used in order to represent a single binary safe string.

Bulk Strings are encoded in the following way:
A "**s**" byte followed by the number of bytes composing the string (a prefixed length), terminated by **LF**. Then the actual string data. A final **LF**.

So the string "foobar" is encoded as follows: "`s6\nfoobar\n`"

When an empty string, Bulk Strings are encoded without string data and second **LF**, is just: "`s0\n`"

### USERPRO Arrays

Array is a collection of USERPRO type elements. USERPRO Arrays are sent using the following format:
A "**a**" character as the first byte, followed by the number of elements in the array as a decimal number, followed by **LF**. An additional USERPRO type for every element of the Array.

So an empty Array is just the following: "`a0\n`"

While an array of two USERPRO Bulk Strings "foo" and "bar" is encoded as:
"a2\ns3\nfoo\ns3\nbar\n"
As you can see after the a<count>LF part prefixing the array, the other data types composing the array are just concatenated one after the other. For example an Array of three integers is encoded as follows: "`a3\ni1\ni2\ni3\n`"
Arrays can contain mixed types, it's not necessary for the elements to be of the same type. For instance, a list of 2 integers and a bulk string can be encoded as the follows:
(The reply was split into multiple lines for clarity).
```
a3\n
i10\n
i42\n
s6\n
foobar\n
```
The first line the server sent is `a3\n` in order to specify that 3 replies will follow. Then every reply constituting the items of the Multi Bulk reply are transmitted.
Arrays of arrays are possible in USERPRO. For example an array of two arrays is encoded as follows:
(The format was split into multiple lines to make it easier to read).
```
a2\n
a3\n
i1\n
i2\n
i3\n
a2\n
lFoo\n
lBar\n
```

The above USERPRO data type encodes a two elements Array consisting of an Array that contains three Integers 1, 2, 3 and an array of two Simple Strings.

### USERPRO Maps

Map is a hash table. It has key-value structure. USERPRO Maps are sent using the following format:
A "**m**" character as the first byte, followed by the number of elements in the map as a decimal number, followed by **LF**. An additional USERPRO two types for every element of the map as key and velue.

So an empty map is just the following: "`m0\n`"

Maps can contain mixed types for keys or values.

Example: let's move json user data `{"name": "Alexander","age":33, "city":"London"}` to USERPRO map:
```
m3\n
lname\n
lAlexander\n
lage\n
i33\n
lcity\n
lLondon\n
```

### USERPRO Predefined Constants

This type is just a LF terminated string representing an predefined constant, prefixed by a "**c**" byte.

Predefined Constants are:

| Constant | Description | Encoded as |
|-----------|------------------------------------------------------------|------------|
| Null | The special NULL value represents a variable with no value | `cnull` |
| NaN | The value for Not a numbers | `cnan\n` |
| -Infinity | Negative infinity | `c-inf\n` |
| +Infinity | Positive infinite | `c+inf\n` |


### USERPRO Errors

USERPRO has a specific data type for errors. Actually errors are exactly like USERPRO Bulk or Multilines Strings, but the first character is a minus "**e**" character instead of "**s**". Errors are treated by clients as exceptions, and the string that composes the Error type is the error message itself.
The basic format is: "`e13\nError message\n`"
Error replies are only sent when something wrong happens, for instance if you try to perform an operation against the wrong data type, or if the command does not exist and so forth. An exception should be raised by the library client when an Error Reply is received.

### USERPRO Data structures

...

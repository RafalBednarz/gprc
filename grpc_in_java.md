# Practical guide to gRPC in Java

gRPC is an open-source language-agnostic RPC framework. It was developed at Google in 2015 based on experience 
with Stubby - their own framework used internally to handle billions of requests per second.

## Comparison with SOAP and Rest

This article is based on my experience in using Stubby and gRPC for quite some time. I assume basic 
knowledge about what is gRPC, i.e. how to define a service in .proto file, generate
server and client code in Java and run it.

However if you are new to gRPC you may want to read first: 
 - [gRPC basics in Java tutorial](https://grpc.io/docs/tutorials/basic/java/)
 - [gRPC Spring Boot Starter](https://github.com/yidongnan/grpc-spring-boot-starter) with many examples
 - [our previous blog article](https://blog.j-labs.pl/2019/04/gRPC-over-HTTP2-or-How-I-learned-to-stop-depending-on-REST-and-love-gRPC)

The following table compares gRPC with another most widely used RPC/webservice technologies nowadays:

|               | SOAP           | Rest  | gRPC|
| ------------- |:-------------:|:-----:|:---:|
|Contract| required contract-first or contract-last | not required - may be contract-first or contract-last | required - contract-first|
|Contract format     | WSDL | YAML (most popular)  |  .proto file |
| data transfer format      | XML      | JSON (most popular) | protocol buffers (most popular |

For me, it's more similar to SOAP since you have mandatory contract and you simply define functions
you would like to invoke with a list and type of arguments and return type. However, WSDL contracts
in SOAP were considered quite complex and parsing XML was quite memory and compute intense. 
That's why REST, which is more lightweight, was becoming more and more popular in the recent years.

gRPC combines advantages of both technologies: .proto file format is much simpler than WSDL
and its most widely used data exchange format - protocol buffers - is much smaller and faster than XML.

Apart from that, gRPC makes development in Java quite easy (null-safety and immutability by default) and 
it is quite easy to develop new versions of the contract without breaking its compatibility. This
is very important feature in large distributed systems.

So let's discuss these features in more details!

## Protocol buffers under the hood

You may have heard that the protocol buffers (a.k.a. protobuf) is a method of serializing data. Unlike the other data
 formats like XML, JSON or YAML it is not human readable, yet it is quite important 
to know how it is built.
Each protobuf message consists of series of key-value pairs. Key specifies which message from 
 proto definition we are sending and of what type.

![Image 1](./images/proto_buffers.png)

Let's examine it on a simple example. We have `Car` message definition and we set its id to "44" and 
name to "Tesla":

```
message Car {
  int32 id = 1;
  string name = 2;
}
```

Based on [protobuf encoding guide](https://developers.google.com/protocol-buffers/docs/encoding) 
the protobuf message will look like that:

![Image 2](./images/protobuf_example.png)

We are not particularly interested in the details of values encoding, it's not important for the sake of 
this article. Nonetheless, it is worth to know it was designed to outperform other formats like JSON or XML 
in terms of the network load. If you are interested in the details you can find it in the aforementioned protobuf guide.

## Required, optional and default values

In older version of proto specification, namely *proto2*, you were able to specify if a field is 
*required* or *optional*.
However *required* have been removed from the new proto3 syntax. 
There were long discussions and debates about 
usefulness of *required* keyword; long story short - the argument against *required* is that 
you cannot add or remove required field to not break wire compatibility with previous version of a contract. 
Since all fields became not-required, *optional* keyword was also dropped from proto3.

Does it mean that all fields became optional? It depends what you mean by optional. In the database world we usually 
set not-null constraint if we require value to be present. In Java we also enforce presence of an 
Object by checking if it's not null. This doesn't apply to primitives which are initialized
 automatically with their default values. You can write:
```
int i;
```
which is equivalent to:
```
int i = 0;
```

proto3 fields are similar to Java primitives: they are never null. Looking from this perspective 
they are always "required" - they must have some value.

The bottom line is: you cannot
tell the difference if a field was explicitly set to its default value or not set at all.

Here are the default values for the most popular proto3 types:
- `0`, for numeric fields
- `false`, for boolean
- `""` (empty string), for string

You can check the full list in [proto3 language guide](https://developers.google.com/protocol-buffers/docs/proto3#default)

So is it possible to enforce the user to set a field to something different than the default value?
Not as a part of the contract; you have to validate it manually in the code.

## Proto files backward compatibility

Armed with the knowledge about how the protocol buffers work and the default values, let's take a look at an example.

Assume we have a car rental company which exposes an API in order to be used by the car rental aggregators. As a part of proto3 we 
have the following `Car` message definition:
```
message Car {
  int32 id = 1;
  string name = 2;
  string description = 3;
  Color color = 4;
}

enum Color {
  UNSPECIFIED = 0;
  BLACK = 1;
  PINK = 2;
  RED = 3;
}
```

It is of course oversimplified for the sake of clarity.

Now, we realize that our customers don't care about the color of a car but they are interested
in the number of seats available. We also decided to change *description* property to *additional information* 
(just name of the field, we still store the same information there).

This is our new `Car` message definition:
```
message Car {
  int32 id = 1; 
  string name = 2;
  string additional_information = 3; // (#1) change description to additional_information
  // (#2) remove color
  int32 number_of_seats = 5; (#3) add number_of_seats
  
}
```
The question is: can we deploy this service to production knowing that we have a lot of clients
using the service? Are we risking that clients' services will crash? 
Let's examine these cases one by one:

1. **We changed field's name to `additional_information`.** At first sight it seems like this is breaking
change, isn't it? Well, not if you remember how the protobuf message is sent over the wire. The key by which we 
identify the value is only a field index ("3" in our case, which didn't change) and a wire type (`string`, which also didn't change). So we are safe to do this change. 
(caveat: it will not work when you use Field Masks, I will discuss it later)

2. **Removal of color also does not look safe.** What happens if we don't send the color over the
wire? On the client side, with old version of the proto, it will automatically evaluate to the default value,
which in case of enum is whatever we assign to 0 (`UNDEFINED` in our case). So apart from losing the
 information about the car color this contract change shouldn't require any code changes on the client side.

3. **Addition of another field is also backward compatible.** A client with an old version will just ignore
the message with an unknown index.

As you can see it is quite hard to break proto3 backward compatibility. 
The general rules while changing the proto files definitions are: 

1. **Do not change** the tags for the field messages.

2. **Do not reuse** the tags for the field messages, i.e do not introduce new messages in a place of removed ones.

## Using gRPC in Java
As noted before, gRPC is always contract-first and in each language we get the code generated by the protocol buffer compiler. 
I would like to stress two main features of Java generated code:
- We build proto messages using Builder pattern and they are immutable.
- It's (almost) impossible to get `NullPointerException` while manipulating protobuf objects in Java.

I think the purpose of the first feature is quite clear. It enables the use in a thread-safe manner.
However, what is even better about using the proto messages in Java, is the fact that you don't have to care 
about checking for nulls.

The reason for that should be pretty clear by now. I already mentioned that the default scalar values 
in proto are never null. They are `0` for numeric, `false` for boolean, etc.

However, how about nested messages? [Developers guide](https://developers.google.com/protocol-buffers/docs/proto3#default) 
says "for message fields, the field is not set. Its exact value is language-dependent."
It's a bit unclear and it's hard to find what it really means for Java, so let's try to test it. 

We modify the `Car` message to have some nested message there. Let it be *Engine*:

```
service CarService {
    rpc GetCar (GetCarRequest) returns (Car) {
    }
}

message Car {
    string name = 1;
    Engine engine = 2;
}

message Engine {
    int32 capacity = 1;
    int32 torque = 2;
}

```

On the server side we populate `Car` object like that:

```
Car reply = Car.newBuilder().setName("Tesla").build();
```

Note that we didn't set the `engine` field value at all.

On the client side we deserialize the message:

![Image 3](./images/getEngine.png)


Then we see in the debug mode that the `engine` field is null, so we expect `NullPointerExcception` while invoking the next line:
```
int engineCapacity = response.getEngine().getCapacity();
```
But instead we get `0` assigned as `engineCapacity`. Why is that? Let's have a look what `engine` getter does:

![Image 4](./images/null_Example.png)

Well, we get a new object with the default values created ad-hoc. So instead of writing:
```
Optional.ofNullable(response.getEngine()).map(Engine::getCapacity).orElse(0);
```
We can chain getters without any null checks. How cool is that, right?

And one more thing: in case of messages we can check for its presence using `hasXXX` method, for example:

```
if(response.hasEngine()) {
  System.out.println("This car has an Engine);
}
```
We cannot do it for scalar fields.

## Field masks
So far so good, we no longer care about the infamous `NullPointerException`. However "null" is also 
some kind of information and we don't have the possibility to transmit it via protobuf.
Let's assume that we would like to update resource partially, like in PATCH method of REST. 
Normally you just wouldn't send the values you want to update.
That is not possible with gRPC - even if you don't send the value over the wire it 
will be evaluated to a default value while deserializing. If you have an empty string how do you know
if it means "don't update the value" or "set the value to empty"?

That's why gRPC developers came up with [FieldMasks](https://developers.google.com/protocol-buffers/docs/reference/java/com/google/protobuf/FieldMask) 
which are used to specify a subset of fields 
that should be modified by an update operation (or returned by a get operation). Field Mask is just 
a set of field names (yes, names, not tags, that's why you need to be careful changing the contract when using Field Mask)
which are to be modified.

You could use [FieldMaskUtil](https://developers.google.com/protocol-buffers/docs/reference/java/com/google/protobuf/util/FieldMaskUtil)
class which greatly facilitates working with Field Masks.

## Conclusion
In the article I tried to gather the most common problems and questions you may encounter developing 
and maintaining your service using gRPC in Java.

I hope that it is much clearer now, good luck on your coding journey!

## References
https://developers.google.com/protocol-buffers/docs/encoding

https://grpc.io/

https://stackoverflow.com/questions/31801257/why-required-and-optional-is-removed-in-protocol-buffers-3

https://capnproto.org/faq.html#how-do-i-make-a-field-required-like-in-protocol-buffers

https://github.com/protocolbuffers/protobuf/issues/2497

https://stackoverflow.com/questions/31801257/why-required-and-optional-is-removed-in-protocol-buffers-3

https://capnproto.org/faq.html#how-do-i-make-a-field-required-like-in-protocol-buffers

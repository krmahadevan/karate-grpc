# karate-grpc

[![Build Status](https://api.travis-ci.org/thinkerou/karate-grpc.svg)](https://travis-ci.org/thinkerou/karate-grpc)

karate-grpc made gRPC testing simple by [karate](https://github.com/intuit/karate), and its dynamic client built based on [polyglot](https://github.com/grpc-ecosystem/polyglot).

karate-grpc can get all the benefits of [karate](https://github.com/intuit/karate#features), it makes it really easy to build protobuf complex request payloads via json, traverse data within the responses and chain data from responses into the next request.

## Hello World
<a href="https://github.com/thinkerou/karate-grpc/blob/master/karate-grpc-demo/src/test/java/demo/helloworld/helloworld-new.feature"><img src="assets/karate-grpc-hello-world.png" /></a>

## Testing hello world

Prefer to use Maven:

```
$ # compile the whole project
$ mvn clean compile package -Dmaven.test.skip=true
$ cd karate-grpc-demo
$ # run all tests
$ mvn test
$ # or run single test
$ mvn test -Dtest=HelloWorldNewRunner
```

Because have started hello world server on `TestBase.java`, we not need to start it alone.

## Real World Examples

A set of real-life examples which includes `single rpc`, `client stream rpc`, `server stream rpc` and `bidi stream rpc` can be found here: [karate-grpc-demo](karate-grpc-demo)

## Getting Started

karate-grpc requires Java 8 and then Maven to be installed, these also are required by [karate](https://github.com/intuit/karate) and [polyglot](https://github.com/grpc-ecosystem/polyglot).

> karate-grpc only support Maven currently.

### Maven

You need to add the following `<dependencies>`:

```maven
<dependency>
    <groupId>com.github.thinkerou</groupId>
    <artifactId>karate-grpc-core</artifactId>
    <version>0.4.11</version>
</dependency>
```

### Gradle

> TODO: need to test!!!

Alternatively for Gradle you need to add the following entry:

```gradle
testCompile 'com.github.thinkerou:karate-grpc-core:0.4.11'
```

## What to need for testing grpc server

Testing one grpc server, we have the follow info:

- grpc server `ip` and `port`.

- protobuf file corresponding grpc server, but usually it's protobuf `jar package` not one single file or more files.

So, we could test it based on the two point.

**⚠️ Using karate-grpc we can perfect to solve it!**

## What to need in pom file

For testing your grpc server, as above, need protobuf jar dependency and protobuf build plugins - [`protobuf-maven-plugin`](https://www.xolstice.org/protobuf-maven-plugin/).

**MUST** appoint `descriptorSetFileName` and `protoSourceRoot` params:

```
<descriptorSetFileName>karate-grpc.protobin</descriptorSetFileName>
<protoSourceRoot>${project.build.directory}/dependency/demo</protoSourceRoot>
```

Especially, `descriptorSetFileName` **MUST** equal `karate-grpc.protobin`, please see [here](karate-grpc-core/src/main/java/com/github/thinkerou/karate/constants/DescriptorFile.java) about more details.

And other pom settings are the same as `karate`.

## How to write karate feature

We need to use [Java interop](https://github.com/intuit/karate#java-interop) of Karate in order to call us define grpc client.

And use `JSON.parse` javascript function parse the response of grpc server return value.

So, use `karate-grpc` need the following steps:

1. Calls into karate-grpc GrpcClient via Java Interop.

```
* def Client = Java.type('com.github.thinkerou.karate.GrpcClient')
```

2. Builds one public Grpc client using your grpc ip and port.

```
* def client = Client.create('localhost', 50051)
```

If you want to list protobuf by service name or/and message name, you should use:

```
* def client = Client.create()
```

Because not need grpc server ip/port when listing protobuf.

3. Reads JSON data corresponding your protobuf definition.

4. Calls your Grpc server using `call` of karate-grpc.

```
* def response = client.call('helloworld.Greeter/SayHello', payload)
```

`call` have two params: protobuf full name(`format:<package-name>.<service-name>/<rpc-name>`) and JSON data.

If you input protobuf full name error, `call` will fail and output protobuf message by `list`, like this:

When input `helloworld.Greeter/SayHello1`, it will fail and print log:

```
Oct 10, 2018 5:06:46 PM com.github.thinkerou.karate.service.GrpcCall invoke
警告: Call grpc failed, maybe you should see the follow grpc information.
Oct 10, 2018 5:06:46 PM com.github.thinkerou.karate.service.GrpcList invoke
信息: /var/folders/zp/p4s985f95z75jkk5p88h_khm0000gn/T/karate.grpc.3614949790091757570.list.result
Oct 10, 2018 5:06:46 PM com.github.thinkerou.karate.service.GrpcCall invoke
信息:
helloworld.Greeter => helloworld.proto
 helloworld.Greeter/SayHello

 helloworld.Greeter/AgainSayHello

 helloworld.Greeter/SayHelloBiStreaming

 helloworld.Greeter/SayHelloClientStreaming

 helloworld.Greeter/SayHelloServerStreaming

```

5. Converts response string to JSON.

```
* def response = JSON.parse(response)
```

Because `call` of karate-grpc returns JSON string, we need to convert it and then can use `match` assertion.

6. Asserts payload.

7. (Optional) Saves response for second Grpc.

If have second Grpc use the response of first Grpc, we should save it, like:

```
* def message = response[0].message
```

And use it on JSON file:

```
[
  {
    "message": "#(message)",
    "address": "BeiJing"
  }
]
```

8. (Optional) Second Grpc call using response data before.

One whole example likes [this](karate-grpc-demo/src/test/java/demo/helloworld/helloworld-new.feature):

```
Feature: grpc helloworld example by grpc dynamic client

  Background:
    * def Client = Java.type('com.github.thinkerou.karate.GrpcClient')
    * def client = Client.create('localhost', 50051)

  Scenario: do it
    * def payload = read('helloworld.json')
    * def response = client.call('helloworld.Greeter/SayHello', payload)
    * def response = JSON.parse(response)
    * print response
    * match response[0].message == 'Hello thinkerou'
    * def message = response[0].message

    * def payload = read('again-helloworld.json')
    * def response = client.call('helloworld.Greeter/AgainSayHello', payload)
    * def response = JSON.parse(response)
    * match response[0].details == 'Details Hello thinkerou in BeiJing'
```

### How to write JSON file

Because `karate-grpc` supports stream grpc, we use `list` JSON.

Input JSON file like:

```$xslt
[
  {
    "name": "thinkerou"
  },
  {
    "name": "田欧"
  }
]
```

Output JSON string also like:

```$xslt
[
  {
    "message": "Hello thinkerou"
  },
  {
    "message": "Hello 田欧"
  }
]
```

**That's all!!!**

## How to write grpc client

**Note:**

> - The part content is outdated draft which initially think about the topic which continues to have saved is for reference only.
> - Usually you no need to care it and skip it, because `karate-grpc-core` have completed the function.

You only need two steps:

- Read json file and parse protobuf object to the request of grpc server

- format the response of grpc server to json string and return it as grpc server

Like this:

```java
public class Client {

  // ...

  public static String greet(String input) {
    HelloRequest.Builder requestBuilder = HelloRequest.newBuilder();
    try {
      JsonFormat.parser().merge(input, requestBuilder);
    } catch (ProtocolBufferException e) {
      // ...
    }

    HelloReply response = null;
    try {
      response = blockingStub.sayHello(requestBuilder.build());
    } catch (StatusRuntimeException e) {
      // ...
    }

    String res = "";
    try {
      res = JsonFormat.printer().print(response);
    } catch (ProtocolBufferException e) {
      // ...
    }

    return res;
  }

  // ...
}

```

## Thanks 

Thanks [Peter Thomas](https://github.com/ptrthomas) for his work for [karate](https://github.com/intuit/karate) and his generous help, also thanks [Dino Wernli](https://github.com/dinowernli) for his contributions for [polyglot](https://github.com/grpc-ecosystem/polyglot).

## Reference

Maybe you want to know more information about Karate or other, please read the follow contents:

- karate project home: [https://github.com/intuit/karate](https://github.com/intuit/karate)

- polyglot project home: [https://github.com/grpc-ecosystem/polyglot](https://github.com/grpc-ecosystem/polyglot)

- ProtoBuf(via JSON) project home: [https://github.com/protocolbuffers/protobuf/tree/master/java](https://github.com/protocolbuffers/protobuf/tree/master/java)

- grpc-java project home: [https://github.com/grpc/grpc-java](https://github.com/grpc/grpc-java)

## License

[karate-grpc](https://thinkerou.com/karate-grpc/) is licensed under MIT License.

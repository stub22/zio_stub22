---
id: with-cloud-aws
title: "How to Interop with AWS Cloud?"
sidebar_label: "AWS Cloud"
---
### AWS is a commercial service, not affiliated with ZIO

Please keep in mind that AWS is a commercial service, not affiliated with Ziverge or the ZIO contributor community. Users of AWS may need to pay for any services consumed by their software, which may be amplified by bugs in any component.  The ZIO community takes no responsibility for consequences of using any component of software or data, as explained in the ZIO License. 
 
# Overview of Architectural Options
We divide discussion into 3 main approaches:  1) AWS Client Only, 2) AWS App Containers, 3) AWS Serverless

### 1) AWS Client Only

To run as a mere client of AWS running independently runtimes is relatively easy, and can be a decent approach to providing some cloud functionality to our apps, without taking on AWS deployment of our code.  Client-only is a fast and easy way to unit test app feature code that reads and writes to AWS services.  This same app feature code may then be used in the more ambitious AWS deployments discussed below.  

ZIO-enabled libraries offer functional effect APIs to access AWS resources throught the AWS SDK for Java 2.0.

`zio-aws` provides a set of generated service wrapper libraries for many AWS services, in theory all" of them.

Additional higher level client libraries exist for some services.
  
  * `zio-connect` wraps AWS Dynamodb AND ____,  
  * `zio-dynamodb` also wraps AWS Dynamodb database, with more features.
  * `zio-s3` wraps connections to AWS S3 Storage.
  * `zio-sqs` wraps the AWS Simple Query Service.

These libraries are discussed in more detail below.

### 2) AWS App Containers

If you fully embrace the containerized (docker/other) app deploy+test model for AWS then 
your application can get the full benefit of ZIO 2.0 concurrency while running in the AWS cloud. 
Containerized Scala + ZIO apps may include their own JVMs, may use Scala Native, Graal Native, 
or other approaches.  We briefly list two possible ways to use Scala + ZIO app containers on AWS.

2A) Persistent cloud app containers deployed in AWS Elastic cloud services such as EC2, EBS, EKS.
All general AWS guidance for containerized java applications applies to the case of running Scala ZIO apps.
Containerized ZIO apps may use the same ZIO AWS client features discussed in scenario #1 above.

2B) Serverless cloud apps deployed as containers using AWS tools are in general fully compatible with Scala and ZIO, which from this point of view are simply high level libraries used in a JVM/native application.

`zio-lambda` version _____ using Graal native ____ is crafted to support this approach.  It routes Lambda invocation
data directly from AWS JSON-text streams into serialized Scala values, using `zio-json`.  Introductory container 
deployment instructions are provided. `zio-lambda` offers fast start times, and all the development power of 
Scala + ZIO.   This is very different DX from the non-containerized Section 3 below.

### 3) AWS Serverless Scala using JVM *and/or* Scala.js 
(using AWS provided builtin runtimes for JVM or Node.js, without any docker container)

Scala on JVM(aws-builtin) for Serverless is overall quite viable in its tradeoff between DX, deployment + admin power and ease, with
variable latency depending on the use cases applied.  The most common complaint is aws-builtin-JVM cold-start times, which which can 
be addressed using one of these engineering pathways: 

3A) [Lambda Snapstart](https://aws.amazon.com/blogs/aws/new-accelerate-your-lambda-functions-with-lambda-snapstart/) for Java 11 Coretto, introduced 2022-11-28. 

3B) Careful engineering of the Serverless app : small code size, smart data caching, clever riddles

3C) Scala.js deployment to use Node.js runtime.  In principle the same Scala codebase may be used for deployments to both JVM and Scala.js. For an inspirational explanation, see docs for [Feral] in the Cats-Effect ecosystem.

3D) Eventual migration towards one of the containerized application approaches above.

## Selected Topics in ZIO - AWS interop
The remainder of this document provides some more detailed notes on the use of conventional Scala + ZIO 2.x for JVM with AWS, focusing only on Scenarios 1 and 3 from above:  Pure AWS Clients (running in any Scala + ZIO environment) and direct Serverless Scala-JVM Lambdas for the JVM.  


### ZIO-AWS client libraries

The [zio-aws](https://zio.dev/zio-aws/) project ([github](https://github.com/zio/zio-aws)) provides a large set of .jar libraries, each implementing a thin Scala + ZIO wrapper around one of the underlying AWS Java APIs.  This project uses code generation, and attempts to provide wrappers for _all_ AWS services in the AWS SDK for Java v2.

These shallow wrappers are useful when direct access to AWS API features is desirable for our ZIO application.  This may occur when or to be used as a building block for higher level wrappers around specific services."

These wrappers make the API input + output value objects more friendly to the Scala-ZIO developer, but they do not add much service-specific client functionality.
Therefore the `zio-aws` setup is appropriate when you want relatively direct access to the AWS client functionality, and you are prepared to build up your Scala + ZIO application or library without additional help.

You can see the long list of available libraries here. https://zio.dev/zio-aws/artifacts

The design approach of the wrappers is explained here.  https://zio.dev/zio-aws/wrappers

Additional details on AWS Configuration and network connection options are explained in the other pages of the zio-aws microsite.

#### Service-specific ZIO ecosystem libraries

The ZIO ecosystem contains several higher-level client libraries which provide additional helpful functionality on top of the bare-bones `zio-aws` features.  TODO: Confirm that all of these go through zio-aws layer.
  * [zio-s3](https://zio.dev/zio-s3/)
  * [zio-dynamodb](https://zio.dev/zio-dynamodb/) Provides higher-level features for accessing DynamoDB, including bindiing of a case class to a table schema.  Uses `zio-schema`.
  * [zio-sqs](https://zio.dev/zio-sqs/)
  * [zio-connect](https://zio.dev/zio-connect/)

## Executing ZIO Effects inside AWS Serverless Lambdas

Application Scala code (running inside a Lambda invocation) usually builds up a ZIO effect to be run, which we may then execute unsafely.  
  * [Task, Runtime and Exit](https://zio.dev/reference/#core-data-types) are the core ZIO concepts needed.
  * We may choose to replace "println" with our choice of logging call.

``` 
import zio.{Exit, Task, Runtime => ZRuntime, Unsafe => ZUnsafe, Trace => ZTrace}
def doRunUnsafeTaskToExit[Rslt](task : Task[Rslt]) : Exit[Throwable, Rslt] = {
  
  val zioRuntime: ZRuntime[Any] = ZRuntime.default
  println(s"==== println: UnsafeTaskRunner.doRunUnsafeTaskToExit inputTask=${task}, zioRuntime=${zioRuntime}")
  val xitOut : Exit[Throwable, Rslt] = ZUnsafe.unsafe { implicit unsafeThingy =>
    val apiInst: zioRuntime.UnsafeAPI = zioRuntime.unsafe
    
    apiInst.run(task)
  }
  println(s"==== println: UnsafeTaskRunner.doRunUnsafeTaskToExit END with Exit class=${xitOut.getClass.getName}")
  xitOut
}
```

## Input/Output Values for Lambdas Serverless Apps

### AWS SDK RequestHandler API
When using the `RequestHandler` API, the AWS SDK for Java provides its own JSON De/serialization code, which produces+consumes mutable Java objects.  Using this mechanism comes with tradeoffs.

In this scenario our deployed .jar file (deployed with SAM or other AWS mechanisms) contains Scala JVM code that implements the [`com.amazonaws.services.lambda.runtime.RequestHandler`](https://docs.aws.amazon.com/lambda/latest/dg/java-handler.html) interface. 
```
public interface RequestHandler<I, O> {
    O handleRequest(I input, Context context);
}
```

The `I` and `O` type parameters to this Lambda function may take several different forms.  
#### AWS SDK Builtin Event objects

When a Lambda function is invoked from another AWS service, it will generally pass try to pass in service-specific SDK Event objects that are appropriate, such as `APIGatewayProxyRequestEvent`. 
Note that when integrating with AWS `ApiGateway`, the bodies of HTTP messages passed are available as an unparsed `java.lang.String` in the `.body` field of the Event.  It is possible to use ApiGateway mapping features to change this behavior.

Using zio-aws Wrappers for Events

#### AWS SDK Builtin JSON De/Serialization 
A RequestHandler may accept and returns a `java.util.Map[String, AnyRef]`, which the AWS SDK converts to and from a JSON record, without any help. This form can be useful when we wish to invoke the Lambda function directly from our own client code. The `java.util.Map` represents the top level of the JSON data tree, and its values (the AnyRefs) may contain more Maps and Lists inside it (as well as Strings, Integers, Doubles, Booleans).  If this function is called from a Serverless integration, then the Event data will be copied to/from the `java.util.Map`, which is usually not a great way to proceed.  See the mention of "Builtin Event objects" instead.  

The AWS Lambda-for-Java infrastructure merely copies all of the input JSON data to/from an agnostic Java Map data structure, without any semantic checking of validity.  

### Scala JSON Serialization

AWS Client code may use `zio-json` facilities to encode+decode AWS events.  

It should be possible to use zio-json based JSON decoding mechanism together with the AWS
[`RequestStreamHandler`](https://github.com/aws/aws-lambda-java-libs/blob/main/aws-lambda-java-core/src/main/java/com/amazonaws/services/lambda/runtime/RequestStreamHandler.java) which processes `java.io.Stream`s.

`void handleRequest(InputStream input, OutputStream output, Context context) throws IOException`

### Object Binding 
`zio-schema`

### Network
`zio-netty`

### Logging
`LambdaLogger`
`zio-log`

## Clients for Lambda Functions

Invoking Lambda functions from our client code is similar to accessing other AWS services.

`zio-aws-lambda`



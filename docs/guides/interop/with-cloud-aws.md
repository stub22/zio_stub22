---
id: with-cloud-aws
title: "How to Interop with AWS Cloud?"
sidebar_label: "AWS Cloud"
---

## ZIO on AWS (Amazon Web Services)

Scala and ZIO can be integrated with AWS in numerous ways.
Naturally we rely on the AWS Java APIs which are thoroughly documented by AWS.
The ZIO ecosystem offers several projects to make integration of Scala, ZIO and AWS easier and more productive.  
In this document we give some links, guidelines and tips to help you get started with your ZIO-on-AWS project.  Let's start by outlining the most common integration scenarios.
Two main kinds of integration that ZIO is helpful with are:
1) Connecting your ZIO application as a client of AWS Services.  Some popular services to connect with as a *client* include
   * S3 storage
   * DynamoDB
   * SQS and SNS
   * Lambda, API Gateway, Serverless apps.  Remember, here we are talking about connecting as a *client* of some service you have permission to access on AWS.  That service might use any programming language such as Node.js, Python, Go, Java.  But your client is written in Scala and ZIO, and wants to make calls to this service.
[Link To FF](#Connecting as AWS Client)

2) Running your ZIO-enabled code inside of AWS services
AWS offers numerous ways to deploy and run JVM based code inside the AWS cloud.
In this document we mention two main approaches

    * Running Scala + ZIO code as an AWS Lambda function, in an AWS Serverless Application.
    * Running Scala + ZIO code as a containerized application deployed using Docker and related virtualization tools, hosted by Amazon EC2, EBS, and similar services.

One thing to notice right away is that when we deploy our Scala code inside AWS services, we usually *also* want that code to connect to some useful AWS services for storage and messaging.  Thus we wind up using the capabilities of #1 above as we build our application to deploy+run under #2. Fortunately the AWS infrastructure is very supportive of these needs.  But we must keep these different roles of the AWS infrastructure clear in our minds as we plan out our coding, testing and deployment. For example if Sammy says vaguely "I am trying to connect to Lambda", does he mean he wants to connect a client to invoke an AWS Lambda function, or does he want to deploy a new AWS Lambda function of his own?

### Connecting as AWS Client

Functioning as an AWS client is similar to accessing other network services. 

Before you start using Scala and ZIO together with AWS you will want to become generally familiar with AWS, set up your own account, and get some security credentials established.
You can follow the AWS instructions for creating a Java client of the service you are interested in (e.g. S3 or DynamoDB) to verify that you have set up your security credentials
properly.  Once you have verified this Java client access, you can look into ways to integrate your Scala application with the backend AWS services of interest.  Generally
the same security configuration will apply to your Scala programs, e.g. ~/.aws/credentials works the same way for a Scala client as for a Java client.

#### Direct Access to AWS Java APIs

Since Scala code can call Java code, you can of course call into the AWS libraries directly.
Amazon provides instructions for the [AWS SDK for Java 2.0](https://aws.amazon.com/sdk-for-java/).
Note AWS also supplies a bevy of example Java client programs here: https://github.com/awsdocs/aws-doc-sdk-examples/tree/main/javav2/example_code
Reading this Java code can give you a good sense of the basic usage pattern of the underlying APIs.
Our Scala and ZIO-specific features generally are built on top of these Java APIs.

#### ZIO-AWS client libraries

The `zio-aws` project provides a large set of .jar libraries, each implementing a thin Scala + ZIO wrapper around one of the underlying AWS Java APIs.
https://zio.dev/zio-aws/
https://github.com/zio/zio-aws

These wrappers make the API input+output value objects more friendly to the Scala developer, but they do not add much service-specific client functionality.
Therefore this zio-aws are appropriate when you want relatively direct access to the AWS client functionality, and you are prepared to build up your Scala + ZIO application without any additional help.
You can see the long list of available libraries here. https://zio.dev/zio-aws/artifacts
The design approach of the wrappers is explained here.  https://zio.dev/zio-aws/wrappers
Additional details (AWS Configuration, Network connection options) are explained in the other pages of the microsite.

#### Service-specific client libraries

The ZIO ecosystem contains several higher-level client libraries which provide additional helpful functionality on top of the bare-bones zio-aws features.
  * zio-connect


JSON Serialization

[Link To FF](#from-future)
Object Binding 

Network

Available Libraries

### Building an AWS Serverless Application 

https://docs.aws.amazon.com/lambda/latest/dg/lambda-java.html


To clarify what our code does so far:  We deploy a single .jar file using the regular AWS SAM method for deploying serverless applications (for Java / JVM), 
using the AWS SAM Cli executable. The .jar file we deploy is currently about 45MB, which is acceptable for now. Could be reduced and/or split into "Lambda layers".

Our .jar file contains Scala code that implements the Lambda Java API, by extending `com.amazonaws.services.lambda.runtime.RequestHandler`
which means JSON serialization and deserialization is handled by the AWS runtime for Java.
Our Lambda function accepts and returns a `java.util.Map[String, AnyRef]`
That Map represents the top level of the JSON data tree, and may contain more Maps and Lists inside it (as well as Strings, Integers, Doubles, Booleans).  
The AWS Lambda-for-Java infrastructure merely copies all of the input JSON data into that Java Map data structure, without any semantic checking of validity.  
The data contents may turn out to be semantically invalid, later, but at least the bridge from JSON-to-Java(/Scala) is crossed in a straightforward way 
(with no code from us). From there we are inside the world of typesafe Scala code, with ZIO available, running in AWS serverless infra.

The nice part is that deployment and testing are easy to get started with (assuming you have an AWS setup).  SAM Cli can launch a local Docker instance for integration testing
of our Scala + ZIO function. Then finally SAM Cli can deploy our function .jar to the AWS cloud with all necessary permissions (e.g. to read databases and access other AWS services).

The ZIO-specific part:  Our Scala code (running inside the Lambda invocation) builds up a ZIO effect to be run, which we then execute unsafely like so:

``` 
import zio.{Runtime => ZRuntime, Unsafe => ZUnsafe}
def doRunTaskNow[Rslt](task : Task[Rslt]) : Rslt = {
  val zioRuntime = ZRuntime.default
  ZUnsafe.unsafe { implicit unsafeThingy =>
    zioRuntime.unsafe.run(task).getOrThrowFiberFailure()
  }
} 
```

This approach is in stark contrast to what zio-lambda currently does (although `zio-lambda` is surely better for some use cases!), so that's why 
I think it's worth sharing more introductory info for devs approaching the ZIO-on-AWS Serverless / Lambda space for the first time.
I also think there could be useful middle-ground where some utility code from `zio-lambda` could be applied in different kinds of Lambda deployments.  
For example it should be possible AFAIK to use the zio-json based JSON decoding mechanism together with the AWS RequestStreamHandler,
deployed with a conventional SAM Cli deployment of a .jar file (or through AWS console, etc.).
`RequestStreamHandler` is the AWS-Java built-in way to pass the JSON data to/from our Java/Scala Lambda function as a JSON text Stream (instead of as java objects),
which is generally comparable to the streaming approach of ZIO-lambda's custom runtime (although ZIO-lambda interacts differently with AWS Lambda internals).  

https://github.com/aws/aws-lambda-java-libs/blob/main/aws-lambda-java-core/src/main/java/com/amazonaws/services/lambda/runtime/RequestStreamHandler.java


### Building an AWS Container-ized App


### From Future
whoopee!


I recently made some basic zio-dynamodb code work under AWS lambda, without using zio-lambda.
My setup and assumptions wound up being rather different from what the zio-lambda component presumes, although I started off trying to use it.

I am partially echoing the deployment perspective of earlier questions+comments in this chat by @rbraley and @CarlosLaraFP .

I feel like the ZIO ecosystem can benefit from a document explaining "How to use ZIO with AWS Lambda and SAM", 
explaining what the options are, including what zio-lambda currently offers, and what other basic approaches are possible.

I also feel like perhaps the ZIO ecosystem would benefit from a homepage for introductory content explaining "ZIO-on-AWS",

to help newcomers sort out which of the AWS-oriented ZIO ecosystem libraries ( ⁠zio-aws , ⁠zio-connect , ⁠zio-lambda , ⁠zio-dynamodb, ⁠zio-s3 , ⁠zio-sqs ,  ...) are most beneficial to their projects
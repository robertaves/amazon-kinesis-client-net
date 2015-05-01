# Amazon Kinesis Client Library for .NET

This package provides an interface to the [Amazon Kinesis Client Library][amazon-kcl] (KCL) [MultiLangDaemon][multi-lang-daemon] for the .NET Framework.

Developers can use the KCL to build distributed applications that process streaming data reliably at scale. The KCL takes care of many of the complex tasks associated with distributed computing, such as load balancing across multiple instances, responding to instance failures, checkpointing processed records, and reacting to changes in stream volume.

This package wraps and manages the interaction with the *MultiLangDaemon*, which is provided as part of the [Amazon KCL for Java][amazon-kcl-github] so that developers can focus on implementing their record processing logic.

A record processor in C# typically looks something like the following:

```csharp
using System;
using System.Collections.Generic;
using Amazon.Kinesis.ClientLibrary;

namespace Sample
{
    class SampleRecordProcessor : IRecordProcessor
    {
        public void Initialize(InitializationInput input)
        {
            // initialize
        }

        public void ProcessRecords(ProcessRecordsInput input)
        {
            // process batch of records (input.Records) and
            // checkpoint (using input.Checkpointer)
        }

        public void Shutdown(ShutdownInput input)
        {
            // cleanup
        }
    }

    class MainClass
    {
        public static void Main(string[] args)
        {
            KCLProcess.Create(new SampleRecordProcessor()).Run();
        }
    }
}
```

For more information about [Amazon Kinesis][amazon-kinesis] and the client libraries, see the
[official documentation][amazon-kinesis-docs] as well as the [Amazon Kinesis forums][kinesis-forum].

## Getting started


### Set up your AWS credentials

Before running a KCL application, make sure that your environment is configured to allow the *MultiLangDaemon* to access your [AWS security credentials](http://docs.aws.amazon.com/general/latest/gr/aws-security-credentials.html).

If you've installed the [AWS SDK for .NET][aws-sdk-dotnet], you may have already configured your AWS credentials using the SDK credential store in Microsoft Visual Studio; however, because Amazon KCL for .NET applications never deal with your credentials directly but defer to the *MultiLangDaemon*, this store is not available to your KCL application.

Instead, you can provide your credentials through environment variables (**AWS\_ACCESS\_KEY\_ID** and **AWS\_SECRET\_ACCESS\_KEY**) or a [credential profile][aws-sdk-dotnet-credentials] in your home directory.

### Building and running the sample projects

In addition to source code for the Amazon KCL for .NET itself, this repository contains a sample application, which can serve as a starting point for your KCL application. To try out this sample application, you can [download a ZIP][master-zip] of the latest sources, the contents of which can be opened as a solution in [Microsoft Visual Studio][visual-studio].

The sample application consists of two projects:

* **A data producer** (_SampleProducer\\SampleProducer.cs_)  
This program creates an Amazon Kinesis stream and continuously puts random records into it. There is commented-out code that deletes the created stream at the end; however, you should uncomment and use this code only if you do not intend to run SampleConsumer.

* **A data processor** (_SampleConsumer\\SampleConsumer.cs_)  
A new instance of this program is invoked by the *MultiLangDaemon* for each shard in the stream. It consumes the data from the shard. If you no longer need to work with the stream after running SampleConsumer, remember to delete both the Amazon DynamoDB checkpoint table and the Kinesis stream in your AWS account.

The following defaults are used in the sample application:

* **Stream name**: _myTestStream_ 
* **Number of shards**: _1_

### Running the data producer

To run the data producer, run the *SampleProducer* project.
#### Notes

* The [AWS SDK for .NET][aws-sdk-dotnet] must be installed as a prerequisite to running the producer.

### Running the data processor

Because the Amazon KCL for .NET requires the *MultiLangDaemon*, which is provided by the Amazon KCL for Java, a bootstrap program has been provided. This program downloads all required dependencies prior to invoking the *MultiLangDaemon*, which executes the processor as a subprocess.

To run the processor, first **build the SampleConsumer project**, then **run the bootstrap project** with the following configuration:

* **Working directory**: _the root of the SampleConsumer project_
* **Arguments**: `--properties kcl.properties --execute`


#### Notes

* You must have [Java][jvm] installed.
* If you omit the `--execute` argument, the bootstrap program outputs a command that can be used to start the KCL directly.
* The *MultiLangDaemon* reads its configuration from the `kcl.properties` file, which contains a few important settings:
  * **executableName = SampleProcessor.exe**  
    The name of the processor executable.
  * **streamName = myTestStream**  
    The name of the Kinesis stream from which to read data. This must match the stream name used by your producer.
  * More options are described in the [properties file][sample-properties].

### Cleaning up

This sample application creates a few resources in the default region of your AWS account:

* A Kinesis stream named _myTestStream_, which stores the data generated by your producer
* A DynamoDB table named _DotNetKinesisSample_,  which tracks the state of your processor

Each of these resources will continue to incur AWS service costs until they are deleted. After you are finished testing the sample application, you can delete these resources through the [AWS Management Console][aws-console].

## What you should know about the MultiLangDaemon

The Amazon KCL for .NET uses the [Amazon KCL for Java][amazon-kcl-github] internally. We have implemented a Java-based daemon, called the *MultiLangDaemon*, which handles all of the heavy lifting. Our approach has the daemon spawn the user-defined record processor program as a sub-process. The *MultiLangDaemon* communicates with this sub-process over standard input/output using a simple protocol, and therefore the record processor program can be written in any language.

At runtime, there will always be a one-to-one correspondence between a record processor, a child process,
and an [Amazon Kinesis shard][amazon-kinesis-shard]. The *MultiLangDaemon* ensures that, without
any developer intervention.

In this release, we have abstracted these implementation details and exposed an interface that enables
you to focus on writing record processing logic in C#. This approach enables the [Amazon KCL][amazon-kcl] to
be language-agnostic, while providing identical features and similar parallel processing model across
all languages.

## See Also

* [Developing Processor Applications for Amazon Kinesis Using the Amazon Kinesis Client Library][amazon-kcl]
* [Amazon KCL for Java][amazon-kcl-github]
* [Amazon KCL for Python][amazon-kinesis-python-github]
* [Amazon KCL for Ruby][amazon-kinesis-ruby-github]
* [Amazon KCL for Node.js][amazon-kinesis-nodejs-github]
* [Amazon Kinesis documentation][amazon-kinesis-docs]
* [Amazon Kinesis forum][kinesis-forum]

[amazon-kinesis]: http://aws.amazon.com/kinesis
[amazon-kinesis-docs]: http://aws.amazon.com/documentation/kinesis/
[amazon-kinesis-shard]: http://docs.aws.amazon.com/kinesis/latest/dev/key-concepts.html
[amazon-kcl]: http://docs.aws.amazon.com/kinesis/latest/dev/kinesis-record-processor-app.html
[amazon-kcl-github]: https://github.com/awslabs/amazon-kinesis-client
[amazon-kinesis-python-github]: https://github.com/awslabs/amazon-kinesis-client-python
[amazon-kinesis-ruby-github]: https://github.com/awslabs/amazon-kinesis-client-ruby
[amazon-kinesis-nodejs-github]: https://github.com/awslabs/amazon-kinesis-client-nodejs
[multi-lang-daemon]: https://github.com/awslabs/amazon-kinesis-client/blob/master/src/main/java/com/amazonaws/services/kinesis/multilang/package-info.java
[DefaultAWSCredentialsProviderChain]: http://docs.aws.amazon.com/AWSJavaSDK/latest/javadoc/com/amazonaws/auth/DefaultAWSCredentialsProviderChain.html
[kinesis-forum]: http://developer.amazonwebservices.com/connect/forum.jspa?forumID=169
[master-zip]: https://github.com/awslabs/amazon-kinesis-client-net/archive/master.zip
[aws-sdk-dotnet]: http://aws.amazon.com/developers/getting-started/net/
[aws-sdk-dotnet-credentials]: http://docs.aws.amazon.com/AWSSdkDocsNET/latest/DeveloperGuide/net-dg-config-creds.html#net-dg-config-creds-creds-file
[aws-console]: http://aws.amazon.com/console/
[sample-properties]: https://github.com/awslabs/amazon-kinesis-client-net/blob/master/SampleConsumer/kcl.properties
[visual-studio]: http://www.visualstudio.com/
[jvm]: http://java.com/en/download/

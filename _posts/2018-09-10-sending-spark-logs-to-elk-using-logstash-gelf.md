---
layout: post
title: Sending Spark logs to ELK using logstash-gelf
author: Karim Essawi
comments: true
tags: [Logging, Spark]
---
An important part of any application is its underlying log system. Logs are fundamental for debugging and traceability, but can also be useful for further analysis and even in areas such as business intelligence and key performance indicators (KPIs). At Auto Trader, we emphasise the importance of building a robust application logging system that can be integrated into our [ELK](https://www.elastic.co/elk-stack) stack that serves as a centralised log store.

### What Is Spark?
[Apache Spark](https://spark.apache.org/) is a popular tool used in the big data/data science domain. In industry, it is often used to transform large amounts of data, and to carry out computationally expensive machine learning jobs. At Auto Trader, we use it to provide real-time self-serve access to our repository of data which stores our historic view of the UK automotive marketplace.

Spark allows our Data Engineers to create applications and services which provide customised queries against this data to other parts of the organisation. For example, we use Spark to run queries which determine whether the price on an advert is above or below the market average.

### Log4j in Spark
An integral part of the Spark ecosystem is logging. Spark uses [log4j](https://logging.apache.org/log4j/2.x/) as the standard library for its own logging. Everything that happens inside Spark gets logged to the shell console and to the configured underlying destination.

A standard practice within Auto Trader is to send application logs to [Logstash](https://www.elastic.co/products/logstash) for troubleshooting and further processing. This task has been made easy for developers as they can use an in-house library to configure their application logging. But this library was incompatible with Spark applications as it was written for [Logback](https://logback.qos.ch/) (which is our standard logging implementation) since it needed access to parts of the logging API not exposed in [SLF4J](https://www.slf4j.org/). To get around this issue, we used ‘logstash-gelf’ directly, which is what the in-house library uses behind the scenes.

### Logstash-gelf
[Logstash-gelf](https://github.com/mp911de/logstash-gelf) is an open-source library that provides logging to Logstash using the [Graylog Extended Logging Format](http://docs.graylog.org/en/2.4/pages/gelf.html) (or simply GELF). With logstash-gelf, one can use a log4j XML file to configure Spark to log to Logstash. Here is an example XML configuration file:

```xml
<Configuration packages="biz.paluch.logging.gelf.log4j2">
    <Appenders>
        <Gelf name="gelf" host="udp:localhost" port="12201" version="1.1" extractStackTrace="true"
              filterStackTrace="true" mdcProfiling="true" includeFullMdc="true" maximumMessageSize="8192"
              originHost="%host{fqdn}" additionalFieldTypes="fieldName1=String,fieldName2=Double,fieldName3=Long">
            <Field name="timestamp" pattern="%d{dd MMM yyyy HH:mm:ss,SSS}" />
            <Field name="level" pattern="%level" />
            <Field name="simpleClassName" pattern="%C{1}" />
            <Field name="className" pattern="%C" />
            <Field name="server" pattern="%host" />
            <Field name="server.fqdn" pattern="%host{fqdn}" />
            
            <!-- This is a static field -->
            <Field name="fieldName2" literal="fieldValue2" />
             
            <!-- This is a field using MDC -->
            <Field name="mdcField2" mdc="mdcField2" /> 
            <DynamicMdcFields regex="mdc.*" />
            <DynamicMdcFields regex="(mdc|MDC)fields" />
        </Gelf>
    </Appenders>
    <Loggers>
        <Root level="INFO">
            <AppenderRef ref="gelf" />			
        </Root>
    </Loggers>
</Configuration>   
```

The above example creates a GELF appender with a Logstash host `udp:localhost` on port `12201`. It then adds the GELF appender to the root logger. Notice how the transfer protocol is specified as part of the hostname. So if you prefer TCP over UDP, then the host will be `tcp:hostname`.

This will work perfectly fine, but you may have noticed that the properties file is quite verbose, which could lead to errors and inconsistencies when shared across Spark applications. Luckily, this can all be done in code as well.

### Configure logstash-gelf programmatically
Logstash-gelf lets you create and configure appenders programmatically. Logstash-gelf's building block is the `GelfLogAppender` class which creates GELF messages and posts them using UDP (default) or TCP. It can be configured by setting class properties such as `host`, `port` etc. One can simply write a method that takes the desired configuration and applies them to an instance of `GelfLogAppender`. We're going to use Scala in the following examples to keep it consistent with our Spark Scala codebase, as we'll show later how to use Logstash-gelf in your Spark applications.

```scala
def createGelfLogAppender(host: String, port: String): GelfAppender = {
  val appender = new GelfLogAppender()
  appender.setHost(host)
  appender.setPort(port)
  appender.setExtractStackTrace(true)
  appender.setFilterStackTrace(false)
  appender.setMaximumMessageSize(8192)
  appender.setIncludeFullMdc(true)
  appender.activateOptions()
  
  appender
} 
```

Notice the call to `activateOptions()` at the end; this method creates a `GelfSender` based on the passed in configuration. `GelfSender` is a strategy interface to send a `GelfMessage` to the specified Logstash host without being opinionated about the underlying transport. For example, if you specify UDP in your host configurtion, the `GelfUDPSender` implementation will be used.

Note that if you do choose UDP as the transport protocol, then you'll need to take into account the [MTU (Maximum Transmission Unit)](https://en.wikipedia.org/wiki/Maximum_transmission_unit) of your network when calling `setMaximumMessageSize` to avoid large UDP packets being silently dropped, as this is a common symptom when working with logs and large stack traces. You can  determine the appropritate max message size by either trial and error or getting advice from your network engineers.

That's nice and tidy, but we still need to add this GELF appender to the root logger. To do this, you could write something like:

```scala
val rootLogger = Logger.getRootLogger
rootLogger.addAppender(createGelfLogAppender(host = "your Logstash host", port = port))
```

That was easy, but it still feels a bit overkill to repeat this piece of code for every Spark application.

Here at Auto Trader, we thought of creating a centralised logging solution that easily enables developers to configure their Spark applications to send logs to Logstash. Conveniently, we already had a library for common Spark code that can be shared across applications. So we just had to extend the library with the above code snippets. This enabled developers to create a Spark session and configure a GELF appender all in one go. Here is the code to achieve this:

```scala
object Spark {
  def createSessionWithLogging(
      appName: String,
      appVersion: String,
      sparkConfig: Map[String, String]): SparkSession = {

    val rootLogger = Logger.getRootLogger
    rootLogger.addAppender(createGelfLogAppender(host = "udp:logstash-host.your-network", port = 123))
    
    SparkSession.builder()
        .appName(appName)
        .config(sparkConfig)
        .getOrCreate()
  }
}
```

This way, the only thing that developers would need to do is call `Spark.createSessionWithLogging()` with their desired configuration parameters and they will get a Spark session with logging to Logstash already configured.

### In Summary 
We've shown how to send Spark logs to Logstash using **logstash-gelf**. It's quite flexible as you can configure logging with a log4j XML file or you can do it in code if you don’t like the verboseness of XML.

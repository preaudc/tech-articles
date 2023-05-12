# Fix flume fileChannels filling because of invalid country header

Apache Flume is an efficient and flexible tool for ingesting and processing large amounts of data in a distributed environment

Flume operates in a client-server architecture, where the agents on each machine collect the data and transport it to a centralized Flume server, which then writes the data to the configured data sinks. It supports a variety of data sources, including log files, syslog, netcat, and more. Flume also provides a rich set of plugins and extensions, which makes it highly customizable and extensible.

Flume is especially suitable for moving large amounts of log data and other streaming data from various sources to a centralized data store such as HDFS.

## MyCompany flume topology description

rMyLogType is defined as an [Avro Source](https://flume.apache.org/releases/content/1.11.0/FlumeUserGuide.html#avro-source):

```
a1.sources.rMyLogType.type = avro
a1.sources.rMyLogType.port = 44444
a1.sources.rMyLogType.bind = flume.mycompany.com
a1.sources.rMyLogType.channels = cMyLogType
```

cMyLogType is a [File Channel](https://flume.apache.org/releases/content/1.11.0/FlumeUserGuide.html#file-channel).
This flume channel send its events from the rMyLogType source to the kMyLogType sink:

```
a1.channels.cMyLogType.type = file
```

Finally kMyLogType is an [HDFS Sink](https://flume.apache.org/releases/content/1.11.0/FlumeUserGuide.html#hdfs-sink).
This flume sink uses the country header to set the directory path of the file in which the events are written (see `%{country}` in the `a1.sinks.kMyLogType.hdfs.path` property below):

```
a1.sinks.kMyLogType.channel = cMyLogType
a1.sinks.kMyLogType.type = hdfs
a1.sinks.kMyLogType.hdfs.filePrefix = MyLogType_%Y%m%d_%H_flume_a1
a1.sinks.kMyLogType.hdfs.inUseSuffix = .tmp
a1.sinks.kMyLogType.hdfs.codeC = gzip
a1.sinks.kMyLogType.hdfs.fileType = SequenceFile
a1.sinks.kMyLogType.hdfs.path = hdfs://mycompany-cluster/user/mycompany/logs/flume/myLogType/%{country}/current
a1.sinks.kMyLogType.hdfs.fileSuffix = .seq
```

The problem is that the country header, which should be a 2-character country code, is defined using an unreliable external string field, and can sometimes be set to an invalid value, e.g.:

```
fr'+(select*from(select(sleep(20)))a)+'
fr'.)(.,")).
fr'XJdBkx<'">tHndBx
fr'nvOpzp; AND 1=1 OR (<'">iKO)),
fr) AND 1884=1884-- ffjs
fr) AND 4904=4904 AND (5459=5459
fr) AND 6726=8402 AND (8335=8335
fr) AND 8263=3105-- WYZL
fr)) AND 1535=5731 AND ((3940=3940
fr)) AND 1884=1884-- XkBD
fr)) AND 4904=4904 AND ((3073=3073
fr)) AND 7007=7794-- lVlA
fr)) ORDER BY 1-- Hmpc
fr70236806' or 5536=5540--
friy3j4h234hjb23234
```

Moreover, the country header sometimes contains characters which are invalid for an HDFS directory name. The flume agent will raise an IllegalArgumentException exception when that happens:

```
Caused by: java.lang.IllegalArgumentException: Pathname /user/mycompany/logs/flume/myLogType/fr';WAITFOR DELAY '0:0:32'--/current/_MyLogType_20220707_14_flume01_a3.1657202736341.seq.tmp from hdfs://mycompany-cluster/user/mycompany/logs/flume/myLogType/fr';WAITFOR DELAY '0:0:32'--/current/_MyLogType_20220707_14_flume01_a3.1657202736341.seq.tmp is not a valid DFS filename.
```

Even worse, the event will be stuck in the cMyLogType file channel, and will block all the following events!

As a consequence, the channel fill percentage will kept growing, you can check that with checkFlumeFillPercentages.sh

```shell
# FLUME_HOME is set to the path to the flume home directory, e.g.
# FLUME_HOME=/opt/flume
# 
# The monitoring ports for each flume agent are defined in the configuration
# file at $FLUME_HOME/conf/flume-env.sh, e.g. AGENT_1_MONITORING_PORT=34545
--> curl -s http://flume.mycompany.com:$(grep AGENT_1_MONITORING_PORT $FLUME_HOME/conf/flume-env.sh | cut -d= -f2)/metrics | jq -Mr '.["CHANNEL.cMyLogType"] | .ChannelFillPercentage'
4.56146
```

Step-by-step guide

    Use the below topology (dubbed flushFileChannelTopology) to flush the cMyLogType file channel:
    [Data Platform > Fix flume fileChannels filling because of invalid country header > MyLogTypeChannelFix_02.png]
        flush the events from the cMyLogType channel into the kMyLogTypeAvro avro sink
        fix the country header (drop all characters except the first two ones) in the rMyLogTypeAvro avro source using a custom flume interceptor
        write the events with fixed country header in kMyLogType hdfs sink (by routing them through cMyLogTypeAvro file channel)

    Restart the impacted flume agents, and wait for the channel fill percentage of cMyLogType file channels to decrease to 0%

    ## e.g. for agent a1
    # set flushFileChannelTopology above in $FLUME_HOME/conf/agent_a1/flume.conf
    # add jar with custom flume interceptor in $FLUME_HOME/javalib/
    # restart the flume agent a1 as root
    service flume_a1 restart
    # check file channels fill percentage on your local machine
    ./checkFlumeFillPercentages.sh flume.mycompany.com


    Revert back to the old topology, but with the above interceptor configured in the rMyLogType source (dubbed interceptorTopology):
    first diagram is the original topology, second one is the interceptorTopology
    [Data Platform > Fix flume fileChannels filling because of invalid country header > MyLogTypeChannelFix_01.png]
        fix the country header (drop all characters except the first two ones) in the rMyLogType avro source using a custom flume interceptor
        write the events with fixed country header in kMyLogType hdfs sink (by routing them through cMyLogType file channel)

    Restart the impacted flume agents to load the interceptorTopology:

    ## e.g. for agent a1
    # set interceptorTopology above in $FLUME_HOME/conf/agent_a1/flume.conf
    # restart the flume agent a1 as root
    service flume_a1 restart
    # check file channels fill percentage on your local machine
    ./checkFlumeFillPercentages.sh flume.mycompany.com


The explanation above only mentions the cMyLogType file channel, but the reasoning and fix is exactly the same for the cMyLogTypeLocal file channel.
Internal resources

    example file for flushFileChannelTopology: flume.conf
    relevant sections:

    (...)

    a1.sources = rKlsAccess rImp rOfferImp rLead rTracking rSalesTracking rSalesTrackingActivation rLinkMonetizerImpression rLinkMonetizerClick rCookieConsent rMyLogType rMyLogTypeAvro rMyLogTypeLocalAvro
    a1.sinks = kKlsAccess kImp kOfferImp kLead kTracking kSalesTracking kSalesTrackingActivation kLinkMonetizerImpression kLinkMonetizerClick kKlsAccessLocal kSalesTrackingLocal kTrackingLocal kSalesTrackingActivationLocal kCookieConsent kMyLogType kMyLogTypeLocal kMyLogTypeAvro kMyLogTypeLocalAvro
    a1.channels = cKlsAccess cKlsAccessLocal cImp cOfferImp cLead cTracking cTrackingLocal cSalesTracking cSalesTrackingLocal cSalesTrackingActivation cSalesTrackingActivationLocal cLinkMonetizerImpression cLinkMonetizerClick cCookieConsent cMyLogType cMyLogTypeLocal cMyLogTypeAvro cMyLogTypeLocalAvro

    (...)

    a1.sinks.kMyLogType.type = hdfs
    a1.sinks.kMyLogType.channel = cMyLogTypeAvro

    (...)

    a1.sinks.kMyLogTypeLocal.type = hdfs
    a1.sinks.kMyLogTypeLocal.channel = cMyLogTypeLocalAvro

    (...)

    a1.sources.rMyLogTypeAvro.type = avro
    a1.sources.rMyLogTypeAvro.bind = localhost
    a1.sources.rMyLogTypeAvro.port = 44567
    a1.sources.rMyLogTypeAvro.channels = cMyLogTypeAvro
    a1.sources.rMyLogTypeAvro.interceptors = i1
    a1.sources.rMyLogTypeAvro.interceptors.i1.type = com.mycompany.common.flume.interceptor.FixCountryHeaderInterceptor$Builder

    a1.sources.rMyLogTypeLocalAvro.type = avro
    a1.sources.rMyLogTypeLocalAvro.bind = localhost
    a1.sources.rMyLogTypeLocalAvro.port = 44568
    a1.sources.rMyLogTypeLocalAvro.channels = cMyLogTypeLocalAvro
    a1.sources.rMyLogTypeLocalAvro.interceptors = i1
    a1.sources.rMyLogTypeLocalAvro.interceptors.i1.type = com.mycompany.common.flume.interceptor.FixCountryHeaderInterceptor$Builder

    a1.sinks.kMyLogTypeAvro.type = avro
    a1.sinks.kMyLogTypeAvro.channel = cMyLogType
    a1.sinks.kMyLogTypeAvro.hostname = localhost
    a1.sinks.kMyLogTypeAvro.port = 44567

    a1.sinks.kMyLogTypeLocalAvro.type = avro
    a1.sinks.kMyLogTypeLocalAvro.channel = cMyLogTypeLocal
    a1.sinks.kMyLogTypeLocalAvro.hostname = localhost
    a1.sinks.kMyLogTypeLocalAvro.port = 44568

    a1.channels.cMyLogTypeAvro.type = file
    a1.channels.cMyLogTypeAvro.capacity = 30000000
    a1.channels.cMyLogTypeAvro.checkpointDir = /opt/mycompany/data/flume/agent_a1/fileChannelCheckpoint_cMyLogTypeAvro
    a1.channels.cMyLogTypeAvro.dataDirs = /opt/mycompany/data/flume/agent_a1/fileChannelData_cMyLogTypeAvro

    a1.channels.cMyLogTypeLocalAvro.type = file
    a1.channels.cMyLogTypeLocalAvro.capacity = 3000000
    a1.channels.cMyLogTypeLocalAvro.checkpointDir = /opt/mycompany/data/flume/agent_a1/fileChannelCheckpoint_cMyLogTypeLocalAvro
    a1.channels.cMyLogTypeLocalAvro.dataDirs = /opt/mycompany/data/flume/agent_a1/fileChannelData_cMyLogTypeLocalAvro


    example file for interceptorTopology: flume.conf
    relevant section:

    a1.sources.rMyLogType.type = avro
    a1.sources.rMyLogType.bind = flume.mycompany.com
    a1.sources.rMyLogType.port = 44546
    a1.sources.rMyLogType.channels = cMyLogType cMyLogTypeLocal
    a1.sources.rMyLogType.interceptors = i1
    a1.sources.rMyLogType.interceptors.i1.type = com.mycompany.common.flume.interceptor.FixCountryHeaderInterceptor$Builder


    custom flume interceptor: FixCountryHeaderInterceptor.java






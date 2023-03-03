# Fix flume fileChannels filling because of invalid country header

The cClickTracking file channel send its events to the kClickTracking hdfs sink. This sink uses the country header to set the directory path of the file in which the events are written:

# for example, for agent a1
a1.sinks.kClickTracking.hdfs.path = hdfs://prod-cluster/user/kookel/logs/flume/clickTracking/%{country}/current

However, the country header, which should be a 2-character country code, is sometimes set to an invalid value, e.g.:

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

Besides, the country header sometimes contains characters which are invalid for an HDFS directory name. The flume agent will raise an IllegalArgumentException exception when that happens:

Caused by: java.lang.IllegalArgumentException: Pathname /user/kookel/logs/flume/clickTracking/fr';WAITFOR DELAY '0:0:32'--/current/_ClickTracking_20220707_14_flume01_a3.1657202736341.seq.tmp from hdfs://prod-cluster/user/kookel/logs/flume/clickTracking/fr';WAITFOR DELAY '0:0:32'--/current/_ClickTracking_20220707_14_flume01_a3.1657202736341.seq.tmp is not a valid DFS filename.

Even worse, the event will be stuck in the cClickTracking file channel, and will block all the following events!

As a consequence, the channel fill percentage will kept growing, you can check that with checkFlumeFillPercentages.sh

--> ./checkFlumeFillPercentages.sh dc1-kdp-prod-flume-01.prod.dc1.kelkoo.net,dc1-kdp-prod-flume-02.prod.dc1.kelkoo.net cClickTracking
--- dc1-kdp-prod-flume-01.prod.dc1.kelkoo.net ---
(...)
cClickTracking                a1:0.00000                    a2:0.00000                    a3:4.56146                    a4:4.56222
(...)
--- dc1-kdp-prod-flume-02.prod.dc1.kelkoo.net ---
(...)
cClickTracking                a1:4.55971                    a2:4.57077                    a3:0.00000                    a4:0.00001
(...)


The issue is the same with the cClickTrackingLocal file channel, except that the set of characters allowed for a directory name is more restricted on HDFS than on EXT4.
Step-by-step guide

    Use the below topology (dubbed flushFileChannelTopology) to flush the cClickTracking file channel:
    [Data Platform > Fix flume fileChannels filling because of invalid country header > ClickTrackingChannelFix_02.png]
        flush the events from the cClickTracking channel into the kClickTrackingAvro avro sink
        fix the country header (drop all characters except the first two ones) in the rClickTrackingAvro avro source using a custom flume interceptor
        write the events with fixed country header in kClickTracking hdfs sink (by routing them through cClickTrackingAvro file channel)

    Restart the impacted flume agents, and wait for the channel fill percentage of click tracking file channels to decrease to 0%

    ## e.g. for agent a1
    # set flushFileChannelTopology above in /opt/kookel/kelkooFlume/current/conf/agent_a1/flume.conf
    # add jar with custom flume interceptor in /opt/kookel/kelkooFlume/current/javalib/
    # restart the flume agent a1 as root
    service kelkooFlume_a1 restart
    # check file channels fill percentage on your local machine
    ./checkFlumeFillPercentages.sh dc1-kdp-prod-flume-01.prod.dc1.kelkoo.net,dc1-kdp-prod-flume-02.prod.dc1.kelkoo.net


    Revert back to the old topology, but with the above interceptor configured in the rClickTracking source (dubbed interceptorTopology):
    first diagram is the original topology, second one is the interceptorTopology
    [Data Platform > Fix flume fileChannels filling because of invalid country header > ClickTrackingChannelFix_01.png]
        fix the country header (drop all characters except the first two ones) in the rClickTracking avro source using a custom flume interceptor
        write the events with fixed country header in kClickTracking hdfs sink (by routing them through cClickTracking file channel)

    Restart the impacted flume agents to load the interceptorTopology:

    ## e.g. for agent a1
    # set interceptorTopology above in /opt/kookel/kelkooFlume/current/conf/agent_a1/flume.conf
    # restart the flume agent a1 as root
    service kelkooFlume_a1 restart
    # check file channels fill percentage on your local machine
    ./checkFlumeFillPercentages.sh dc1-kdp-prod-flume-01.prod.dc1.kelkoo.net,dc1-kdp-prod-flume-02.prod.dc1.kelkoo.net


The explanation above only mentions the cClickTracking file channel, but the reasoning and fix is exactly the same for the cClickTrackingLocal file channel.
Internal resources

    example file for flushFileChannelTopology: flume.conf
    relevant sections:

    (...)

    a1.sources = rKlsAccess rImp rOfferImp rLead rTracking rSalesTracking rSalesTrackingActivation rLinkMonetizerImpression rLinkMonetizerClick rCookieConsent rClickTracking rClickTrackingAvro rClickTrackingLocalAvro
    a1.sinks = kKlsAccess kImp kOfferImp kLead kTracking kSalesTracking kSalesTrackingActivation kLinkMonetizerImpression kLinkMonetizerClick kKlsAccessLocal kSalesTrackingLocal kTrackingLocal kSalesTrackingActivationLocal kCookieConsent kClickTracking kClickTrackingLocal kClickTrackingAvro kClickTrackingLocalAvro
    a1.channels = cKlsAccess cKlsAccessLocal cImp cOfferImp cLead cTracking cTrackingLocal cSalesTracking cSalesTrackingLocal cSalesTrackingActivation cSalesTrackingActivationLocal cLinkMonetizerImpression cLinkMonetizerClick cCookieConsent cClickTracking cClickTrackingLocal cClickTrackingAvro cClickTrackingLocalAvro

    (...)

    a1.sinks.kClickTracking.type = hdfs
    a1.sinks.kClickTracking.channel = cClickTrackingAvro

    (...)

    a1.sinks.kClickTrackingLocal.type = hdfs
    a1.sinks.kClickTrackingLocal.channel = cClickTrackingLocalAvro

    (...)

    a1.sources.rClickTrackingAvro.type = avro
    a1.sources.rClickTrackingAvro.bind = localhost
    a1.sources.rClickTrackingAvro.port = 44567
    a1.sources.rClickTrackingAvro.channels = cClickTrackingAvro
    a1.sources.rClickTrackingAvro.interceptors = i1
    a1.sources.rClickTrackingAvro.interceptors.i1.type = com.kelkoogroup.common.flume.interceptor.FixCountryHeaderInterceptor$Builder

    a1.sources.rClickTrackingLocalAvro.type = avro
    a1.sources.rClickTrackingLocalAvro.bind = localhost
    a1.sources.rClickTrackingLocalAvro.port = 44568
    a1.sources.rClickTrackingLocalAvro.channels = cClickTrackingLocalAvro
    a1.sources.rClickTrackingLocalAvro.interceptors = i1
    a1.sources.rClickTrackingLocalAvro.interceptors.i1.type = com.kelkoogroup.common.flume.interceptor.FixCountryHeaderInterceptor$Builder

    a1.sinks.kClickTrackingAvro.type = avro
    a1.sinks.kClickTrackingAvro.channel = cClickTracking
    a1.sinks.kClickTrackingAvro.hostname = localhost
    a1.sinks.kClickTrackingAvro.port = 44567

    a1.sinks.kClickTrackingLocalAvro.type = avro
    a1.sinks.kClickTrackingLocalAvro.channel = cClickTrackingLocal
    a1.sinks.kClickTrackingLocalAvro.hostname = localhost
    a1.sinks.kClickTrackingLocalAvro.port = 44568

    a1.channels.cClickTrackingAvro.type = file
    a1.channels.cClickTrackingAvro.capacity = 30000000
    a1.channels.cClickTrackingAvro.checkpointDir = /opt/kookel/data/kelkooFlume/agent_a1/fileChannelCheckpoint_cClickTrackingAvro
    a1.channels.cClickTrackingAvro.dataDirs = /opt/kookel/data/kelkooFlume/agent_a1/fileChannelData_cClickTrackingAvro

    a1.channels.cClickTrackingLocalAvro.type = file
    a1.channels.cClickTrackingLocalAvro.capacity = 3000000
    a1.channels.cClickTrackingLocalAvro.checkpointDir = /opt/kookel/data/kelkooFlume/agent_a1/fileChannelCheckpoint_cClickTrackingLocalAvro
    a1.channels.cClickTrackingLocalAvro.dataDirs = /opt/kookel/data/kelkooFlume/agent_a1/fileChannelData_cClickTrackingLocalAvro


    example file for interceptorTopology: flume.conf
    relevant section:

    a1.sources.rClickTracking.type = avro
    a1.sources.rClickTracking.bind = dc1-kdp-prod-flume-01.prod.dc1.kelkoo.net
    a1.sources.rClickTracking.port = 44546
    a1.sources.rClickTracking.channels = cClickTracking cClickTrackingLocal
    a1.sources.rClickTracking.interceptors = i1
    a1.sources.rClickTracking.interceptors.i1.type = com.kelkoogroup.common.flume.interceptor.FixCountryHeaderInterceptor$Builder


    custom flume interceptor: FixCountryHeaderInterceptor.java

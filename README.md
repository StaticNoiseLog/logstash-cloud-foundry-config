- [Cloud Foundry and Logstash](#cloud-foundry-and-logstash)
  * [Basics](#basics)
  * [Why Not the Syslog Input Plugin](#why-not-the-syslog-input-plugin)
  * [Why HTTP Instead of TCP](#why-http-instead-of-tcp)
  * [The Multiline Problem](#the-multiline-problem)
  * [Known Weaknesses and Alternatives](#known-weaknesses-and-alternatives)
- [Setting Up Cloud Foundry for Logstash](#setting-up-cloud-foundry-for-logstash)
  * [JSON Log Output With Spring Boot](#json-log-output-with-spring-boot)
  * [Deploy Logstash on Cloud Foundry](#deploy-logstash-on-cloud-foundry)
  * [Creating a Route to Logstash in Cloud Foundry](#creating-a-route-to-logstash-in-cloud-foundry)
    + [Verifying an HTTP Route to Logstash](#verifying-an-http-route-to-logstash)
  * [Creating a Cloud Foundry Log Drain](#creating-a-cloud-foundry-log-drain)
- [The Logstash Pipeline](#the-logstash-pipeline)
  * [Debugging the Pipeline](#debugging-the-pipeline)
  * [Grok](#grok)
    + [Grok Example 1](#grok-example-1)
    + [Grok Example 2](#grok-example-2)
    + [Grok Example 3](#grok-example-3)
- [Cleaning Up](#cleaning-up)
  * [Unused Log Drains](#unused-log-drains)
  * [Ports Used by Logstash](#ports-used-by-logstash)
  * [Port Not Needed Anymore in Logstash](#port-not-needed-anymore-in-logstash)
  * [Goodbye ELK](#goodbye-elk)

<small><i><a href='http://ecotrust-canada.github.io/markdown-toc/'>Table of contents generated with markdown-toc</a></i></small>


Cloud Foundry and Logstash
==========================

Basics
------
To collect log lines from apps, Cloud Foundry recommends creating a so-called "log drain" service:

    cf cups MY-LOG-DRAIN -l syslog://log-aggregator.com

That log drain is then bound to apps running on Cloud Foundry and it forwards the apps' log lines to the specified destination (log-aggregator.com in this example). A popular log aggregator is Logstash.

The `syslog:` protocol tells us what the receiver (Logstash) has to expect: Syslog protocol. This looks promising because there is a [Syslog input plugin](https://www.elastic.co/guide/en/logstash/current/plugins-inputs-syslog.html) that can be used in the receiving Logstash pipeline:
```
input {
  syslog {
    port => 12345
    codec => cef
    syslog_field => "syslog"
    grok_pattern => "<%{POSINT:priority}>%{SYSLOGTIMESTAMP:timestamp} CUSTOM GROK HERE"
  }
}
```
One also finds examples where a syslog ```type``` is used with the ```tcp``` and ```udp``` input plugins:
```
input {
  tcp {
    port => 5000
    type => syslog
  }
  udp {
    port => 5000
    type => syslog
  }
}
```

Why Not the Syslog Input Plugin
-------------------------------
Experiments in 2020 revealed that Cloud Foundry log drains produce the [RFC5424](https://tools.ietf.org/html/rfc5424) version of the Syslog protocol, while the Logstash components seem to expect the older version [RFC3164](https://tools.ietf.org/html/rfc3164). Cloud Foundry with Syslog did not work "out of the box" and as other teams in our environment did not even consider the Syslog input plugin, we abandonded it.

At a later point in time it may be worth re-evaluating this option with newer verisons of Logstash.

Useful link: [How to Read a Syslog Message](https://docs.secureauth.com/display/KBA/How+to+Read+a+Syslog+Message)

Why HTTP Instead of TCP
-----------------------
I tried the [```tcp``` input plugin](https://www.elastic.co/guide/en/logstash/current/plugins-inputs-tcp.html) but it turned out that log messages would be split up into segments of about 1000 bytes. Maybe this is caused by a limited TCP window size somewhere in Cloud Foundry. But whatever the reason, I did not see a way to gain control over this.

Because others in our work environment were using the [```http``` input plugin](https://www.elastic.co/guide/en/logstash/current/plugins-inputs-http.html) with some success, we also went with this choice.

The Multiline Problem
----------------------
A logical unit of log information can comprise more than one line. A stack trace is a typical example. For a clear picture in Kibana you want these lines stored together in a single document in Elasticsearch.

There is a multiline codec plugin for Logstash that is supposed to be able to detect log lines that belong together. But even if that would work, it would not solve the problem in this context: A Cloud Foundry log drain can and should be used for many services. This means that a log drain forwards a mix of log lines from many services. A stack trace from one service could be interrupted by log lines from other services. Detecting which lines belong together in this scenario is impossible with Logstash's multiline codec.

The solution is to modify the apps so that they use JSON documents as the unit of logging. This is well supported for Spring Boot services.

Known Weaknesses and Alternatives
---------------------------------
Decoding of the Cloud Foundry log messages with Grok is based on trial and error and may break with future releases of Cloud Foundry.

Cloud Foundry components sometimes create multiline logs and they do *not* group them into a unit with JSON. This is not under our control and such messages look ugly in Kibana.

Better alternatives for the future might be:

- Replace Logstash by something like [Vector](https://vector.dev/).
- Try harder with Logstash's Syslog plugin.
- See if [Filebeat for Cloud Foundry](https://www.elastic.co/guide/en/beats/filebeat/master/running-on-cloudfoundry.html) has arrived.
- Write your own input plugin (Ruby) or codec ([possible in Java](https://www.elastic.co/guide/en/logstash/6.7/contributing-java-plugin.html)).


Setting Up Cloud Foundry for Logstash
=====================================
You basically need these components:

1. Apps that log in JSON format.
1. Logstash running on Cloud Foundry.
1. A Cloud Foundry route through which Logstash can be reached.
1. A Cloud Foundry log drain bound to the apps that fetches all log lines and forwards them to the route associated with Logstash.
1. A Logstash pipeline that can successfully decode all incoming log data so it can be reasonably stored in Elasticsearch and queried with the Kibana GUI.

The following sections describe the details of each point.

JSON Log Output With Spring Boot
--------------------------------
Use logback for logging and add these Gradle dependencies:

    runtimeOnly("net.logstash.logback:logstash-logback-encoder:6.3")
    runtimeOnly("ch.qos.logback:logback-classic:1.2.3")

In `logback-spring.xml` define an appender for JSON and activate it for the profiles and loggers as desired. You probably don't want to use JSON formatted log lines when developing on your computer, but when an app is deployed on Cloud Foundry it shall log in JSON.

    <appender name="json" class="ch.qos.logback.core.ConsoleAppender">
        <encoder class="net.logstash.logback.encoder.LogstashEncoder"/>
    </appender>

    <springProfile name="test, integration, production">
        <root level="INFO">
            <appender-ref ref="json"/>
        </root>
        <logger name="com.acme.nwb" level="TRACE" additivity="false">
            <appender-ref ref="json"/>
        </logger>
    </springProfile>

Deploy Logstash on Cloud Foundry
--------------------------------
Here is an example `manifest.yml` that can be used to `cf push` an official Logstash Docker image.

Under `routes:` the routes must be listed through which Logstash will be accessed by Cloud Foundry log drains. Associating a route with a service can be done in Cloud Foundry's web GUI (Routes, Map Route). But when the application is newly pushed that association would be forgotten, so make it permanent in the manifest.

And - important! - `XPACK_MANAGEMENT_PIPELINE_ID` must contain the name of all pipelines that you want to use. Defining the pipelines in the Kibana GUI alone is *not* enough!
```
applications:
  - name: digital-billing-logstash
    memory: 2G
    instances: 1
    routes:
      - route: digital-billing-logstash-apc.scapp-corp.acme.com
      - route: tcp-scapp-corp.acme.com:39337
    services:
      - digital-billing-elasticsearch
    docker:
      image: docker.elastic.co/logstash/logstash:7.3.1
    command: |
      export XPACK_MANAGEMENT_ELASTICSEARCH_HOSTS=$(echo $VCAP_SERVICES | grep -Po '"host":\s"\K(.*?)(?=")') &&
      export XPACK_MANAGEMENT_ELASTICSEARCH_USERNAME=$(echo $VCAP_SERVICES | grep -Po '"full_access_username":\s"\K(.*?)(?=")') &&
      export XPACK_MANAGEMENT_ELASTICSEARCH_PASSWORD=$(echo $VCAP_SERVICES | grep -Po '"full_access_password":\s"\K(.*?)(?=")') &&
      export XPACK_MONITORING_ELASTICSEARCH_HOSTS=$(echo $VCAP_SERVICES | grep -Po '"host":\s"\K(.*?)(?=")') &&
      export XPACK_MONITORING_ELASTICSEARCH_USERNAME=$(echo $VCAP_SERVICES | grep -Po '"full_access_username":\s"\K(.*?)(?=")') &&
      export XPACK_MONITORING_ELASTICSEARCH_PASSWORD=$(echo $VCAP_SERVICES | grep -Po '"full_access_password":\s"\K(.*?)(?=")') &&
      /usr/local/bin/docker-entrypoint
    env:
      XPACK_MANAGEMENT_ENABLED: true
      XPACK_MANAGEMENT_PIPELINE_ID: '["logs-apc", "logs-esc"]'
      XPACK_MONITORING_ENABLED: true
      PATH_CONFIG: ""
```

Creating a Route to Logstash in Cloud Foundry
---------------------------------------------
Because we will receive data in Logstash with the HTTP plugin, we need an HTTP route in Cloud Foundry. Such routes listen on the standard ports 80 (HTTP) and 443 (HTTPS) and forward requests to a destination (Logstash in our case).

Cloud Foundry space (Prod) and domain (scapp-corp.acme.com) are given in our case. But the hostname (and optionally the path) can be used to distinguish an **HTTP route**:

    cf create-route Prod scapp-corp.acme.com --hostname digital-billing-logstash-apc

To continue we need the GUID of the HTTP route just created:

    cf curl /v3/routes?per_page=1000
    ...
      {
         "guid": "630371ca-7669-4836-8427-f373ba819e92",  <<---- GUID of route
         "created_at": "2020-10-12T18:43:53Z",
         "updated_at": "2020-10-12T18:43:53Z",
         "protocol": "http",
         "host": "digital-billing-logstash-apc",          <<---- yes, our route!

**Decide what port you want to use for listening with the HTTP input plugin of your Logstash pipeline.** 5044 seems to be a default, but you can use something else. Here we choose **5046**.

First the Logstash app must be configured in Cloud Foundry so it will be capable of listening on port 5046. For this we need the GUID of the Logstash app:

    cf app digital-billing-logstash --guid
    9ba9860b-54d0-4ddb-ba0a-c30192890fc6

Find out the ports on which Logstash is already listening:

    cf curl /v2/apps/9ba9860b-54d0-4ddb-ba0a-c30192890fc6
    ...
     "ports": [
        5044,
        9600,
        37363,
        39337
     ],
    ...

Then do a PUT to **add the new port 5046 to Logstash**, *making sure to preserve the existing ports*:

    cf curl /v2/apps/9ba9860b-54d0-4ddb-ba0a-c30192890fc6 -X PUT -d "{\"ports\": [5044,5046,9600,37363,39337]}"

The next step is to create a **route-mapping**. This simply means establishing the destination to which the route leads: When an HTTP call is made to the route's URL it will be forwarded to that destination. The destination is the GUID of the target app (Logstash) and the desired port (5046).

Create the route-mapping, from the newly created route to the Logstash app, port 5046:

    cf curl /v2/route_mappings -X POST -d "{\"route_guid\": \"630371ca-7669-4836-8427-f373ba819e92\", \"app_guid\": \"9ba9860b-54d0-4ddb-ba0a-c30192890fc6\", \"app_port\": 5046"}

You can verify that the desired route-mapping exists in the output of the following command (look in the destinations, there should be one with the app.guid of Logstash and port 5046):

    cf curl /v3/apps/9ba9860b-54d0-4ddb-ba0a-c30192890fc6/routes

Our new route is now visible with `cf routes`. It is composed of the hostname (digital-billing-logstash-apc) and the domain (scapp-corp.acme.com) used when creating it: **digital-billing-logstash-apc.scapp-corp.acme.com**

### Verifying an HTTP Route to Logstash ###
To verify if the route actually works you can use the netcat utility `nc`. You need an app on Cloud Foundry to which you can connect with SSH:

    cf ssh my-cf-app

In the shell prompt type this:

    nc -v digital-billing-logstash-apc.scapp-corp.acme.com 80

If you have set up a pipeline with an HTTP input plugin listening on the correct port (the one used in the route destination, 5046 in our example) your terminal should wait for input. Try entering this:

    GET /url

If everything works, you should see something like this in response:

    HTTP/1.0 302 Found
    Location: https:///url
    Connection: close
    Content-Length: 0


There is no reason why HTTPS should not work, but test it, too:

    nc -v digital-billing-logstash-apc.scapp-corp.acme.com 443

If the `nc` command does not wait for input but returns you to the command prompt (possibly with an error) then something is missing. Either no pipeline running or something wrong with the route/connectivity.

Creating a Cloud Foundry Log Drain
----------------------------------
A log drain is a service that collects all log data of the apps that are bound to it. It forwards the log data to the specified destination, in this case the route to Logstash that we have created.

    cf cups digital-billing-log-drain -l https://<user>:<password>@digital-billing-logstash-apc.scapp-corp.acme.com

The values for "user" and "password" must correspond to what is defined in the HTTP input plugin of the Logstash pipeline.

"cf cups" is short for "cf create-user-provided-service".


The Logstash Pipeline
=====================
The file [`digital-billing-apc.conf`](https://github.com/StaticNoiseLog/logstash-cloud-foundry-config/blob/master/digital-billing-apc.conf) is a real-world example of a Logstash pipeline that works with Cloud Foundry log drains. Versions used were Logtash 7.3.1 and Cloud Foundry 2.153.0 (`cf api`). Some key points of this pipeline are:

* The filter plugin part is generic and should work for any project.
* The most relevant (earliest) timestamp is extracted from each unit of log data and assigned to `@timestamp`. This should result in the most accurate sequence of events possible.
* Only fields that seem to be of practical use when monitoring apps are stored in the Elasticsearch document, but the full and unaltered log data is always available as `raw_message`.
* It was necessary to make case differentiations for the Cloud Foundry components APP, RTR, API and STG because they either log special data or they put the same information in a different place in the log data.
* `tag_on_failure` is used everywhere so that problems with the pipeline itself can be discovered. A Kibana filter "field: tags, Operator: is, Value: failed" should work.

Debugging the Pipeline
----------------------
To see what errors prevent your pipeline from running, you can observe the log output of Logstash. The Gorouter (RTR) produces a lot of output so you may have to suppress that. Here is how this can be done in the Windows CMD shell:

    cf logs digital-billing-logstash | findstr /V RTR

Grok
----
A Grok expression finds the first match and that becomes the "current position". The following expression then moves on *from the current position* to find the next match. You can force each expression to start matching at the beginning of the line using the `^` symbol, but the default behavior is to keep moving forward, match by match.

The **Grok Debugger** is helpful for developing Grok expressions. It is available online, but it is better to use the one integrated in the Kibana GUI (Dev Tools).

Avoid the confusion with quotes: Put the pure log data in the "Sample Data" field of the Grok debugger, as you can see 
it with `cf logs digital-billing-logstash`. When you have successfully extracted a segment of the log message in the Grok debugger, that segment will be displayed as a value of a JSON document. Therefore the extracted segment will have its quotes escaped with a backslash! To continue analyzing that segment in the Grok debugger with a another expression, I recommend removing the inserted backslashes (or extract the segment manually from the original log data).

Once you have created a valid Grok match expression with the Grok debugger, you will want to use it in a pipeline. For this it must be written as a string, i.e. surrounded by quotes. *If the Grok expression itself already contains quotes they must now be escaped with a backslash!*

### Grok Example 1 ###
Match from the current position up to and including the first `space_name="`, without assigning the match to a field (skip over this match):

    (.*?space_name=")?

The trailing question mark `?` makes the match optional: If `space_name="` is not found, the current position remains unchanged, no error condition. Without the question mark you would get a "patterns do not match data in the input" error and Grok aborts.  
Note that this expression contains a quote `"` which must be escaped with a backslash as `\"` when using it in the pipeline.

### Grok Example 2 ###
Match from the current position up to the first quote `"` character (excluding) and assign it to the field `environment`:

    (?<environment>[^"]*)?

The regular expression `[^"]` simply means "the set of all characters that are *not* (`^`) a quote.  
The trailing `?` makes the match optional and the contained `"` must be escaped when the expression is used in a string for a pipeline.

### Grok Example 3 ###
Match from the current position up to and *excluding* the first appearance of `payload` and assign it to the field `main_message`.

    (?<main_message>.*?(?=payload))?

The `?=` is the so called "lookahead" feature of regex. It asserts that what immediately follows the current position is the string `payload`.  
The trailing `?` makes the match optional.

You can find more information on regex lookahead and lookbehind [here](https://www.rexegg.com/regex-lookarounds.html).


Cleaning Up
===========

Unused Log Drains
-----------------
Use `cf services` to check whether you have unused log drains. If you are not sure whether a service is a log drain, you can use the following command. It returns the details of all services from all spaces. Log drains have an attribute `syslog_drain_url`, which is the destination to which the log lines are forwarded.

    cf curl /v3/service_instances

Unbind any apps still associated with the log drain service and then delete it in the Cloud Foundry web GUI or with this command:

    cf delete-service name-of-log-drain

Ports Used by Logstash
----------------------
To decide if you can delete a route to a port, you need to know on which ports Logstash is listening.

Get the GUID of the Logstash app:

    cf app digital-billing-logstash --guid
    9ba9860b-54d0-4ddb-ba0a-c30192890fc6

If everything was set up properly, you can find all ports in the details of the Logstash app. Note that 5044 and 9600 are default ports.

    cf curl /v2/apps/9ba9860b-54d0-4ddb-ba0a-c30192890fc6
    ...
      "ports": [
         5044,
         9600,
         12316    <<---- candidate for removal.
      ],
    ...

To make sure you can also query the routes details and look for the Logstash GUID in the output (in "destinations"). Besides the default ports 5044 and 9600, are there any other ports where a route tries to reach Logstash?

    cf curl /v3/routes?per_page=1000

Port Not Needed Anymore in Logstash
-----------------------------------
You do not need port 12316 anymore in any Logstash pipeline and want to clean up.

GUID of the logstash app:

    cf app digital-billing-logstash --guid
    9ba9860b-54d0-4ddb-ba0a-c30192890fc6

Find all routes that map to the Logstash app on port 12316:

    cf curl /v3/routes?per_page=1000

In the output look for "app.guid" and "port" under "destinations" and note the GUID of the route-mapping and the route itself:

      {
         "guid": "4a0087ae-8267-42f3-bd55-3bd41b040bd6",           <<---- GUID of route
         "created_at": "2020-08-17T18:23:17Z",
         "updated_at": "2020-08-17T18:23:17Z",
         "protocol": "tcp",
         "host": "",
         "path": "",
         "port": 12316,
         "url": "tcp-scapp-corp.acme.com:12316",
         "destinations": [
            {
               "guid": "45612786-5b64-47f7-8788-9eabb3dc3298",     <<---- GUID of route-mapping
               "app": {
                  "guid": "9ba9860b-54d0-4ddb-ba0a-c30192890fc6",  <<---- GUID of Logstash app MATCH
                  "process": {
                     "type": "web"
                  }
               },
               "weight": null,
               "port": 12316                                       <<---- destination port MATCH
            }
         ],

Delete the route-mapping and then the route itself using their respective GUIDs:

    cf curl /v2/route_mappings/45612786-5b64-47f7-8788-9eabb3dc3298 -X DELETE
    cf curl /v3/routes/4a0087ae-8267-42f3-bd55-3bd41b040bd6 -X DELETE

Now you can remove the unwanted port from the Logstash app. 
For this you must first get all ports in use by the app:

    cf curl /v2/apps/9ba9860b-54d0-4ddb-ba0a-c30192890fc6
    ...
      "ports": [
         5044,
         5045,
         9600,
         37363,
         39337,
         12316    <<---- the only port NOT needed
      ],
    ...

Then you do a PUT *preserving all the other ports* and omitting only the one you want to get rid off:

    cf curl /v2/apps/9ba9860b-54d0-4ddb-ba0a-c30192890fc6 -X PUT -d "{\"ports\": [5044,5045,9600,37363,39337]}"

Finally delete the Logstash pipeline that expected input on port 12316. This can be done in the Kibana GUI.

Goodbye ELK
-----------
Deleting the entire ELK stack deployment can be done mostly in the web GUI.

After deleting any log drains, the Elasticsearch service as well as the Logstash and Kibana apps, use `cf routes` to check for orphaned routes and delete them. Here is an example:

    cf delete-route scapp-corp.acme.com --hostname digital-billing-kibana-test
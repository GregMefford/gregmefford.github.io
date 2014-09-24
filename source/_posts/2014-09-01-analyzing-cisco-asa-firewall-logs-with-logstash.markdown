---
layout: post
title: "Analyzing Cisco ASA Firewall Logs with Logstash"
date: 2014-09-01 22:14:01 -0400
comments: true
categories: 
- Logstash
- ASA
- Firewall
- Log Analysis
---

A year ago, I had a need to collect, analyze, and archive firewall logs from several Cisco ASA appliances.
The problem with Cisco's ASA syslog format is that each type of message is a special snowflake, apparently designed for human consumption rather than machine parsing.
The obvious solution was, of course, to use what is now know as [the ELK stack][elk]: ElasticSearch, Logstash, and Kibana.
In this article, I'll walk through the process I used to tackle this problem, ending with a full configuration file that you can use in your environment.

[elk]: http://www.elasticsearch.org/overview/

<!-- more -->

## TL;DR

If you're interested in the details about how this configuration was developed, read more below. If you're just here for the final configuration file, here it is:

``` text cisco-asa.conf
input {
  udp {
    port => 514
    type => "cisco-asa"
  }
}

filter {
  if type == "cisco-asa" {
    # Pull the syslog part and Cisco tag out of the message
    grok {
      match => ["message", "%{CISCO_TAGGED_SYSLOG} %{GREEDYDATA:cisco_message}"]
    }

    # Parse the syslog severity and facility
    syslog_pri { }

    # Extract fields from the each of the detailed message types
    # The patterns provided below are included in Logstash since 1.2.0
    grok {
      match => [
        "cisco_message", "%{CISCOFW106001}",
        "cisco_message", "%{CISCOFW106006_106007_106010}",
        "cisco_message", "%{CISCOFW106014}",
        "cisco_message", "%{CISCOFW106015}",
        "cisco_message", "%{CISCOFW106021}",
        "cisco_message", "%{CISCOFW106023}",
        "cisco_message", "%{CISCOFW106100}",
        "cisco_message", "%{CISCOFW110002}",
        "cisco_message", "%{CISCOFW302010}",
        "cisco_message", "%{CISCOFW302013_302014_302015_302016}",
        "cisco_message", "%{CISCOFW302020_302021}",
        "cisco_message", "%{CISCOFW305011}",
        "cisco_message", "%{CISCOFW313001_313004_313008}",
        "cisco_message", "%{CISCOFW313005}",
        "cisco_message", "%{CISCOFW402117}",
        "cisco_message", "%{CISCOFW402119}",
        "cisco_message", "%{CISCOFW419001}",
        "cisco_message", "%{CISCOFW419002}",
        "cisco_message", "%{CISCOFW500004}",
        "cisco_message", "%{CISCOFW602303_602304}",
        "cisco_message", "%{CISCOFW710001_710002_710003_710005_710006}",
        "cisco_message", "%{CISCOFW713172}",
        "cisco_message", "%{CISCOFW733100}"
      ]
    }

    # Parse the date
    date {
      match => ["timestamp",
        "MMM dd HH:mm:ss",
        "MMM  d HH:mm:ss",
        "MMM dd yyyy HH:mm:ss",
        "MMM  d yyyy HH:mm:ss"
      ]
    }
  }
}

output {
  elasticsearch_http { host => "elasticsearch-server-name" }
}
```

## Surely This is a Solved Problem

After doing some Google searches, I found that several people were trying to do this with varying levels of success, but no one had yet documented even an 80% solution.
The best I could find was [a gist by dav3860][dav3860_gist], which gave me a great start on how to parse some of the [many, many Cisco ASA syslog message formats][asa_syslog].
Unfortunately, this gist didn't cover many of the message formats that I wanted to analyze from my environment and didn't capture all of the available data fields from the messages.
Seeing that there was demand for it on the mailing list and no one else seemed to have a good solution, I took it upon myself to do it myself.

[dav3860_gist]: https://gist.github.com/dav3860/5345656
[asa_syslog]: http://www.cisco.com/en/US/docs/security/asa/syslog-guide/logmsgs.html

## Stand Back, I Know Regular Expressions

iI started by collecting a large-ish sample of about half a million events from my production environment.
The ASA Device Manager (ASDM) software made this process pretty easy, if a bit tedious, by providing a file manager to download the log files from its internal Flash memory.
This allowed me to iteratively develop Grok expression against a known corpus of data.
When I hit an edge-case that failed to parse for some reason, I could tweak the expression and try again with the same input.

If you're here trying to figure out how to parse your Cisco ASA logs, you've probably already seen what they look like.
In case you haven't, and for the sake of discussion, here's a little taste (obviously contrived and anonymized):

```
<134>Sep 02 2014 11:50:10: %ASA-6-302013: Built inbound TCP connection 123456789 for inside:10.0.1.1/1234 (10.0.1.1/1234) to outside:10.0.2.2/80 (10.0.2.2/80)
<134>Sep 02 2014 11:50:10: %ASA-6-302014: Teardown TCP connection 123456789 for inside:10.0.1.1/1234 to outside:10.0.2.2/80 duration 0:00:00 bytes 420 TCP FINs
<134>Sep 02 2014 11:50:15: %ASA-6-106015: Deny TCP (no connection) from 10.0.1.1/4567 to 10.0.3.3/80 flags FIN ACK  on interface inside
<163>Sep 02 2014 11:50:25: %ASA-3-710003: TCP access denied by ACL from 10.0.4.4/6666 to outside:10.0.1.1/22
```

Just from peering dubiously at these logs for a minute, I was able to divine a few things:

  * There's a "standard" syslog prefix at the beginning with a severity code and a timestamp.
  * The timestamp is in some random format, and it's in local time instead of UTC. ::sigh::
  * There's some kind of Cisco proprietary code at the beginning that I'm going to call a `ciscotag` for want of a better name.
  * After the `ciscotag`, there's a human-readable message containing the meat and potatoes that we want to parse.

With a little [Grok-fu][grok], I was able to build the following patterns file to match the beginning part of the messages:

[grok]: http://logstash.net/docs/latest/filters/grok

``` text patterns/patterns.txt
CISCO_TAGGED_SYSLOG ^<%{POSINT:syslog_pri}>%{CISCOTIMESTAMP:timestamp}( %{SYSLOGHOST:sysloghost})?: %%{CISCOTAG:ciscotag}:
CISCOTIMESTAMP %{MONTH} +%{MONTHDAY}(?: %{YEAR})? %{TIME}
CISCOTAG [A-Z0-9]+-%{INT}-(?:[A-Z0-9_]+)
```

The keen-eyed reader will notice that these patterns don't match my logs identically.
For example, my logs don't have a `SYSLOGHOST`, and I have the `YEAR` marked as optional even though it's there in my logs.
Based on feedback from other people who tried to use my patterns, I learned that the ASA syslog output can vary based on firmware version and some config settings. 
The awesome power of Grok patterns is that it's pretty easy to tweak them to work flexibly for everyone without making a tangled mess.

## Now We're Getting Somewhere

At this point, we have enough to actually run the ELK stack to guide the rest of the development.
The following is a simple "test fixture" setup that I found useful during this process.
I'm going to assume that you're trying to accomplish this task using a Windows machine, but it's pretty easy to tell how the same would work in Bash.
Let's be honest. There are too few tutorials that help people be productive with tools like this on Windows.

First, download [the latest Logstash core zip file (currently 1.4.2)][logstash_1.4.2] and unpack it somewhere convenient.

[logstash_1.4.2]: https://download.elasticsearch.org/logstash/logstash/logstash-1.4.2.zip

Then create the a configuration file called `logstash.conf` and a batch file called `run_logstash.bat` containing the following:

``` text logstash.conf
input {
  stdin { }
}

filter {
  grok {
    patterns_dir => ".\patterns\" # The directory containing the patterns file mentioned above
    match => ["message", "%{CISCO_TAGGED_SYSLOG} %{GREEDYDATA:cisco_message}"]
  }

  date {
    match => ["timestamp",
      "MMM dd HH:mm:ss",
      "MMM  d HH:mm:ss",
      "MMM dd yyyy HH:mm:ss",
      "MMM  d yyyy HH:mm:ss"
    ]
  }

  syslog_pri { }
}

output {
  stdout { codec => "rubydebug" } 
  if "_grokparsefailure" in [tags] {
    file {
      path => "./failure.json"
    }
  }
  else {
    file {
      path => "./success_%{ciscotag}.json"
    }
  }
}
```

``` bat run_logstash.bat
logstash-1.4.2\bin\logstash.bat agent -f logstash.conf
```

When you execute `run_logstash.bat`, Logstash will fire up and wait for input on `STDIN`.
When you paste a set of events into the console, they will be processed and the results will be displayed on the screen as well as being appended to the specified files.

I had a big sample of event data that I had collected, so it resulted in a bunch of `success_<something>.json` files of varying sizes.
Each of these files contained the JSON-ified version of each event.
For example, the following would show up on the console, with the same being appended to the `success_ASA-3-710003.json` file (except on one line per event):

``` console
{
                 "message" => "<134>Sep 02 2014 11:50:10: %ASA-6-302013: Built inbound TCP connection 123456789 for inside:10.0.1.1/1234 (10.0.1.1/1234) to outside:10.0.2.2/80 (10.0.2.2/80)\r",
                "@version" => "1",
              "@timestamp" => "2014-09-02T15:50:10.000Z",
                    "host" => "Forklift",
              "syslog_pri" => "134",
               "timestamp" => "Sep 02 2014 11:50:10",
                "ciscotag" => "ASA-6-302013",
           "cisco_message" => "Built inbound TCP connection 123456789 for inside:10.0.1.1/1234 (10.0.1.1/1234) to outside:10.0.2.2/80 (10.0.2.2/80)\r",
    "syslog_severity_code" => 6,
    "syslog_facility_code" => 16,
         "syslog_facility" => "local0",
         "syslog_severity" => "informational"
}
```

Looking at the relative sizes of the resulting files, I could make inferences about the log volume of each type, allowing me to prioritize which patterns to focus on first.
The `ASA-6-302013` event is logged whenever a connection is successfully established across the firewall, so that tends to be pretty frequent.
By adding the following Grok filter to the filter list, I was able to easily parse out all of the relevant fields:

``` text logstash.conf
# ...
filter {
# ...

  syslog_pri { }

  grok {
    match => [
      "cisco_message",
      "Built inbound TCP connection %{INT:connection_id} for %{DATA:src_interface}:%{IP:src_ip}/%{INT:src_port}( \(%{IP:src_mapped_ip}/%{INT:src_mapped_port}\))? to %{DATA:dst_interface}:%{IP:dst_ip}/%{INT:dst_port}( \(%{IP:dst_mapped_ip}/%{INT:dst_mapped_port}\))?"
    ]
  }
}
# ...
```

Which results in the following magical output:

``` console
{
                 "message" => "<134>Sep 02 2014 11:50:10: %ASA-6-302013: Built inbound TCP connection 123456789 for inside:10.0.1.1/1234 (10.0.1.1/1234) to outside:10.0.2.2/80 (10.0.2.2/80)\r",
                "@version" => "1",
              "@timestamp" => "2014-09-02T15:50:10.000Z",
                    "host" => "Forklift",
              "syslog_pri" => "134",
               "timestamp" => "Sep 02 2014 11:50:10",
                "ciscotag" => "ASA-6-302013",
           "cisco_message" => "Built inbound TCP connection 123456789 for inside:10.0.1.1/1234 (10.0.1.1/1234) to outside:10.0.2.2/80 (10.0.2.2/80)\r",
    "syslog_severity_code" => 6,
    "syslog_facility_code" => 16,
         "syslog_facility" => "local0",
         "syslog_severity" => "informational",
           "connection_id" => "123456789",
           "src_interface" => "inside",
                  "src_ip" => "10.0.1.1",
                "src_port" => "1234",
           "src_mapped_ip" => "10.0.1.1",
         "src_mapped_port" => "1234",
           "dst_interface" => "outside",
                  "dst_ip" => "10.0.2.2",
                "dst_port" => "80",
           "dst_mapped_ip" => "10.0.2.2",
         "dst_mapped_port" => "80"
}
```

After figuring out several such Grok patterns and consulting with the documentation, I discovered that many of the formats are pretty similar to one another.
For those, I broke out the common fragments from each message and combined the similar Grok expressions into a pattern file.
The format of the pattern file is the same as what would go in the `match` clause of the `grok` filter, except with an `ALL_CAPS_NAME` at the beginning of each line.

Here's a snippet of the resulting patterns file in all its complex glory, but only for the demo messages we're discussing here.
These patterns and a bunch more are already included in Logstash, so you won't need to include them in your own pattern files.

``` text patterns/patterns.txt
#== Cisco ASA ==
CISCO_TAGGED_SYSLOG ^<%{POSINT:syslog_pri}>%{CISCOTIMESTAMP:timestamp}( %{SYSLOGHOST:sysloghost})?: %%{CISCOTAG:ciscotag}:
CISCOTIMESTAMP %{MONTH} +%{MONTHDAY}(?: %{YEAR})? %{TIME}
CISCOTAG [A-Z0-9]+-%{INT}-(?:[A-Z0-9_]+)

# Common Particles
CISCO_ACTION Built|Teardown|Deny|Denied|denied|requested|permitted|denied by ACL|discarded|est-allowed|Dropping|created|deleted
CISCO_REASON Duplicate TCP SYN|Failed to locate egress interface|Invalid transport field|No matching connection|DNS Response|DNS Query|(?:%{WORD}\s*)*
CISCO_DIRECTION Inbound|inbound|Outbound|outbound

# ASA-6-106015
CISCOFW106015 %{CISCO_ACTION:action} %{WORD:protocol} \(%{DATA:policy_id}\) from %{IP:src_ip}/%{INT:src_port} to %{IP:dst_ip}/%{INT:dst_port} flags %{DATA:tcp_flags}  on interface %{GREEDYDATA:interface}

# ASA-6-302013, ASA-6-302014, ASA-6-302015, ASA-6-302016
CISCOFW302013_302014_302015_302016 %{CISCO_ACTION:action}(?: %{CISCO_DIRECTION:direction})? %{WORD:protocol} connection %{INT:connection_id} for %{DATA:src_interface}:%{IP:src_ip}/%{INT:src_port}( \(%{IP:src_mapped_ip}/%{INT:src_mapped_port}\))?(\(%{DATA:src_fwuser}\))? to %{DATA:dst_interface}:%{IP:dst_ip}/%{INT:dst_port}( \(%{IP:dst_mapped_ip}/%{INT:dst_mapped_port}\))?(\(%{DATA:dst_fwuser}\))?( duration %{TIME:duration} bytes %{INT:bytes})?(?: %{CISCO_REASON:reason})?( \(%{DATA:user}\))?

# ASA-7-710001, ASA-7-710002, ASA-7-710003, ASA-7-710005, ASA-7-710006
CISCOFW710001_710002_710003_710005_710006 %{WORD:protocol} (?:request|access) %{CISCO_ACTION:action} from %{IP:src_ip}/%{INT:src_port} to %{DATA:dst_interface}:%{IP:dst_ip}/%{INT:dst_port}

#== End Cisco ASA ==
```

Now, we can use the pattern file to extract the complexity from the `logstash.conf` configuration file.
The previous configuration snippet can now be replaced by this one:

``` text logstash.conf
# ...
filter {
# ...

  syslog_pri { }

  grok {
    patterns_dir => "./patterns/"
    match => [
      "cisco_message", "%{CISCOFW106015}",
      "cisco_message", "%{CISCOFW302013_302014_302015_302016}",
      "cisco_message", "%{CISCOFW710001_710002_710003_710005_710006}",
    ]
  }
}
```

This results in the same output as above, only now the Grok filter will try to match against each of the requested patterns until one of them succeeds.

## Cleaning Up



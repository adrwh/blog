---
title: "Creating Watchers in Elasticsearch to catch Domain Admin security group changes"
date: 2020-01-23T21:27:54+11:00
description: ""
ghissueid: 2
tags: ["elasticsearch","watcher","alerting","security"]
draft: false
---

We are going to setup an alert for whenever our Domain Admin group is changed.  This is great for security context as well as general auditing.

> Please note: This is not a defender safeguard to detect privilege escalation.  Smart attackers will not be adding themselves to the Domain Admin group.

First create your search using Kibana.  In this example i am going to search for changes to the Domain Admins security group.  In the Kibana search bar, type `winlog.event_data.TargetSid:S-1-5-21-your domain sid-512 and event.code: 4737`

AD Global Security Group changes are audited in EventID 4737, and the Domain Admins group SID can be found with `Get-ADGroup "Domain Admins"`. More information here <https://support.microsoft.com/en-au/help/243330/well-known-security-identifiers-in-windows-operating-systems>

Click the Inspect link.  This will pop-out a new window showing details about the search statistics, the request and the response.  Click on the Request tab, which will show you the raw JSON used to make the request.  Technically you can save this to file and make the same search using the CLI, Curl etc.  Now scroll down the JSON request, look for the `query: {}` object and copy it.  Armed with the search query JSON object we are now ready to check out how the Watcher API works.

Read about Elastic Watcher API here <https://www.elastic.co/guide/en/elasticsearch/reference/current/watcher-api-put-watch.html>

So to create a new watcher we have to send a `PUT` request with some `parameters` and a `body` to `_watcher/watch/<watch_id>` where the `watch_id` is the name or identifier we choose.  The request `body` needs a JSON document with a `trigger`, `input`, `condition` and `actions` objects.

We want the watcher to run every 5 minutes, then look back over the last 5 minutes and search for 1 or more hits, using the search query we built earlier.  Then we want it to perform an action like, send an email to the administrator or post a message to our Team.

In this example i am going to use the following criteria to build my watcher `body` object..

| Title     | Description     |   |
|-----------|-----------------|---|
| trigger   | every 5 minutes |   |
| input     | if the Domain Admins group is changed           |   |
| condition | if results greater than 0            |   |
| actions   | send a message to MS Teams          |   |
|           |                 |   |

Here is the full request that will create my watcher.

```sh
PUT _watcher/watch/domain_admin_group_was_changed // <-- this is the name/id for your watcher.
{
  "trigger": {
    "schedule": {
      "interval": "5m"  // < -- I want the watcher to run every 5 minutes
    }
  },
  "input": {
    "search": {
      "request": {
        "indices": [
          "winlogbeat-*" // <-- set the index you want to search
        ],
        "body": {
          "query": {  // <-- This is the query you copied from the kibana search earlier, paste it here.
            "bool": {
              "must": [],
              "filter": [
                {
                  "match_all": {}
                },
                {
                  "match_phrase": {
                    "winlog.event_data.TargetSid": {
                      "query": "S-1-5-21-nnnn-512" // <-- replace this with your Domain Admins SID
                    }
                  }
                },
                {
                  "match_phrase": {
                    "winlog.event_id": {
                      "query": 4737
                    }
                  }
                },
                {
                  "range": { // <-- Search back the last 5 minutes
                    "@timestamp": {
                      "from": "now-5m",
                      "to": "now"
                    }
                  }
                }
              ],
              "should": [],
              "must_not": []
            }
          }
        }
      }
    }
  },
  "condition": {
    "compare": {
      "ctx.payload.hits.total": {
        "gt": 0
      }
    }
  },
  "actions": {
    "teams_webhook": {
      "transform": {
        "script": { // <-- This script is needed to format the message to meet the MS Teams message requirements.
          "source": """
            def pl = ctx.payload;
            pl["@type"] = 'MessageCard';
            pl["@context"] = 'http://schema.org/extensions';
            pl.themeColor = 'ff0000';
            pl.summary = "The Summary";
            pl.title = '4737 The Domain Admin group has changed!';
            return pl; 
""",
          "lang": "painless"
        }
      },
      "webhook": {
        "scheme": "https",
        "host": "outlook.office.com",
        "port": 443,
        "method": "post",
        "path": "/your-teams-webhook-id", // <-- put your own Teams webhook ID here.
        "params": {},
        "headers": {
          "Content-Type": "application/json"
        },
        "body": "{{#toJson}}ctx.payload{{/toJson}}"
      }
    }
  }
}
```

Now send this PUT request to your Elastcsearch service and the watch is created.

The next time, a change is made to your Domain Admins group, you will get a message like this in Teams.

![domain-admin-was-changed-teams-alert.png](/img/domain-admin-was-changed-teams-alert.png)

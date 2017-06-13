1.  After the stack comes up, open localhost:9200 in a browser and verify that the cluster is healthy
2.  visit localhost:8080 and refresh a few times.  Return to kopf.  2 Indexes should have been created.
3.  Open localhost:5601 to open kibana.  Add a index pattern logstash*.  You should now be able to search in kibana
4.  Play around with visiting the apache test page and generating searches
5.  You can also visit localhost:1936 to see the admin UI of haproxy.
6.  (Optional) Try to add syslog messages and cron messages to the vagrant vm's /etc/logstash/conf.d/logstash.conf file.  An example of what that looks like can be found in this directory.  

EX 1 - Mapping Templates and Aliases

In kibana, create a new Pie Chart visualization against the logstash* index pattern.  Split the slices against the term request.  Notice that the request path was split on the '/' by elasticsearch by default.  This isn't what we want.
We need to fix that field to not delimit on '/' while still preserving the logs we have.

1.  In KOPF, in the cluster view, select the accesslog index dropdown and choose show mappings.  Copy the mappings to your clipboard
2.  Go to the index templates section in the more dropdown
3.  On the left side, give the mapping template a name like "accesslogs"
4.  In the JSON editor section, create the following:

  "template": "logstash-access*",
  "settings": {},
  "mappings": {
    "_default_": {
      "_all": {
        "enabled": false
      },
      "properties": {
        "request": {
          "index": "not_analyzed",
          "type": "string"
        },
        "auth": {
          "type": "string"
        },
        "ident": {
          "type": "string"
        },
        "verb": {
          "type": "string"
        },
        "message": {
          "type": "string"
        },
        "type": {
          "type": "string"
        },
        "tags": {
          "type": "string"
        },
        "indexed_by": {
          "type": "string"
        },
        "path": {
          "index": "not_analyzed",
          "type": "string"
        },
        "@timestamp": {
          "format": "strict_date_optional_time||epoch_millis",
          "type": "date"
        },
        "bytes": {
          "type": "integer"
        },
        "response": {
          "type": "string",
          "index": "not_analyzed"
        },
        "clientip": {
          "index": "not_analyzed",
          "type": "string"
        },
        "@version": {
          "type": "string"
        },
        "host": {
          "index": "not_analyzed",
          "type": "string"
        },
        "httpversion": {
          "type": "string"
        },
        "timestamp": {
          "type": "string"
        }
      }
    }
  },
  "aliases": {
    "alias-accesslog": {}
  }
}

  In addition to making the "request" field not analyzed, I've included a few others that shouldn't be delimited. I also made bytes an integer (so it can be summed).  At the very top, I set a pattern for matching indexes.  At the very bottom, I set an alias for indexes like this, such that they belong to an alias called "alias-accesslog"

5.  Save that template.  Note that at this point, nothing will change.  You'd have to wait until the daily index rolls over (in 24 hours) before the mapping template would apply.
6.  We want to apply the mapping template now, and fix the old data.  There's a plugin called elasticsearch-reindexing that makes the reindexing process a bit easier.
7.  From the vagrant machine run curl -XPOST 'http://localhost:9200/logstash-accesslog-2017.01.30/_reindex/logstash-accesslog-2017.01.30-1' 
8.  Open KOPF cluster view and you should see a new index has been created with the same number of documents as the original.  Note that the newly created index also matches the pattern set in the JSON above.
9.  From KOPF, delete the origin logstash-accesslog-2017.01.30 index, since we have it reindexed in logstash-accesslog-2017.01.30-1
10. Go back to the pie chart visualization.  Note that the field is not broken up by '/'.
11. Visit localhost:8080 in your browser again and check KOPF.  You'll see that an index was created.  You'll also see 2 indexes that have an alias tag under them.
12. In kibana, in settings, add a new Index pattern, and type in alias-accesslog, and save.
13. Searching against alias-accesslog searches across the reindexed older logs and the new logs, both of which have themapping template applied.


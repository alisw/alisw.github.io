---
title: Build logs processing
layout: main
categories: infrastructure 
---

The {{site.experment}} build infrastructure uses the [ELK](https://kibana.com)
(Elasticsearch, Logstash, Kibana) stack to aggregate, parse, index and data
mine build logs.

The [logstash jenkins
plugin](https://wiki.jenkins-ci.org/display/JENKINS/Logstash+Plugin) pushes
each line of the logs to a REDIS instance which is used as a buffer for
logstash analysis. This will eventually allow scaling of the infrastructure by
having multiple logstash entries pulling from REDIS.

The logstash configuration file is maintained in
<https://github.com/alisw/ali-bot/tree/master/logstash>. Each line of the logs
are trated as a separate event and can be found in `[message][0]`. In order to
deploy a new configuration you need to commit it to the master branche of the
github repository, and then restart the logstash entry using [ALICE Marathon
instance](http://alimarathon.cern.ch) only visible from inside CERN.

Events which pass the logstash filter are pushed to [ALICE elasticsearch
instance](http://elasticsearch.marathon.mesos:9200), which is only visible from
inside the build cluster and eventually, where it makes sense, to Monalisa.

A Jenkins job
[generate-web-pages](https://alijenkins.cern.ch/job/generate-web-pages/) will
then periodically query the elasticsearch instance, using the script
[get_builds](https://github.com/alisw/ali-bot/blob/master/script/get_builds.sh)
and populate relevant web pages in <https://ali-ci.cern.ch> or its Github
mirror <http://alisw.github.io>. Query results are stored as JSON files in the
[es](https://github.com/alisw/ali-bot/blob/master/es)
directory and Jekyll Pages are populated using the data provided there.

Finally, for quick data-mining, a Kibana 4 interface is provided at
<http://kibana.marathon.mesos:5601>, again only visible from inside the build
cluster.

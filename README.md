# Foreman/Katello/Satellite and ElasticSearch

Configuration for foreman-journald-rsyslog-elasticsearch logging setup. Works
with Katello or Satellite as well. Initially these instructions were published
in a Red Hat knowledgebase article: https://access.redhat.com/solutions/3545571

These instructions will be converted to Ansible or Puppet (installer) later
on.

This has been tested with the following versions:

* Foreman 1.22
* Satellite 6.6

## Instructions

TODO:

* finish document and ask for review
* document audit.log
* send candlepin to production.log as well
* add field foreman.app
* make field table script from YAML
* move the instructions from gdoc to this README

Candlepin:

    <appender name="SyslogAppender" class="net.logstash.logback.appender.LogstashSocketAppender">
        <host>127.0.0.1</host>
        <port>514</port>
        <encoder class="net.logstash.logback.encoder.LogstashEncoder" />
        <prefix class="ch.qos.logback.classic.PatternLayout">
            <pattern>&lt;34&gt;1 - localhost candlepin - - - @cee:</pattern>
        </prefix>
    </appender>



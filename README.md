# Foreman/Katello/Satellite and ElasticSearch

Configuration for foreman-journald-rsyslog-elasticsearch logging setup. Works
with Katello or Satellite as well. Initially these instructions were published
in a Red Hat Knowledgebase solution: https://access.redhat.com/solutions/3545571

These instructions will be converted to Ansible or Puppet (installer) later
on.

This has been tested with the following versions:

* Foreman 1.22
* Satellite 6.6

## System journal metadata

Foreman core (Rails app), Smart Proxy and Dynflow leverage journald with the following metadata:

Name | Description
------ | ------------
LOGGER | Logger name (see below)
USER_LOGIN | User login name
ORG_ID | Organization database ID
LOC_ID | Location database ID
ORG_NAME | Organization name
LOC_NAME | Location name
ORG_LABEL | Organization label
LOC_LABEL | Location label
REMOTE_IP |  Remote IP address of a client
REQUEST |  Request ID generated by ActionDispatch
SESSION |  Random ID generated per session or request for session-less request
AUDIT_ACTION | Audit action: create, update or delete (audit logger)
AUDIT_TYPE | Audit resource type, e.g. Host, Subnet or ContentView (audit logger)
AUDIT_ID | Audit resource database ID as a number  (audit logger)
AUDIT_ATTRIBUTE | Audit resource field or database column that changed (audit logger)
AUDIT_FIELD_OLD | Old audit value for update action (audit logger)
AUDIT_FIELD_NEW | New audit value  (audit logger)
EXCEPTION_MESSAGE |  Exception message when error is logged
EXCEPTION_CLASS |  Exception Ruby class when error is logged
EXCEPTION_BACKTRACE |  Exception backtrace as a multiline string when error is logged
TEMPLATE_NAME |  Template name (blob logger)
TEMPLATE_DIGEST |  Digest (SHA256) of rendered template contents (blob logger)
TEMPLATE_HOST_NAME | Host name for a rendered template if present (blob logger)
TEMPLATE_HOST_ID | Host database ID for a rendered template if present (blob logger)
AUDIT_ACTION | Action performed (e.g. create/update/delete)
AUDIT_TYPE | Database model class or type, subject of an audit record (e.g. Hostgroup or Subnet)
AUDIT_ID | Record database ID of the audit subject
AUDIT_ATTRIBUTE | Attribute name or column an action was performed on (e.g. name or description)

Logger name is only set by Foreman core (Rails) application. Current list of
loggers enabled by default can be found with:

    grep add_loggers -A14 /usr/share/foreman/config/application.rb

Name | Description
---- | -----------
app | Main application logger
audit | Contains copies of audit events available in Monitor - Audit
ldap | Detailed logs from LDAP communication
permissions | Detailed debugging information from permission subsystem
proxy | Information from proxy communication
sql | All SQL queries performed by the system (verbose)
templates | Information from template renderer component
notifications | Information from notifications component
background | Information from background processing component
dynflow | Information from Dynflow process
telemetry | Debugging information from telemetry
blob | Contents of rendered templates for auditing purposes (might contain sensitive data)

All fields are forwarded from journald to rsyslog for further processing.

## Elasticsearch metadata

Rsyslog performs data transformation and sends entries to Elasticsearch in the
Red Hat Common Data model. Documentation for all JSON fields can be found in
[org.foreman.viaq-cdm.asciidoc](org.foreman.viaq-cdm.asciidoc).

## Configuring Elasticsearch and Kibana

The following part describes quick installation of Elasticsearch and Kibana.
Both components must be installed on a separate server and they are not
provided or supported by Red Hat. In this article, the Foreman or the Satellite
host is named `server`, and the ElasticSearch host is named `elastic`.

ElasticSearch version 5.x is required because Elastic version 6.x does not work
with rsyslog. For more information on the issue, see:
[BZ#1600171](https://bugzilla.redhat.com/show_bug.cgi?id=1600171). Also the
operating system was Red Hat Enterprise Linux 7.6.

    elastic# wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-5.6.16.rpm
    elastic# wget https://artifacts.elastic.co/downloads/kibana/kibana-5.6.16-x86_64.rpm

Install ElasticSearch, then enable and start the service. By default
ElasticSearch listens only on localhost, configure either an external IP address
which is reachable from Satellite Server or listening on all interfaces:

    elastic# rpm -i elasticsearch-5.6.16.rpm

    elastic# grep network.host /etc/elasticsearch/elasticsearch.yml
    network.host: 0.0.0.0

    elastic# systemctl daemon-reload
    elastic# systemctl enable elasticsearch.service
    elastic# systemctl start elasticsearch.service

Install the Kibana UI, configure external IP address in the same way and then
enable and start the service.

    elastic# rpm -i kibana-5.6.16-x86_64.rpm

    elastic# grep server.host /etc/kibana/kibana.yml
    server.host: "0.0.0.0"

By default, Kibana expects ElasticSearch running on localhost port 9200 which is
sufficient if it was configured to listen on all interfaces. In case it was
configured for a particular IP address, change the configuartion:

    elastic# grep elasticsearch.url /etc/kibana/kibana.yml
    elasticsearch.url: "http://1.2.3.4:9200"

And restart the service:

    elastic# systemctl daemon-reload
    elastic# systemctl enable kibana.service
    elastic# systemctl start kibana.service

Configure the firewall to allow the server to upload logs on port TCP 9200 and
users to access Kibana Web interface on port TCP 5601.

    elastic# firewall-cmd --add-port=9200/tcp
    elastic# firewall-cmd --add-port=5601/tcp
    elastic# firewall-cmd --runtime-to-permanent

To access Kibana visit http://elastic.example.com:5601

Navigate to Management > Advanced Settings

Locate `indexPattern:placeholder`, clear the default entry and set it to `project.foreman-logs*`

## Configuring ElasticSearch

Before proceeding with configuring Foreman or Satellite, make sure the server
can connect to remote ElasticSearch server:

    server# curl -XGET elastic.example.com:9200
    {
      "name" : "SL5yuZz",
      "cluster_name" : "elasticsearch",
      "cluster_uuid" : "UJy6MBGWQAKWcnv2VQL3YQ",
      "version" : {
            "number" : "5.6.10",
            "build_hash" : "b727a60",
            "build_date" : "2018-06-06T15:48:34.860Z",
            "build_snapshot" : false,
            "lucene_version" : "6.6.1"
      },
      "tagline" : "You Know, for Search"
    }

Log metadata follows Red Hat Common Data Model, which is used across various
Red Hat projects and products (e.g. OpenShift). Before any data is sent to
ElasticSearch, a JSON template must be downloaded and sent into ElasticSearch.

    server# yum -y install git rsyslog-elasticsearch rsyslog-mmnormalize rsyslog-mmjsonparse \
      foreman-journald foreman-proxy-journald
    server# git clone https://github.com/lzap/foreman-elasticsearch
    server# cd foreman-elasticsearch
    server# ls *.{json,conf} -1
    foreman.conf
    normalize_level.json
    org.foreman.viaq-cdm.2.4.4.index-pattern.json
    org.foreman.viaq-cdm.2.4.4.template.json
    org.foreman.viaq-cdm.5.5.2.index-pattern.json
    org.foreman.viaq-cdm.5.5.2.template.json
    prio_to_level.json

The git repository contains index templates which allow you to define templates
that will automatically be applied when new indices are created. The templates
include both settings and mappings and a simple pattern template that controls
whether the template should be applied to the new index. Mapping is the process
of defining how a document, and the fields it contains, are stored and indexed.
For instance, use mappings to define which string fields should be treated as
full text fields, which fields contain numbers, dates, or geolocations.

Upload template from the server to ElasticSearch server. Be sure to pick the
correct version as more than one JSON templates might have been generated. For
ElasticSearch 5.x choose JSON index template version 5.5.2.

    server# curl -X PUT "elastic.example.com:9200/_template/project.foreman" \
      -H 'Content-Type: application/json' -d@org.foreman.viaq-cdm.5.5.2.template.json

Rsyslog SELinux policy prevents from connecting to Elasticsearch port (9200),
the only option is to compile new policy or turn off SELinux for Rsyslog (a
bugzilla was filed for a boolean switch -
https://bugzilla.redhat.com/show_bug.cgi?id=1630874):

    server# semanage permissive -a syslogd_t

Configure syslog to normalize logs and pass them into ElasticSearch.
The following command copies foreman.conf and two more normalization JSON files
into /etc/rsyslog.d directory and configures ElasticSearch host and port.

    server# ./configure_rsyslog
    server# systemctl restart rsyslog
    server# systemctl status rsyslog

After restarting rsyslog check it's status and system journal. From now on, all
logs are forwarded to ElasticSearch instance. Errors in ES output module are
logged into /var/lib/rsyslog/es-errors.log file. Communication with
ElasticSearch can be secured via SSL/TLS. For more details, see:
https://www.rsyslog.com/doc/v8-stable/configuration/modules/omelasticsearch.html.

## Configuring Foreman and Smart Proxy

Configure the Foreman Rails application to send data into system journal rather
than /var/log/foreman/production.log directly. They will eventually appear in
the same file as copies to avoid confusion.

    server# foreman-installer --foreman-logging-level info --foreman-logging-type journald \
      --foreman-logging-layout pattern --foreman-proxy-log JOURNAL

Older versions of Foreman or Satellite might not have some installer options
available, in that case configure logging manually:

    server# cat /etc/foreman/settings.yaml | grep :logging: -A5
    :logging:
      :level: info
      :production:
            :type: journald
            :layout: pattern

    server# cat /etc/foreman-proxy/settings.yml | grep :log_file
    :log_file: JOURNAL

Foreman installer should have restarted both services, in case configuration
was modified manually restart the services:

    server# systemctl restart httpd foreman-proxy

Note when foreman-journald or foreman-proxy-journald packages were not
installed, there will be a warning in log and logging falls back to
stdout->journal logging which does not have any structured fields. Check for
warnings "Journald is not available on this platform. Falling back to STDOUT."

## Configuring Candlepin

The following section only applies when configuring Katello or Satellite. The
following configure script makes backup of
`/var/lib/tomcat/webapps/candlepin/WEB-INF/classes/logback.xml` file and then
overwrites it with [candlepin.xml](candlepin.xml) file which makes the
Candlepin application to send all logs into rsyslog via UDP/514 interface in
JSON.

    server# ./configure_candlepin
    server# systemctl restart tomcat

## Final notes

Use Elasticsearch API to query indices and records:

    server# curl -XGET elastic.example.com:9200/_cat/indices?v
    server# curl -XGET elastic.example.com:9200/project.foreman-logs.2019.05.22/events/_search

Data appears under project.foreman-logs.20YY.MM.DD indices where YMD represents
current year, month and day. All log entries are also copied into
`/var/log/messages` for easier local troubleshooting.

In addition to that, rsyslog is configured to make additional copies of
relevant records at the following well-known locations:

* /var/log/foreman/production.log
* /var/log/foreman-proxy/proxy.log
* /var/log/candlepin/candlepin.log

The following files are no longer in use:

* /var/log/candlepin/error.log (entries logged in candlepin.log)
* /var/log/candlepin/audit.log (entries logged in candlepin.log)

And the following files are new in this configuration:

* /var/log/foreman/audit.log (contains audit events from production.log)

### Log file rotation

Log entries which end up in system journal (when persistent mode is enabled) or
in `/var/log/messages` are automatically rotated, however for easy
troubleshooting logs are also copied to well-known locations. Log rotation
scripts which ship with Foreman, Katello or Satellite should be all replaced
with configuration which notifies rsyslog rather than daemons which send logs
to syslog. These are:

* `/etc/logrotate.d/foreman`
* `/etc/logrotate.d/foreman-proxy`
* `/etc/logrotate.d/candlepin`

Example configuration (taken from `/etc/logrotate.d/syslog`):

    /var/log/foreman/production.log
    /var/log/foreman-proxy/proxy.log
    /var/log/candlepin/candlepin.log
    {
        missingok
        sharedscripts
        postrotate
            /bin/kill -HUP `cat /var/run/syslogd.pid 2> /dev/null` 2> /dev/null || true
        endscript
    }

There is a script to do exactly that: it replaces the logrotate scripts above
with a dummy comment so RPM will treat them correctly and deploy a new config
`/etc/logrotate.d/foreman-elasticsearch` with standard rsyslog rotation
options.

    server# ./configure_logrotate

## Known issues

* Satellite 6.6 build is missing `foreman-proxy-journald` package, therefore when proxy is configured for system journal output, it prints a warning `Journald is not available on this platform. Falling back to STDOUT.` Logging from proxy works but it has no fields like `REQUEST` or `REMOTE_IP` available. Filed: https://bugzilla.redhat.com/show_bug.cgi?id=1713641

## TODO

* use $programname and if-else for better smart proxy processing
* add support for pulp and all its components as well
* configure tomcat with rsyslog as well (no correlation)
* review /var/log/messages


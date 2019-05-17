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

Install required components and checkout the configuration files:

    yum -y install git rsyslog-elasticsearch rsyslog-mmnormalize rsyslog-mmjsonparse foreman-journald
    git checkout https://github.com/lzap/foreman-elasticsearch
    cd foreman-elasticsearch

Copy few configuration files to `/etc/rsyslog.d`:

    cp ./foreman.conf ./*.json /etc/rsyslog.d/foreman.conf

Run the installer to change `/etc/foreman/settings.yaml` configuration for journald output. Older versions of Foreman installer might not understand some parameters, in that case edit the configuration file manually.

    foreman-installer --foreman-logging-level info --foreman-logging-type journald --foreman-logging-layout pattern

This is how the configuration should look like:

    cat /etc/foreman/settings.yaml | grep logging -A4
    :logging:
      :level: info
      :type: journald
      :layout: pattern
      :sys_pattern: "%m\n"

The installer should have restarted `httpd` service, if not do it manually now.

TODO:

/var/lib/tomcat/webapps/candlepin/WEB-INF/classes/logback.xml

https://bugzilla.redhat.com/show_bug.cgi?id=1669101

https://docs.google.com/document/d/1VYlH-VT4qMh9IbxoJSuxgciVxk5QgqL_mOXTWX9VxwA/edit#heading=h.yiwp6kb5amhe

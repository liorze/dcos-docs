---
post_title: Log Management in DC/OS with ELK
menu_order: 1
enterprise: 'yes'
---

You can pipe system and application logs from the nodes in a DC/OS cluster to your existing ElasticSearch, Logstash, and Kibana (ELK) server. This document describes how to store all unfiltered logs directly on ElasticSearch, and then perform filtering and specialized querying on ElasticSearch directly. The filebeat output from each node is sent directly to a centralized ElasticSearch instance, without using Logstash. If you're interested in using Logstash for log processing or parsing, consult the [Filebeat][2] and [Logstash][8] documentation.

**Important:** This document does not describe how to set up secure TLS communication between the Filebeat instances and ElasticSearch. For details on how to achieved this, please check the [Filebeat][2] and [ElasticSearch][5] documentation.

**Prerequisites**

These instructions are based on CentOS 7 and might differ substantially from other Linux distributions.

*   All DC/OS nodes must be able to connect to your ElasticSearch server on the port used for communication between ElasticSearch and Filebeat (9200 by default).

# <a name="all"></a>Step 1: All Nodes

For all nodes in your DC/OS cluster:

  1.  Install Elastic's [filebeat][2]. Installers are available for most major platforms.

  ```bash
  $ curl -L -O https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-5.0.0-x86_64.rpm
  $ sudo rpm -vi filebeat-5.0.0-x86_64.rpm
  ```

  2. Create a filebeat configuration file to monitor all Mesos log files. Also create the `/var/log/dcos` directory, and add another input entry for the file `/var/log/dcos/dcos.log` that we will funnel the DC/OS logs into in a later step. Please make sure to substitute the variables $ELK_HOSTNAME and $ELK_PORT below for the values of the host/port where your ElasticSearch is listening on.

  ```bash
  $ sudo mkdir -p /var/log/dcos
  $ sudo mv /etc/filebeat/filebeat.yml /etc/filebeat/filebeat.yml.BAK
  $ sudo tee /etc/filebeat/filebeat.yml <<-EOF 
  filebeat.prospectors:
  - input_type: log
    paths:
      - /var/lib/mesos/slave/slaves/*/frameworks/*/executors/*/runs/latest/stdout
      - /var/lib/mesos/slave/slaves/*/frameworks/*/executors/*/runs/latest/stderr
      - /var/log/mesos/*.log
      - /var/log/dcos/dcos.log
  tail_files: true
  output.elasticsearch:
    hosts: ["$ELK_HOSTNAME:$ELK_PORT"]
  EOF

  ```

# <a name="master"></a>Step 2: Master Nodes

For each Master node in your DC/OS cluster:

  1. Create a script that will parse the output of the DC/OS Master `journalctl` logs and will funnel them all to `/var/log/dcos/dcos/dcos.log`.

  ```bash

  $ sudo tee /etc/systemd/system/dcos-journalctl-filebeat.service<<-EOF 
  [Unit]
  Description=DCOS journalctl parser to filebeat
  Wants=filebeat.service
  After=filebeat.service

  [Service]
  Restart=always
  RestartSec=5
  ExecStart=/bin/sh -c '/usr/bin/journalctl --no-tail -f \
    -u dcos-3dt.service \
    -u dcos-3dt.socket \
    -u dcos-adminrouter-reload.service \
    -u dcos-adminrouter-reload.timer   \
    -u dcos-adminrouter.service        \
    -u dcos-bouncer.service            \
    -u dcos-ca.service                 \
    -u dcos-cfn-signal.service         \
    -u dcos-cosmos.service             \
    -u dcos-download.service           \
    -u dcos-epmd.service               \
    -u dcos-exhibitor.service          \
    -u dcos-gen-resolvconf.service     \
    -u dcos-gen-resolvconf.timer       \
    -u dcos-history.service            \
    -u dcos-link-env.service           \
    -u dcos-logrotate-master.timer     \
    -u dcos-marathon.service           \
    -u dcos-mesos-dns.service          \
    -u dcos-mesos-master.service       \
    -u dcos-metronome.service          \
    -u dcos-minuteman.service          \
    -u dcos-navstar.service            \
    -u dcos-networking_api.service     \
    -u dcos-secrets.service            \
    -u dcos-setup.service              \
    -u dcos-signal.service             \
    -u dcos-signal.timer               \
    -u dcos-spartan-watchdog.service   \
    -u dcos-spartan-watchdog.timer     \
    -u dcos-spartan.service            \
    -u dcos-vault.service              \
    -u dcos-logrotate-master.service  \
    > /var/log/dcos/dcos.log 2>&1'
  ExecStartPre=/usr/bin/journalctl --vacuum-size=10M

  [Install]
  WantedBy=multi-user.target
  EOF
  ```

# <a name="agent"></a>Step 2: Agent Nodes

For each Agent node in your DC/OS cluster:

  1. Create a script that will parse the output of the DC/OS Agent `journalctl` logs and will funnel them all to `/var/log/dcos/dcos/dcos.log`.

  ```bash
  $ sudo tee /etc/systemd/system/dcos-journalctl-filebeat.service<<-EOF 
  [Unit]
  Description=DCOS journalctl parser to filebeat
  Wants=filebeat.service
  After=filebeat.service

  [Service]
  Restart=always
  RestartSec=5
  ExecStart=/bin/sh -c '/usr/bin/journalctl --no-tail -f      \
    -u dcos-3dt.service                      \
    -u dcos-logrotate-agent.timer            \
    -u dcos-3dt.socket                       \
    -u dcos-mesos-slave.service              \
    -u dcos-adminrouter-agent.service        \
    -u dcos-minuteman.service                \
    -u dcos-adminrouter-reload.service       \
    -u dcos-navstar.service                  \
    -u dcos-adminrouter-reload.timer         \
    -u dcos-rexray.service                   \
    -u dcos-cfn-signal.service               \
    -u dcos-setup.service                    \
    -u dcos-download.service                 \
    -u dcos-signal.timer                     \
    -u dcos-epmd.service                     \
    -u dcos-spartan-watchdog.service         \
    -u dcos-gen-resolvconf.service           \
    -u dcos-spartan-watchdog.timer           \
    -u dcos-gen-resolvconf.timer             \
    -u dcos-spartan.service                  \
    -u dcos-link-env.service                 \
    -u dcos-vol-discovery-priv-agent.service \
    -u dcos-logrotate-agent.service          \
    > /var/log/dcos/dcos.log 2>&1'
  ExecStartPre=/usr/bin/journalctl --vacuum-size=10M

  [Install]
  WantedBy=multi-user.target
  EOF
  ```

# <a name="all-3"></a>Step 3: All Nodes

  1. In all nodes, start and enable the filebeat log parsing services created above:

  ```bash
  $ sudo chmod 0755 /etc/systemd/system/dcos-journalctl-filebeat.service
  $ sudo systemctl daemon-reload
  $ sudo systemctl start dcos-journalctl-filebeat.service
  $ sudo chkconfig dcos-journalctl-filebeat.service on
  $ sudo systemctl start filebeat
  $ sudo chkconfig filebeat on
  ```

# <a name="all"></a>Step 3: ELK Node Notes

The ELK stack will receive, store, search and display information about the logs parsed by the Filebeat instances configured above for all nodes in the cluster. In this document, we are streaming directly from Filebeat into ElasticSearch, so Logstash is not used in this architecture. If you're interested in filtering, parsing and grok'ing the logs with an intermediate Logstash stage, please check the Logstash [documentation][8].

  1. Important parameters to configure in the ElasticSearch node.

  Some parameters will need to be changed from their default values for ElasticSearch to be ready to receive information. Edit the ElasticSearch configuration file (typically `/etc/elasticsearch/elasticsearch.yml`) and edit:


  ```
  network.host = [IP address from the interface in your ElasticSearch node connecting to the Filebeat instances]
  ```

  Other parameters in the file are beyond the scope of this document. For details, please check the ElasticSearch [documentation][5].


### Known Issue

*   The agent node Filebeat configuration expects tasks to write logs to `stdout` and `stderr`. Some DC/OS services, including Cassandra and Kafka, do not write logs to `stdout` and `stderr`. If you want to log these services, you must customize your agent node Filebeat configuration.

# What's Next

For details on how to filter your logs with ELK, see [Filtering DC/OS logs with ELK][3].

 [2]: https://www.elastic.co/guide/en/beats/filebeat/current/filebeat-getting-started.html
 [3]: ../filter-elk/
 [4]: https://www.elastic.co/guide/en/elastic-stack/current/index.html
 [5]: https://www.elastic.co/guide/en/elasticsearch/reference/5.0/index.html
 [6]: https://www.elastic.co/guide/en/kibana/current/install.html
 [7]: https://www.elastic.co/guide/en/logstash/current/installing-logstash.html
 [8]: https://www.elastic.co/guide/en/logstash/current/index.html

# Ansible installation

## Requirements
- ansible

## Installation

To install the stack, run: ``ansible-playbook site.yml -i hosts``

The *site.yml* contains the main configuration regarding the roles that are to be installed to the specified hosts.

The hosts are configured in the *hosts* file.

The following roles can be found in the *roles* folder:
- elasticsearch
- logstash
- kibana
- kafka

Each of the roles consists of their own variables (found in the defaults folder), handlers, templates and most importantly their tasks.
Variables that can be used by more than one of the roles are defined in the *group_vars* folder.

Note: It is desirable that the *elasticsearch* role be installed first, since it adds the necessary repositories for the whole ELK stack.

## Defaults

### Elasticsearch
```
elastic_key_url: "https://artifacts.elastic.co/GPG-KEY-elasticsearch"
elastic_repo: "deb https://artifacts.elastic.co/packages/7.x/apt stable main"
```

### Kibana
```
kibana_version: 7.10.2
kibana_server_ip: "localhost"
kibana_server_port: 5601
kibana_user: "kibana"
kibana_password: "kibana"
dashboard_name: aminer.ndjson
dashboard_path: ../dashboards/{{ dashboard_name }}
dashboard_file: "{{ lookup('file', dashboard_path) }}"
```

### Logstash
```
kafka_server_ip: "localhost"
kafka_server_port: 9092
kafka_topics: "aminer" 
input_beats_port: 5044
elasticsearch_host: "http://localhost:9200"
```

### Kafka
```
kafka_base_url: http://www-eu.apache.org/dist/kafka
kafka_version: 2.5.0
kafka_scala_version: 2.12
kafka_root_dir: /etc
kafka_dir: "{{ kafka_root_dir }}/kafka"
kafka_listener_protocol: PLAINTEXT
kafka_listener_hostname: "localhost"
kafka_listener_port: 9092
systemd_path: /etc/systemd/system
```

## Examples

### Variables

Kibana version, server IP, and port are configured in *kibana/defaults/main.yml* as follows:

```
kibana_version: 7.10.2
kibana_server_ip: "localhost"
kibana_server_port: 5601
```

Note that for AMiner CTI dashboard plugin to work properly, only kibana version 7.10.x are supported for now.

In the main file of kibana tasks, these variables are used to modify the certain lines in the already existing kibana configuration file that is generated by default on kibana installation.

```
- name: Update kibana server ip
  lineinfile:
    destfile: /etc/kibana/kibana.yml
    regexp: 'server.host:'
    line: 'server.host: {{ kibana_server_ip }}'
```

### Handlers

Handlers are shortcut functions that are defined in the same way as the variables, and are called within tasks when a certain action needs to be performed.

A good example are the systemd services, which are usually enabled and started after the desired package is installed and configured as a service.

In our case, the services of _elasticsearch_, _kibana_ and _logstash_ are automatically enabled and started, but kafka and zookeper are not.

To create the *kafka.service* file (same applies to *zookeeper*) we use templates. There we define the necessary configuration and then push the templates to the _systemd_ path. After that, using the following handlers, we _daemon-reload_ and (re)start the service.

```
- name: restart kafka systemd
  systemd:
    name: kafka
    state: restarted
    daemon_reload: yes
```


## Notes
 
> The _elastic_key_url_ and _elastic_repo_ can be found in the defaults folder of _elasticsearch_. In case you do not include the elasticsearch role in the *site.yml*, i.e. if you only want to install the other roles, make sure the _elastic key_ and _repo_ are already present in your machine. In this case, they are added via the elasticsearch role, namely in the first four tasks.

> For now, importing the kibana dashboard (ndjson file) is done using the **curl** comand. The recommended ansible way of accessing a REST API using the **uri** module is not working, since it accepts as *Content-Type* only JSON, form-urlencoded or RAW data, whereas for importing the *ndjson* file that contains the AMiner dashboard **multipart-formdata** is needed.


# AMiner in Kibana

## Creating indices

To access AMiner anomalies saved in *elasticsearch* indices must be created. There are two types of AMiner messages: anomalies and info statuses. Their index configuration can be seen in the table below.

| Index  | Time Filter |
| ------------- | ------------- |
| aminer-anomaly*  | @logtimestamp  |
| aminer-statusinfo*  | @fromtimestamp  |

Index configuration can be accessed in **Management > Index Patterns**.

## Adding new fields to StatusInfo chart

Currently, the status info chart shows values from the following fields: model_type, status_code, classification, event_type_str, host, php, and sp. To add new ones, go to the *Visualize* page and click on **[AMiner] - StatusInfo**. This visualization is of type TSVB, and it includes different visualization options. In the *Timeseries* tab, inside the *data* panel, click on the plus sign next to the trash sign to add a new field. To be shown correctly together with the already defined fields, configure its metric as follows:
Aggregation=Sum, Field={desired_field}, GroupBy=Everything.

Note that for values under 1 the log scaled StatusInfo chart does not show anything, since the log of 0 is undefined. In this case, assume the value is 0.


## AMiner dashboard

The AMiner dashboard and its consisting visualizations and indices can be imported via ansible or by importing the **.ndjson** file found in **roles/kibana/dashboards/** directly to Kibana. The import functionality is located at **Management > Saved Objects**.

The AMiner dashboard can then be accessed in the *Dashboard* section of Kibana. Its visualizations can be edited separately in the *Visualize* section.

Within a dashboard, it is possible to apply different time ranges to different visualizations by clicking the so-called meatballs menu and selecting **Customize time range**.


# Configuration of Kafka and ELK stack

## Kafka

To setup the server IP of the kafka server, go to: 
> roles > kafka > defaults > main.yml

Among others, there you can specify the protocol, hostname, and the port of the server. By default, the value of the hostname (IP) is "localhost".

If manual configuration is desired, go to:
> {{ kafka_dir }}/config/server.properties

Variable {{ kafka_dir }} is configured too in the ansible kafka role (default=/etc/kafka).

## Logstash

Just like with kafka, the main configuration variables can be found under the *logstash role*. There you can specify the following:

- kafka_server_ip: "localhost"
- kafka_server_port: 9092
- kafka_topics: "aminer" 
- input_beats_port: 5044
- elasticsearch_host: "http://localhost:9200"

The values provided here are defaults. Change them according to your needs. The logstash configuration file in the host can be found under:
> /etc/logstash/conf.d/logstash.conf

## Elasticsearch

No configuration is necessary, provided the kafka and kibana instances are installed in the same host as the elasticsearch instance. Otherwise, configure the elastic search network IP(s) under:
> /etc/elasticsearch/elasticsearch.yml

Note that you need root permissions to modify the file.

## Kibana

Under the ansible role *kibana*, the most important configuration variables are:

- Kibana version
- Kibana server IP
- Kibana server port
- AMiner dashboard name (saved in folder *Dashboards*)


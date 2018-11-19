# Sending OP5 performance data to Elasticsearch node #

Below instruction requires that between OP5 node and Elasticsearch node is working Logstash instance.

## Elasticsearch ##
1.	First, settings section in *op5template.sh* should be adjusted, either:
	- there is a default template present on Elasticsearch that already covers shards and replicas then settings sections should be removed from the *op5template.sh* before executing
	- there is no default template - shards and replicas should be adjusted for you environment (keep in mind replicas can be added later, while changing shards count on existing index requires 
	reindexing it)

			"settings": {
			  "number_of_shards": 5,
			  "number_of_replicas": 0
			}

1. In URL *op5perfdata* is a name for the template - later it can be search for or modify with it.
1. The "*template*" is an index pattern. New indices matching it will have the settings and mapping applied automatically (change it if you index name for *op5 perfdata* is different).
1. Mapping name should match documents type:

		"mappings": {
		  "op5perflogs"

1. Running op5template.sh will create a template (not index) for OP5 perf data documents.

## Logstash ##

1.	The *op5perflogs.conf* contains example of *input/filter/output* configuration. It has to be copied to */etc/logstash/conf.d/*. Make sure that the *logstash* has permissions to read the configuration files:
	
		chmod 664 /etc/logstash/conf.d/op5perflogs.conf

1. In the input section comment/uncomment *“beats”* or *“tcp”* depending on preference (beats if *Filebeat* will be used and tcp if *NetCat*). The port and the type has to be adjusted as well:

		port => PORT_NUMBER
		type => "op5perflogs"

1. In a filter section type has to be changed if needed to match the input section and Elasticsearch mapping.
1. In an output section type should match with the rest of a *config*. host should point to your elasticsearch node. index name should correspond with what has been set in elasticsearch template to allow mapping application. The date for index rotation in its name is recommended and depending on the amount of data expecting to be transferred should be set to daily (+YYYY.MM.dd) or monthly (+YYYY.MM) rotation:

		hosts => ["127.0.0.1:9200"]
		index => "op5-perflogs-%{+YYYY.MM.dd}"

1. Port has to be opened on a firewall:

		sudo firewall-cmd --zone=public --permanent --add-port=PORT_NUMBER/tcp
		sudo firewall-cmd --reload

1. Logstash has to be reloaded:	

		sudo systemctl restart logstash

	or

		sudo kill -1 LOGSTASH_PID

## OP5 Monitor ##

1.	You have to decide wether FileBeat or NetCat will be used. In case of Filebeat - skip to the second step. Otherwise:
	- Comment line:

			54    open(my $logFileHandler, '>>', $hostPerfLogs) or die "Could not open $hostPerfLogs"; #FileBeat
			•	Uncomment lines:
			55 #    open(my $logFileHandler, '>', $hostPerfLogs) or die "Could not open $hostPerfLogs"; #NetCat
			...
			88 #    my $logstashIP = "LOGSTASH_IP";
			89 #    my $logstashPORT = "LOGSTASH_PORT";
			90 #    if (-e $hostPerfLogs) {
			91 #        my $pid1 = fork();
			92 #        if ($pid1 == 0) {
			93 #            exec("/bin/cat $hostPerfLogs | /usr/bin/nc -w 30 $logstashIP $logstashPORT");
			94 #        }
			95 #    }
	- In process-service-perfdata-log.pl and process-host-perfdata-log.pl: change logstash IP and port:

			92 my $logstashIP = "LOGSTASH_IP";
			93 my $logstashPORT = "LOGSTASH_PORT";

1. In case of running single op5 node, there is no problem with the setup. In case of a peered environment *$do_on_host* variable has to be set up and the script *process-service-perfdata-log.pl/process-host-perfdata-log.pl* has to be propagated on all of OP5 nodes:

		16 $do_on_host = "EXAMPLE_HOSTNAME"; # op5 node name to run the script on
		17 $hostName = hostname; # will read hostname of a node running the script

1. Example of command definition (*/opt/monitor/etc/checkcommands.cfg*) if scripts have been copied to */opt/plugins/custom/*:

		# command 'process-service-perfdata-log'
		define command{
		    command_name                   process-service-perfdata-log
		    command_line                   /opt/plugins/custom/process-service-perfdata-log.pl $TIMET$
		    }
		# command 'process-host-perfdata-log'
		define command{
		    command_name                   process-host-perfdata-log
		    command_line                   /opt/plugins/custom/process-host-perfdata-log.pl $TIMET$
		    }

1. In */opt/monitor/etc/naemon.cfg service_perfdata_file_processing_command* and *host_perfdata_file_processing_command* has to be changed to run those custom scripts:
 
		service_perfdata_file_processing_command=process-service-perfdata-log
		host_perfdata_file_processing_command=process-host-perfdata-log

1. In addition *service_perfdata_file_template* and *host_perfdata_file_template* can be changed to support sending more data to Elasticsearch. For instance, by adding *$HOSTGROUPNAMES$* and *$SERVICEGROUPNAMES$* macros logs can be separated better (it requires changes to Logstash filter config as well)
1. Restart naemon service:

		sudo systemctl restart naemon # CentOS/RHEL 7.x
		sudo service naemon restart # CentOS/RHEL 6.x

1. If *FileBeat* has been chosen, append below to *filebeat.conf* (adjust IP and PORT):

		filebeat.inputs:
		- type: log
		  enabled: true
		  paths:
		    - /opt/monitor/var/service_performance.log
		    - /opt/monitor/var/host_performance.log

			tags: ["op5perflogs"]


		output.logstash:
		  # The Logstash hosts
		  hosts: ["LOGSTASH_IP:LOGSTASH_PORT"]



	- Restart FileBeat service:

			sudo systemctl restart filebeat # CentOS/RHEL 7.x
			sudo service filebeat restart # CentOS/RHEL 6.x


## Kibana ##

At this moment there should be new index on the Elasticsearch node with performance data documents from OP5 Monitor. 
Login to an Elasticsearch node and run: `curl -XGET '127.0.0.1:9200/_cat/indices?v'` Example output:

	health status index                      pri rep docs.count docs.deleted store.size pri.store.size
	green  open   auth                       5   0          7         6230      1.8mb          1.8mb
	green  open   op5-perflogs-2018.09.14    5   0      72109            0     24.7mb         24.7mb

After a while, if there is no new index make sure that: 

- Naemon is runnig on OP5 node
- Logstash service is running and there are no errors in: */var/log/logstash/logstash-plain.log* 
- Elasticsearch service is running an there are no errors in: */var/log/elasticsearch/elasticsearch.log*

If the index has been created, in order to browse and visualize the data “*index pattern*” needs to be added to Kibana. 

1. After logging in to Kibana GUI go to *Settings* tab and add *op5-perflogs-** pattern. Chose *@timestamp* time field and click *Create*. 
1. Performance data logs should be now accessible from Kibana GUI Discovery tab ready to be visualize .

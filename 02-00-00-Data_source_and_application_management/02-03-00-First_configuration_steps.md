# First configuration steps #

## Run the instalation ##
To install and configure OP5 Log Analytics on the CentOS Linux system you should:

- copy archive OP5 Log Analytics tar.bz2 to the hosted server;
- extract archive OP5 Log Analytics tar.bz2 contain application:

	cd /root/
	tar xvfj archive.tar.bz2

- go to the application directory and run installation script as a root user:

		cd /roo/insatll
		./install.sh

## Installation steps ##

During instalation you will be ask about following tasks:

- add firewall exeption on ports 22(ssh), 5044, 5514 (Logstash), 5601 (Kibana), 9200 (Elastisearch), 9300 (ES cross-JVM);
- installation of Java environment (Open-JDK), if you use your own Java environment - answer "N";
- installation of Logstash application;
- configuration of Logstash with custom OP5 Log Analytics configuration;
- connect to the OP5 CentOS repository, which provides Python libraries, and some fonts;
- installation of Kibana, the OP5 Log Analytics GUI;
- installation of Python dependencies;
- installation of mail components for OP5 Log Analytics notification;
- installation of data-node of Elasticsearch;
- configuration of Elasticsearch as Data Node;
- configuration of Elasticsearch as Master Node.

## Optional installation steps: ##
Optionally you can:

- install and configure the filebeat agent;
- install and configure the winlogbeat agent;
- configure op5 perf_data to integrated with the OP5 Monitor;
- configure naemonLogs to integrated with the Naemon;
- configure integration with Active Directory and SSO servers. You can find necessary information in [12-00-00-Integration_with_AD](/12-00-00-Integration_with_AD/12-00-00-Integration_with_AD.md) and [13-00-00-Windows-SSO](/13-00-00-Windows-SSO/13-00-00-Windows-SSO.md);
- install and conigure monitoring with Marver:

	cd /usr/share/elasticsearch
	sudo bin/plugin install license
	sudo bin/plugin install marvel-agent
	systemctl restart elasticsearch

- enable predictive functionality in Intelligence module:

		curl -XPOST 'http://localhost:9200/_aliases' -d '{
		 		"actions" : [
		     	{ "add" : { "index" : "intelligence", "alias" : "predictive" } },
		    	 { "add" : { "index" : "perfdata-linux", "alias" : "predictive" } }
		 		]}'

- generate writeback index for Alert service:
	
		*/opt/alert/bin/elastalert-create-index --config /opt/alert/config.yaml*

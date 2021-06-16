# Configure logging drivers

## Official Docker Documentation
[Configure logging drivers](https://docs.docker.com/engine/admin/logging/overview/)
or   
[If above link is not working](https://docs.docker.com/config/containers/logging/configure/)


#Docker Logging mechanisms
* Docker supports multiple logging mechanisms to [View logs for a container or service](https://docs.docker.com/config/containers/logging/)
	* Helps get information from running containers and services. 
	* These mechanisms are called logging drivers. 
	* Each Docker daemon has a default logging driver
	* Containers uses it unless we configure it to use a different logging driver
	
	
* (By default) 
	* Docker uses the [json-file logging driver](https://docs.docker.com/config/containers/logging/json-file/)
	* Caches container logs as JSON internally. 
	* Additionally we can also implement and use logging driver plugins.
	* No log-rotation is performed. 
	* So log-files stored by the default can utilize significant amount of disk space.

* Use the Non-default “local” logging driver 
	* Recommended as it performs log-rotation by default
	* prevent disk-exhaustion
	* uses a more efficient file format. 
	

#Configure the default logging driver
* To configure the Docker daemon to default to a specific logging driver
	* Set the value of log-driver to the name of the logging driver in the daemon.json. Refer to the “daemon configuration file” section in the dockerd reference manual for details.

* The default configuration file for Docker is 
	* In linux (/etc/docker/docker.
The default logging driver is json-file. 


The following example sets the default logging driver to the local log driver:

{
  "log-driver": "local"
}
If the logging driver has configurable options, you can set them in the daemon.json file as a JSON object with the key log-opts. The following example sets two configurable options on the json-file logging driver:

{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3",
    "labels": "production_status",
    "env": "os,customer"
  }
}
Restart Docker for the changes to take effect for newly created containers. 
Existing containers do not use the new logging configuration.

log-opts (Refer above)
	configuration options in the daemon.json configuration file 
	must be provided as strings. 
	Boolean and numeric values 
		(e.g. max-file in the example above) 
		must therefore be enclosed in quotes (").

default logging driver is json-file. 
To find the current default logging driver for the Docker daemon
	docker info --format '{{.LoggingDriver}}'

Note

Changing the default logging driver or logging driver options in the daemon configuration 
	only affects containers that are created after the configuration change. 
	Existing containers retain the old logging driver options 
	So if required re-create the container 
	
#Configure the logging driver for a container
While starting containers, 
	we can configure to use a different logging driver than the Docker daemon’s default
		using the --log-driver flag. 
		
	docker run -it --log-driver none alpine ash
	
	N.B: 
		Not all logging driver supports configurable option.
		If the logging driver has configurable options, 
		you can set them using as
		--log-opt <NAME>=<VALUE> flag. Even if the container uses the default logging driver, it can use different configurable options.

	
To find the current logging driver for a running container, 
	docker inspect -f '{{.HostConfig.LogConfig.Type}}' <CONTAINER>

#Configure the delivery mode of log messages from container to log driver
Docker provides two modes for delivering messages from the container to the log driver:
	(default) direct, 
		blocking delivery from container to driver
	non-blocking delivery 
		stores log messages in an intermediate per-container ring buffer for consumption by driver
		Will not block the application during logging back pressure. 
		Applications are likely to fail in unexpected ways when STDERR or STDOUT streams block.
N.B: 
	When the buffer is full and a new message is enqueued
		the oldest message in memory is dropped. 
	This is preferred to blocking the log-writing process of an application.

The mode log option controls whether to use the blocking (default) or non-blocking message delivery.

max-buffer-size log option 
	controls the size of the ring buffer used for intermediate message storage when mode is set to non-blocking. 
	max-buffer-size defaults to 1 MB.

	docker run -it --log-opt mode=non-blocking --log-opt max-buffer-size=4m alpine ping 127.0.0.1

#Use environment variables or labels with logging drivers
Some logging drivers 
	add the value of a container’s --env|-e or --label flags to the container’s logs. 

This example starts a container using the Docker daemon’s default logging driver (let’s assume json-file) but sets the environment variable os=ubuntu.
	docker run -dit --label production_status=testing -e os=ubuntu alpine sh

If the logging driver supports it, this adds additional fields to the logging output. The following output is generated by the json-file logging driver:

"attrs":{"production_status":"testing","os":"ubuntu"}
Supported logging drivers
The following logging drivers are supported. See the link to each driver’s documentation for its configurable options, if applicable. If you are using logging driver plugins, you may see more options.

Driver	Description
none	
	No logs are available for the container and docker logs does not return any output.
local	
	Logs are stored in a custom format designed for minimal overhead.
json-file	
	The logs are formatted as JSON. The default logging driver for Docker.
syslog	
	Writes logging messages to the syslog facility. The syslog daemon must be running on the host machine.
journald	
	Writes log messages to journald. The journald daemon must be running on the host machine.
gelf	
	Writes log messages to a Graylog Extended Log Format (GELF) endpoint such as Graylog or Logstash.
fluentd	
	Writes log messages to fluentd (forward input). The fluentd daemon must be running on the host machine.
awslogs	
	Writes log messages to Amazon CloudWatch Logs.
splunk	
	Writes log messages to splunk using the HTTP Event Collector.
etwlogs	
	Writes log messages as Event Tracing for Windows (ETW) events. Only available on Windows platforms.
gcplogs	
	Writes log messages to Google Cloud Platform (GCP) Logging.
logentries	
	Writes log messages to Rapid7 Logentries.

Note
When using Docker Engine 19.03 or older, the docker logs command is only functional for the local, json-file and journald logging drivers. Docker 20.10 and up introduces “dual logging”, which uses a local buffer that allows you to use the docker logs command for any logging driver. Refer to reading logs when using remote logging drivers for details.

#Limitations of logging drivers
Reading log information requires decompressing rotated log files, which causes a temporary increase in disk usage (until the log entries from the rotated files are read) and an increased CPU usage while decompressing.
The capacity of the host storage where the Docker data directory resides determines the maximum size of the log file information.


Steps to install fluentd on centos7
	https://manastri.blogspot.com/2020/09/how-to-install-td-agent-on-centosrhel-7.html
	curl -L https://toolbelt.treasuredata.com/sh/install-redhat-td-agent4.sh | sh	

Steps to install fluentd on amazon ami/RHEL
	https://docs.fluentd.org/installation/install-by-rpm

sudo su
vi /etc/td-agent/td-agent.conf

-------------------
<source>
  @type forward
  port  24224
</source>
-------------------
This defines the source as forward, which is the Fluentd protocol that runs on top of TCP and will be used by Docker when sending the logs to Fluentd.

When the log records come in,, they will have some extra associated fields, including time, tag, message, container_id, and a few others. You use the information in the _tag_ field to decide where Fluentd should send that data. This is called data routing.

To configure this, define a match section that matches the contents of the tag field and route it appropriately. Add this configuration to the file:

---------------
<match docker.**>
  @type elasticsearch
  logstash_format true
  host 127.0.0.1
  port 9200
  flush_interval 5s
</match>
---------------

This rule says that every record with a tag prefixed with docker. will be send to Elasticsearch, which is running on 127.0.0.1 on port 9200. The flush_interval tells Fluentd how often it should records to Elasticsearch.

	$ sudo systemctl start td-agent.service
	$ sudo systemctl status td-agent.service
If you want to stop 	
	$ sudo systemctl stop td-agent.service

sudo td-agent-gem install fluent-plugin-elasticsearch
yum install gem
gem install fluentd-plugin-elasticsearch --no-rdoc --no-ri

sudo sysctl -w vm.max_map_count=262144
docker run -d -p 9200:9200 -p 9300:9300 elasticsearch:6.5.0










-----------------------------------------------

eyJraWQiOiJzcGx1bmsuc2VjcmV0IiwiYWxnIjoiSFM1MTIiLCJ2ZXIiOiJ2MiIsInR0eXAiOiJzdGF0aWMifQ.eyJpc3MiOiJhZG1pbiBmcm9tIGQ2M2M5YjdjY2VlOCIsInN1YiI6ImFkbWluIiwiYXVkIjoiYWNjZXNzIGZyb20gZG9ja2VyIiwiaWRwIjoiU3BsdW5rIiwianRpIjoiZTQ3MGRmZTEyZDFhZTNiMmMyOTRmMjYyMTcyYWFkN2RlMDVlNTIyMjA3ZDhiZGMzNzU3YzE3NjI1ZDBjNjg1YSIsImlhdCI6MTYyMDcwMzA1MSwiZXhwIjoxNjIyMjI2NTQwLCJuYnIiOjE2MjIyMjY1NDB9.fZTyT3zz5JqpTfFTxcJLYsCFTwjV81X5nTIuvrLwi9FG167dOhB1d-RWll38Rtyq5KgfNb20NYVwyfD-uSQ-pw

docker run --publish 80:80 --log-driver=splunk --log-opt splunk-token=eyJraWQiOiJzcGx1bmsuc2VjcmV0IiwiYWxnIjoiSFM1MTIiLCJ2ZXIiOiJ2MiIsInR0eXAiOiJzdGF0aWMifQ.eyJpc3MiOiJhZG1pbiBmcm9tIGQ2M2M5YjdjY2VlOCIsInN1YiI6ImFkbWluIiwiYXVkIjoiYWNjZXNzIGZyb20gZG9ja2VyIiwiaWRwIjoiU3BsdW5rIiwianRpIjoiZTQ3MGRmZTEyZDFhZTNiMmMyOTRmMjYyMTcyYWFkN2RlMDVlNTIyMjA3ZDhiZGMzNzU3YzE3NjI1ZDBjNjg1YSIsImlhdCI6MTYyMDcwMzA1MSwiZXhwIjoxNjIyMjI2NTQwLCJuYnIiOjE2MjIyMjY1NDB9.fZTyT3zz5JqpTfFTxcJLYsCFTwjV81X5nTIuvrLwi9FG167dOhB1d-RWll38Rtyq5KgfNb20NYVwyfD-uSQ-pw --log-opt splunk-url=https://172.31.77.41:8088 --log-opt splunk-insecureskipverify=true nginx

docker run -d -p 8000:8000 -e SPLUNK_START_ARGS='--accept-license' -e SPLUNK_PASSWORD='password' splunk/splunk:latest
	- user should be admin/password i guesss
	
	<ip>/8000
	
	Search for index="_internal"
	
docker run --p 80:80 --log-driver=splunk --log-opt splunk-token=99E16DCD-E064-4D74-BBDA-E88CE902F600 --log-opt splunk-url=https://192.168.1.123:8088 --log-opt splunk-insecureskipverify=true nginx	
 
 docker run -d --name fwder -e SPLUNK_START_ARGS=--accept-license -e SPLUNK_FORWARD_SERVER=localhost:8000 -e SPLUNK_USER=root splunk/universalforwarder

docker run -d -p 9997:9997 -e SPLUNK_START_ARGS='--accept-license' -e SPLUNK_PASSWORD='password' --name uf splunk/universalforwarder:latest

 docker run -d --name fwder -e SPLUNK_START_ARGS=--accept-license -e "SPLUNK_STANDALONE_URL=<name of splunk> splunk/universalforwarder
 
 docker run -d --name fwder -e SPLUNK_START_ARGS=--accept-license -e "SPLUNK_STANDALONE_URL=tender_tereshkova" splunk/universalforwarder
 
 
  docker run -d -p 9997:9997 -e "SPLUNK_START_ARGS=--accept-license" -e "SPLUNK_PASSWORD=password" --name uf splunk/universalforwarder:latest
 
 docker run --log-driver syslog --log-opt syslog-address=udp://<ip>:port alping echo hello world
 
  "storage-driver": "zfs",
 
{
  "log-driver": "splunk",
  "log-opts": {
    "splunk-token": "",
    "splunk-url": "https://172.31.73.125:8000",
    "splunk-insecureskipverify": "true",
	"tag":"{{.ImageName}}/{{.Name}}/{{.ID}}"
  }
}
 --label type=test --label location=home 
 --log-driver=splunk --log-opt splunk-token=99E16DCD-E064-4D74-BBDA-E88CE902F600 --log-opt  --log-opt splunk-insecureskipverify=true --log-opt tag="{{.ImageName}}/{{.Name}}/{{.ID}}" --log-opt labels=type,location 
 
 
 Web UI, go to the Settings → Data Inputs. Choose HTTP Event Collector. Enable it with Global Settings and add one New Token. After the token is created, you will find the Token Value which is a guid. Write it down, as you will need it later for configuring the Splunk Logging Driver.
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
Restart Docker for the changes to take effect for newly created containers. Existing containers do not use the new logging configuration.


Reference: 
https://docs.docker.com/engine/security/

##Docker security introduction
------------------------------

Four major areas to consider when reviewing Docker security:
	- intrinsic security of the kernel and its support for namespaces and cgroups;
	- attack surface of the Docker daemon itself;
	- loopholes in the container configuration profile, 
		either by default, 
		or when customized by users.
	- “hardening” security features of the kernel and how they interact with containers.


#Kernel namespaces
------------------
Docker containers 
	very similar to LXC containers
	have similar security features. 
	
	Start a container with docker run, 
		behind the scenes 
			Docker creates a set of 
				namespaces and 
				control groups 
					for the container.

Namespaces provide the first and most straightforward form of isolation: 
	
Each container also gets its own 
	network stack
	i.e. container doesn’t get privileged access to the sockets or interfaces of another container. 
	When you specify 
		public ports for your containers or 
		use links 
	then IP traffic is allowed between containers. 
	
	They can 
		ping each other, 
		send/receive UDP packets
		establish TCP connections, 
	This can be restricted if required. 
	
	From a network architecture point of view, 
		all containers on a given Docker host are sitting on bridge interfaces (default). 
	ie.. they are just like physical machines connected through a common Ethernet switch; 
		no more, 
		no less.

How mature is the code providing kernel namespaces and private networking? 
Kernel namespaces 
	introduced between kernel version 2.6.15 and 2.6.26. 
		in July 2008 (date of the 2.6.26 release ), 
	code has been exercised and scrutinized on a large number of production systems. 
	The design and inspiration for the namespaces code are even older. 
		features of OpenVZ 
			initially released in 2005
			both the design and the implementation are pretty mature.

#Control groups
---------------
Control Groups 
	another key component of Linux Containers. 
	Implement resource accounting and limiting. 
	Provide 
		many useful metrics, 
		ensure that each container gets its fair share of 
			memory, 
			CPU, 
			disk I/O; 
			ect.
		Ensure a single container cannot bring the system down 
			by exhausting one of those resources.
			So reduced risk of impact on the host system kernel.
			This helps to reduce denial-of-service attacks probablity. 
			Particularly important on multi-tenant platforms like aws 
				guarantee a consistent 
					uptime (and performance) 
						even when some applications start to misbehave.

Control Groups have been around for a while as well: 
	the code was started in 2006
	initially merged in kernel 2.6.24.

#Docker daemon attack surface
-----------------------------
Running containers (and applications) with Docker implies running the Docker daemon. 
This daemon requires root privileges 
	[unless you opt-in to Rootless mode], 
This exposes some risk. So.
	Only trusted users should be allowed to control your Docker daemon. 
	This is a direct consequence of some powerful Docker features. 
	Docker allows you to 
		share a directory between the Docker host and a guest container
		you can do that without limiting the access rights of the container. 
	So you can start a container where the critical directories are shared.
	Solution: Use volumes as much as possible.
	Security risk can be even higher if your container is exposed through API
	So always 
		expose REST Api over https and secure it using certificates.
		ensure that it is reachable only from a trusted network or VPN.
		
	The REST API server endpoint 
		(used by the Docker CLI to communicate with the Docker daemon) changed in Docker 0.5.2, 
		now uses a UNIX socket instead of a TCP socket bound on 127.0.0.1 
		(TCP socket is prone to cross-site request forgery attacks if you happen to run Docker directly on your local machine, outside of a VM). 
	Use traditional UNIX permission checks to limit access to the control socket.

	If you prefer SSH over TLS
		You can also use DOCKER_HOST=ssh://USER@HOST or ssh -L /path/to/docker.sock:/var/run/docker.sock.

The daemon is also potentially vulnerable to other inputs
	e.g. image loading from either disk with docker load
or 
	docker pull. 
	
As of Docker 1.3.2, 
	images are now extracted in a chrooted subprocess on Linux/Unix platforms, 
		first-step in a wider effort toward privilege separation. 
As of Docker 1.10.0
	all images are stored and accessed by the cryptographic checksums of their contents, 
	limiting the possibility of an attacker causing a collision with an existing image.
Recommendation	
	Run Docker exclusively on a server with bear min. other software

#Linux kernel capabilities
By default, 
	Docker starts containers with a restricted set of capabilities. 

	Capabilities turn the binary “root/non-root” dichotomy into a fine-grained access control system. 
	Processes (like web servers) that just need to bind on a port below 1024 
		do not need to run as root: 
		they can just be granted the net_bind_service capability instead. 
		And there are many other capabilities, for almost all the specific areas where root privileges are usually needed.

This means a lot for container security; let’s see why!

Typical servers run several processes as root
	including the SSH daemon, 
	cron daemon, 
	logging daemons, 
	kernel modules, 
	network configuration tools, 
	ect. 
In containers most of those tasks are handled by the infrastructure around the container:

	SSH 
		managed by a single server running on the Docker host
	cron, 
		can run as a user process
	log management 
		can be handed to Docker, or 
		third-party services like Loggly or Splunk;
	hardware management 
		irrelevant, 
	network management 
		happens outside of the containers
		what is happening within containers can be well managed.
	So most containers do not need “real” root privileges at all. 
		deny all “mount” operations;
		deny access to raw sockets (to prevent packet spoofing);
		deny access to some filesystem operations, 
			like creating new device nodes, 
			changing the owner of files
			altering attributes etc;
		deny module loading;
		etc.
So if an intruder manages to escalate to root within a container
	it is much harder to do serious damage
	or to escalate to the host.

This doesn’t affect regular web apps
but reduces the vectors of attack by malicious users considerably. 

One primary risk with running Docker containers with
	default set of capabilities and mounts given to a container may provide 
		incomplete isolation
		
Docker supports the addition and removal of capabilities
	allowing use of a non-default profile. 
	This may make Docker more secure through capability removal, or less secure through the addition of capabilities. The best practice for users would be to remove all capabilities except those explicitly required for their processes.

#Docker Content Trust Signature Verification
The Docker Engine can be configured to only run signed images. 
The Docker Content Trust signature verification feature 
	built directly into the dockerd binary.

This is configured in the Dockerd configuration file.
To enable this feature
	trustpinning can be configured in daemon.json
	whereby only repositories signed with a user-specified root key can be pulled and run.

Provides more insight to administrators 
	than previously available with the CLI 
	for enforcing and performing image signature verification.

More details: https://docs.docker.com/engine/security/trust/

#Other kernel security features
Capabilities 
	security feature provided by modern Linux kernels. 
	Possible to inegrate existing Docker with well-known systems like 
		TOMOYO, 
		AppArmor, 
		SELinux, 
		GRSEC, 
		etc. with Docker.

While Docker currently only enables capabilities
	it doesn’t interfere with the other systems. 
	
So there are different ways to harden a Docker host. 
For examples.
	Run a kernel with GRSEC and PAX. 
		This adds many safety checks, both at compile-time and run-time; it also defeats many exploits, thanks to techniques like address randomization. It doesn’t require Docker-specific configuration, since those security features apply system-wide, independent of containers.

	Docker ships a template that works with AppArmor
	Red Hat comes with SELinux policies for Docker. 
	These templates provide an extra safety net 
	N.B: It may overlap greatly with capabilities.

	Define your own policies using your favorite access control mechanism.
	Use third-party tools to augment Docker containers, including special network topologies or shared filesystems, tools exist to harden Docker containers without the need to modify Docker itself.

As of Docker 1.10 User Namespaces are supported directly by the docker daemon. This feature allows for the root user in a container to be mapped to a non uid-0 user outside the container, which can help to mitigate the risks of container breakout. This facility is available but not enabled by default.

Refer to the daemon command in the command line reference for more information on this feature. Additional information on the implementation of User Namespaces in Docker can be found in this blog post.


##List of vulnerablities already fixed
https://docs.docker.com/engine/security/non-events/

## Protect Docker Daemon Host 
https://docs.docker.com/engine/security/protect-access/


#Protect the Docker daemon socket

By default, 
	Docker runs through a non-networked UNIX socket. 
	OPTIONALLY communicate using SSH or a TLS (HTTPS) socket.

#Use SSH to protect the Docker daemon socket

The given USERNAME 
	must have permissions to access the docker socket on the remote machine. 
	How to give a non-root user access to the docker socket.
		Refer: https://docs.docker.com/engine/install/linux-postinstall/#manage-docker-as-a-non-root-user
	

Creates a docker context to connect with a remote dockerd daemon on host1.example.com using SSH, and as the docker-user user on the remote machine:

 docker context create \
    --docker host=ssh://docker-user@host1.example.com \
    --description="Remote engine" \
    my-remote-engine
After creating the context, use docker context use to switch the docker CLI to use it, and to connect to the remote engine:

	docker context use my-remote-engine
	docker info

Use the default context to switch back to the default (local) daemon:
	docker context use default

------------------------------------------------
Alternatively, use the DOCKER_HOST environment variable to temporarily switch the docker CLI to connect to the remote host using SSH. This does not require creating a context, and can be useful to create an ad-hoc connection with a different engine:

 export DOCKER_HOST=ssh://docker-user@host1.example.com
 docker info
------------------------------------------------


#Use TLS (HTTPS) to protect the Docker daemon socket
To make Docker reachable through HTTP (not SSH) in a safe manner
	enable TLS (HTTPS) 
	specify the tlsverify flag 
		pointing Docker’s tlscacert flag to a trusted CA certificate.

In the daemon mode,
	(dockerd -D #more details: https://docs.docker.com/engine/reference/commandline/dockerd/)
	it only allows connections from clients authenticated by a certificate signed by that CA.
	In the client mode, it only connects to servers with a certificate signed by that CA.


Create a CA, server and client keys with OpenSSL
Note: Replace all instances of $HOST in the following example with the DNS name of your Docker daemon’s host.

First, on the Docker daemon’s host machine, generate CA private and public keys:

For TLS and SSH

Reference: https://docs.docker.com/engine/security/protect-access/

fyi: Execute with root - sudo su mode
Replace $HOST with private host name.
For the following it is fine to give private and public IP and DNS

echo subjectAltName = DNS:$HOST,IP:10.10.10.20,IP:127.0.0.1 >> extfile.cnf
echo subjectAltName = DNS:<Public DNS>,DNS:<Private DNS>,IP:<Public IP>,IP:127.0.0.1,IP:<Private IP> >> extfile.cnf


----------------------------------------
#Verify repository client with certificates
	The right way to configure the client with the certificates generated from the server is described here..
	
	Watch out for https://docs.docker.com/engine/security/certificates/
	
	
-------------------------------------------
SSH access from one machine to another
ssh-keygen
Hit enter enter enter. You'll have two files:

.ssh/id_rsa
.ssh/id_rsa.pub
On Server A, cat and copy to clipboard the public key:

cat ~/.ssh/id_rsa.pub
[select and copy to your clipboard]
ssh into Server B, and append the contents of that to the it's authorized_keys file:

cat >> ~/.ssh/authorized_keys
[paste your clipboard contents]
[ctrl+d to exit]
Now ssh from server A:

ssh -i ~/.ssh/id_rsa private.ip.of.other.server


------------------
Reference of SSH: same as above "
SSH may not work due to the bug: https://github.com/docker/compose-cli/issues/811
docker context create \
    --docker host=ssh://centos@ip-172-31-77-41.ec2.internal \
    --description="Remote engine" \
    my-remote-engine	
	
 docker context create test --docker "host=ssh://172.31.77.41"	
 
 
##Antivirus software and Docker
	https://docs.docker.com/engine/security/antivirus/
	
##AppArmor security profiles for Docker
	https://docs.docker.com/engine/security/apparmor/
	

AppArmor (Application Armor) 
	Linux security module 
	protects an operating system and its applications from security threats. 
	To use it, 
		associate an AppArmor security profile with each program. 
	Docker expects to find an AppArmor policy loaded and enforced.

Docker automatically generates and loads a default profile for containers named docker-default. 
The Docker binary generates this profile in tmpfs and then loads it into the kernel.

Note: This profile is used on containers, not on the Docker Daemon.

A profile for the Docker Engine daemon exists 
	but it is not currently installed with the deb packages. 
If you are interested in the source for the daemon profile
	it is located in contrib/apparmor in the Docker Engine source repository.

Understand the policies
The docker-default profile is the default for running containers. 
It is moderately protective while providing wide application compatibility. 
The profile is generated from the following template.

When you run a container, 
	it uses the docker-default policy unless you override it with the security-opt option. 
	For example, the following explicitly specifies the default policy:

	$ docker run --rm -it --security-opt apparmor=docker-default hello-world	

Load and unload profiles
To load a new profile into AppArmor for use with containers:

$ apparmor_parser -r -W /path/to/your_profile
Then, run the custom profile with --security-opt like so:

$ docker run --rm -it --security-opt apparmor=your_profile hello-world
To unload a profile from AppArmor:

# unload the profile
$ apparmor_parser -R /path/to/profile
Resources for writing profiles
The syntax for file globbing in AppArmor is a bit different than some other globbing implementations. It is highly suggested you take a look at some of the below resources with regard to AppArmor profile syntax.

Quick Profile Language
Globbing Syntax
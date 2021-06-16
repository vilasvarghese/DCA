
Create User
useradd -m -d /home/lowprivuser -p $(openssl passwd -1 password) lowprivuser

Login 
sudo su lowprivuser
	pwd: password
	
When running as this user, a couple of items change. For example, the user is not able to create or change files in certain locations such as the root directory, 
	touch /root/blocked.
	
	
	docker ps # would fail as the user doesn't have permission
	
Install rootless docker
	curl -sSL https://get.docker.com/rootless | sh
	
export XDG_RUNTIME_DIR=/tmp/docker-1001
export PATH=/home/lowprivuser/bin:$PATH
export DOCKER_HOST=unix:///tmp/docker-1001/docker.sock
mkdir -p $XDG_RUNTIME_DIR

/home/lowprivuser/bin/dockerd-rootless.sh --experimental --storage-driver vfs

sudo su lowprivuser

export XDG_RUNTIME_DIR=/tmp/docker-1001
export PATH=/home/lowprivuser/bin:$PATH
export DOCKER_HOST=unix:///tmp/docker-1001/docker.sock

docker ps
docker info | grep vfs
docker info | grep rootless

docker run -it ubuntu bash


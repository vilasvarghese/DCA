docker run -d -p 5000:5000 --restart=always --name registry registry:2

Pull the ubuntu:16.04 image from Docker Hub.
	$ docker pull ubuntu:16.04


Tag the image as localhost:5000/my-ubuntu. This creates an additional tag for the existing image. When the first part of the tag is a hostname and port, Docker interprets this as the location of a registry, when pushing.
	$ docker tag ubuntu:16.04 localhost:5000/my-ubuntu


Push the image to the local registry running at localhost:5000:
	$ docker push localhost:5000/my-ubuntu


Remove the locally-cached ubuntu:16.04 and localhost:5000/my-ubuntu images, so that you can test pulling the image from your registry. This does not remove the localhost:5000/my-ubuntu image from your registry.
Vilas added
	$ docker container prune
	
	$ docker image remove ubuntu:16.04
	$ docker image remove localhost:5000/my-ubuntu


Pull the localhost:5000/my-ubuntu image from your local registry.
	$ docker pull localhost:5000/my-ubuntu
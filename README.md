# Devpi Server Docker image

This repo contains a docker image for the [devpi](http://doc.devpi.net/latest/) PyPI server along with the configuration files needed for the deployment on [Google Container Engine](https://cloud.google.com/container-engine/) (Kubernetes).

The built image is available on Docker Hub at [fyndiq/docker-devpi](https://hub.docker.com/r/fyndiq/docker-devpi/)

By default a private index will be created that requires HTTP Basic Authentication to access it.

## Configuration

The configuration is specified with the following environment variables:

* `DEVPI_SERVERDIR`: Directory for server files (default: `/data/server`)
* `DEVPI_CLIENTDIR`: Directory for client files  (default: `/data/client`)
* `DEVPI_ROOT_PASSWORD`: Password for root user (user who can create or 
modify indexes)
* `DEVPI_USER`: Username
* `DEVPI_PASSWORD`: Password
* `DEVPI_INDEX`: Index name (default: `dev`. The index where custom packages can be uploaded)

Most of them will be created via Kubernetes Secrets (shown below).

## Running locally (docker-compose)

To test the setup on a local machine run it with [docker-compose](https://docs.docker.com/compose/):

    docker-compose up

Note that compared to the Kubernetes setup this will not start nginx.
The server is running at `http://localhost:3141`. User credentials are stored
in `docker-compose.yml`.

## Running on Google Container Engine

The configuration for Kubernetes is stored in the `devpi-app.yaml` file.

In addition to the devpi-server the pod will include nginx with
HTTP Basic Authentication configured, so that the server is not publicly
available.

### Setup

First, the HTTP Basic Auth htpasswd file needs to be generated. Note that due to a [issue with devpi-client](https://bitbucket.org/hpk42/devpi/issues/331/basic-auth-devpi), the http auth and devpi auth credentials need to be the same.

    htpasswd -bn testuser testpassword > htpasswd

The Kubernetes Secret can be created with the following command:

	kubectl create secret generic devpi \
        --from-literal=root-password=pleasechangeme \
		--from-literal=user=testuser \
		--from-literal=password=testpassword \
		--from-file=htpasswd

A configmap for nginx needs to be created (If you want to remove the authentication part and run a publicly available index you can modify the config at this step):

	kubectl create configmap nginx-conf --from-file=nginx.conf

To have persistent storage a Google Compute Disk needs to be created. It must
have the name `devpi-disk`.

	gcloud compute disks create --size=10GB devpi-disk

### Deployment

To deploy the service:

	kubectl create -f devpi-app.yaml

To get the external ip address (listed under `LoadBalancer Ingress`):

	kubectl describe -f devpi-app.yaml

To update the service:

	kubectl apply -f devpi-app.yaml

To delete the service:

    kubectl delete -f devpi-app.yaml

### Testing the index

	pip install -i http://testuser:testpassword@192.168.99.99/myuser/dev/+simple/ --trusted-host 192.168.99.99 Flask

To configure pip to permanently to use the new index create a `~/.pip/pip.conf` file with the following content:

	[global]
	index-url = http://testuser:testpassword@192.168.99.99/myuser/pypi/+simple/
	trusted-host = 192.168.99.99

## Example: Building and uploading a Wheel for Pandas

The following example shows how to build a wheel for [Pandas](http://pandas.pydata.org/) and upload it to the index. Make sure you have the [devpi-client](https://pypi.python.org/pypi/devpi-client) installed.

    git clone https://github.com/pydata/pandas.git
    cd pandas
    git checkout v0.16.2

    devpi use http://testuser:testpassword@192.168.99.99/myuser/dev
    devpi login myuser --password=testpassword
    devpi upload --formats bdist_wheel

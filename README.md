![GitHub release (latest by date)](https://img.shields.io/github/v/release/fly-apps/postgres-flex)
[![DeepSource](https://deepsource.io/gh/fly-apps/postgres-flex.svg/?label=active+issues&token=VOdkBvMAf90cLzNVB3k0WpJC)](https://deepsource.io/gh/fly-apps/postgres-flex/?ref=repository-badge)

# High Availability Postgres on Fly.io
This repo contains all the code and configuration necessary to run a [highly available Postgres cluster](https://fly.io/docs/postgres/) in a Fly.io organization's private network. This source is packaged into [Docker images](https://hub.docker.com/r/flyio/postgres-flex/tags) which allow you to track and upgrade versions cleanly as new features are added.


## Getting started
```bash
# Be sure you're running the latest version of flyctl.
fly version update

# Provision a 3 member cluster
fly pg create --name <app-name> --initial-cluster-size 3 --region ord --flex
```

## High Availability
For HA, it's recommended that you run at least 3 members within your primary region. Automatic failovers will only consider members residing within your primary region. The primary region is represented as an environment variable defined within the `fly.toml` file.

## Horizontal scaling
Use the clone command to scale up your cluster.
```bash
# List your active Machines
fly machines list --app <app-name>

# Clone a machine into a target region
fly machines clone <machine-id> --region <target-region>
```

## Staying up-to-date!
This project is in active development so it's important to stay current with the latest changes and bug fixes.

```bash
# Use the following command to verify you're on the latest version.
fly image show --app <app-name>

# Update your Machines to the latest version.
fly image update --app <app-name>
```

## PostgreSQL 16

Because Fly doesn't have a PostgreSQL 16 image to use as the basis for a new cluster, and that updating from PostgreSQL 15
to 16 isn't automatic, the process of getting a cluster up is a bit different.

First, you need Docker running locally, and a Docker account.  You'll be pushing a Docker image to Docker Hub to use
when you create your cluster.

### Build the Docker image

Use a script like the below, setting your Docker username

```bash
#!/bin/bash
set -e

VERSION=v0.0.50
DOCKER_USERNAME=<here>

docker build -f Dockerfile-pg16 \
             -t ${DOCKER_USERNAME}/postgres-16-flex \
             -t ${DOCKER_USERNAME}/postgres-16-flex:$VERSION \
             --build-arg VERSION=$VERSION \
             . \
             --platform "linux/amd64"

docker push ${DOCKER_USERNAME}/postgres-16-flex:$VERSION
docker push ${DOCKER_USERNAME}/postgres-16-flex
```

(Pushing the versioned tag is optional, if you want to omit that.)

You'll then need to go to Docker Hub and set the `postgres-16-flex` repository as public.

### Create a cluster using the image

The rest of the process is per above, just supplying your image to the pg create command.

```bash
fly pg create --image-ref your-username/postgres-16-flex --name <app-name> --initial-cluster-size 3 --region ord --flex
```

### Updating the cluster

You're responsible for updating your own image now, of course.  Rebuild the Docker image using the above script,
bumping the version number.  Once you've pushed your new build, you can use the "Staying up-to-date!" section above to 
update your cluster.

## TimescaleDB support
We currently maintain a separate TimescaleDB-enabled image that you can specify at provision time.

```bash
fly pg create  --image-ref flyio/postgres-flex-timescaledb:15
```

## Citus support
This repository contains unofficial support for Citus (https://www.citusdata.com/).  This isn't supported by either
Citus or Fly (at the moment!)

```bash
# Create the pg app - using whatever arguments you desire
fly pg create -n <coordinator-app-name>

# Update the app to use the Citus-enabled image
fly deploy -a <coordinator-app-name> --dockerfile Dockerfile-citus --build-arg VERSION=v0.0.43

# Add the extension to your database
fly pg connect -a <coordinator-app-name>
> \c db-name
> CREATE EXTENSION citus;
```

Add a couple more apps for a Citus worker

```bash
fly pg create -n <worker1-app-name>
fly deploy -a <worker1-app-name> --dockerfile Dockerfile-citus --build-arg VERSION=v0.0.43
fly pg connect -a <worker1-app-name>
> \c db-name
> CREATE EXTENSION citus;
```

```bash
fly pg create -n <worker2-app-name>
fly deploy -a <worker2-app-name> --dockerfile Dockerfile-citus --build-arg VERSION=v0.0.43
fly pg connect -a <worker2-app-name>
> \c db-name
> CREATE EXTENSION citus;
```

Then continue as per https://docs.citusdata.com/en/stable/installation/multi_node_debian.html#steps-to-be-executed-on-the-coordinator-node, substituting the hostnames :

```bash
coordinator-app-name.internal
worker1-app-name.internal
worker2-app-name.internal
```

You can now use the co-ordinator as your DATABASE_URL.

You can then scale each individual app as needed, or add more workers!

### TODO

Add a user to .pgpass for the Citus connections, and add a more restrictive entry to pg_bha.conf
 - This is tricky, the .pgpass needs each worker in it I think
Add some tests for pg_hba.conf and (when implemented) .pgpass

## Having trouble?
Create an issue or ask a question here: https://community.fly.io/

## Contributing
If you're looking to get involved, fork the project and send pull requests.

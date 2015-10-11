# btsync.docker
A **40MB** docker image sync service built for persist data on clusters

```
+------+    +----------+
|  NFS |----| Your App |
+------+    +----------+
```


Imagine a NFS server, but made for cluster and containers.

## Reliability
Ready for production! Until now, its the only BTSync docker imaged that is
harassed with a test suite!

## Motivation

#### CLUSTER PERSISTENT STORAGE

At [findhit.com](https://findhit.com), we have to share data between our cluster
nodes. After testing out some solutions such as: cloud provider's storage ones,
some well known cluster-targeted file systems and among other less valuable
solutions to our architecture (such as NAS sharing), we've got always the same
opinion. None of them could be fast, scalable and easy as plug-n-play.

As so, we looked for p2p syncing solutions. BTSync was our first choice, but it
lacks some important features such as cluster-based configuration storages
(`etcd` for example).

I took a time to plan how I would structure configurations on a cluster, but it
always ends on configurations per node, the only difference with current BTSync
approach would be configuration sharing, and thats not useful.

So, instead of creating an image that relied on `confd` (which is great btw),
I've decided to work on btsync cli before going to sleep.

## How does it works?

A cli will allow us to place a btsync service per cluster node (making use out of
the ephemeral disk they mostly have) and control that service via `docker exec`.

### Service structure example
NOTE: Examples presented  above will rely on a CoreOS environment.

- Submitting a btsync global service (with fleet) will allow us to share one
path (my choice was `/mnt/resource`, which is an ephemeral storage on CoreOS on MS Azure)
- Services could easily ensure or remove a folder by running:
  - `docker exec btsync ctl add gitlab` on `ExecStartPre`
  - `docker exec btsync ctl del gitlab` on `ExecStopPost`
- Storage could be mounted by sharing `/data` volume of `btsync` container, but I really advise you to don't do so. `/data` could have other containers data
and that fact creates a big SECURITY WARNING ON MY HEAD!!! Instead we could use
a path getter such as:
`docker run -v $(docker exec btsync ctl path gitlab):/home/gitlab/data gitlab:latest`

## Usage

There are several ways to configure this service. They were created to fit on
your services structure.

Note: Although they don't mention how to bind data folder into a disk, you could
achieve by mounting it as a volume: `-v /path/to/some/host/disk:/data`

### Launching btsync container

#### As a NFS-redundant Stack Service (RECOMMENDED)

```bash
docker run \
    --name btsync-nfs \
    bootstrap SECRET_HERE some-third-party-image

# Now you just have to make sure your image is able to mount it
# In case the image you pretend doesn't do that, you could work around it by
# replacing entrypoint and command. (Advanced users)
docker run \
    --name some-third-party-image \
    -e CONTAINER_DATA_PATH=/container/data/path \
    --entrypoint /bin/sh \
    some-third-party-image: latest \
    -s -c "mount -t nfs4 btsync-nfs:/some-third-party-image /container/data/path && /path/to/original/entrypoint and/or command;"
```

Shares are exposed on NFS root system, meaning that if you create a namespace
called `yolo` (with `ctl add yolo`), you nfs mount command should look like:
```
mount -t nfs4 btsync.link.or.ip:/yolo /path/to/mount/on
```

#### As a Global Service way

```bash
docker run \
    --name btsync \
    cusspvz/btsync:latest
```

Now each service should add his folder data before mounting it:

```bash
docker exec btsync ctl add --secret="SECRET_HERE" my-app-name
docker run --name="my-app-name" -v /path/to/some/host/disk/my-app-name:/container/data/path my-app-name

# Removing it from global service
docker exec btsync ctl del my-app-name
```

#### As a sidekick volumes-from

```bash
docker run \
    --name my-app-name-data \
    cusspvz/btsync:latest \
    bootstrap SECRET_HERE my-app-name

docker run \
    --volumes-from my-app-name-data \
    -e DATA_DIR=/data/my-app-name \
    my-app-name
```

## Accessing BTSync WebGUI

BTSync WebGUI is available by default at port **8888**.

You could set up an internal load-balancer just to see whats happening under the
hood. But please, avoid to give public access to it, otherwise MR Robot will
nuke all your data, including you chinese servers backup with a Raspberry Pi...

Default credentials:
User: admin
Pass: admin

## Commands

### Running commands on container
```bash
docker exec btsync ctl [commands]
```

### add [--secret=""] namespace
Adds a folder for a determined namespace

```bash
docker exec btsync ctl add --secret="e3hryu35qegqery4w5y164u5u" "gitlab"
```

NOTE: Secret provided on example isn't a valid one, it was generated by a random
pseudo-geek key typing. I'm on a coffee, now people here think I write fast.

### del namespace
Deletes folder related with that namespace

```bash
docker exec btsync ctl del "gitlab"
```

### path [--scope=(host|container)] namespace
Returns host's or container's path to created namespace.
By default, scope is host.

```bash
docker exec btsync ctl path "gitlab" # /mnt/resources/sync/gitlab
```

### has namespace
Exits with 0 (has) or 1 (hasn't) indicating if namespace exists [or not]

```bash
if docker exec btsync ctl has "gitlab"; then
    echo "yeah, we already have that namespace"
fi
```

### list
Prints a spaced-separated list of namespaces

```bash
docker exec btsync ctl add gitlab
docker exec btsync ctl add star-trek
docker exec btsync ctl list # gitlab star-trek
```

## Environment Variables

### HOST_DATA_PATH
Defaults to: /mnt/resources

### DATA_PATH
Defaults to: /data

### CONFIG_PATH
Defaults to: /data/btsync.conf

### CONFIG_INTERVAL_CHECK
Defaults to: 10

### PID_PATH
Defaults to: /var/run/btsync.pid

### DEBUG
Defaults to: "btsync:ctl"

### UNAME
Defaults to: btysnc

### UID
Defaults to: 1000

### GID
Defaults to: 1000

## Development

##### build and run a container
`make run`

##### build and run a container with bash command
`make run-bash`

##### just building a container
`make build`

##### build image, build test image, run multiple containers and harass them with test suite
`make test`

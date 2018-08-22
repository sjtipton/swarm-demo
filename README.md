# Swarm

## Scenario

Let's assume there is one application server serving _x_ amount of clients. If the intention is to serve _twice_ the amount of clients, you could increase the resources on the server load balancing the first server. However you go about this process, it is a repetitive manual process, because _as the need increases_, you will need to increase the cluster by adding application servers.

## Containers manually managed == Problems to be solved

* Container lifecycle automation
* Ease of scaling out/in/up/down
* Ensuring re-creation of failed containers
* Replacement of containers without downtime (aka "blue/green deploy")

## What is a swarm?

* Docker Swarm mode overview: https://docs.docker.com/engine/swarm/
* Swarm mode key concepts: https://docs.docker.com/engine/swarm/key-concepts/

Swarm is the cluster management and orchestration tooling built into Docker that essentially solves these problems. It is build using [swarmkit](https://github.com/docker/swarmkit/), which implements the orchestration layer that is utilized directly inside Docker.

* Consists of multiple Docker hosts running in **swarm mode** and act as managers and workers
* Docker host can be a manager, worker, or both
* When creating the service, you define its optimal state (e.g. number of replicas, network and storage resources, ports exposed, etc.)
* Docker is responsible for maintaining the desired state -- if a worker node becomes unavailable, Docker schedules that node's tasks on other nodes
* A _task_ is a running container belonging to the swarm service managed by the swarm manager, as opposed to a standalone container

A key advantage of swarm services vs. standalone containers is the pattern known as "blue/green deploy." Let's assume you modify a service's configuration -- networks and volumes connected to it as well. Docker will update the configuration, stop the service tasks with the out-of-date configuration, and create new ones with the desired state.

More about **Desired state reconciliation** from the [Swarm mode overview](https://docs.docker.com/engine/swarm/
) Feature highlights:

> **Desired state reconciliation:** The swarm manager node constantly monitors the cluster state and reconciles any differences between the actual state and your expressed desired state. For example, if you set up a service to run 10 replicas of a container, and a worker machine hosting two of those replicas crashes, the manager creates two new replicas to replace the replicas that crashed. The swarm manager assigns the new replicas to workers that are running and available.

## Nodes, Services and Tasks, and Load Balancing

### Nodes

* A **node** is an instance of the Docker engine participating in the swarm
* One or more nodes can be run on a single physical computer or cloud server; however, production deployments typically include nodes distributed across multiple physical and cloud machines
* Deploying an application to a swarm requires the submission of a service definition to a **manager node**. The manager node dispatches units of work called tasks to worker nodes
* Manager nodes perform orchestration and cluster management to maintain the desired state of the swarm, and elect a single leader to conduct orchestration
* **Worker nodes** receive and execute tasks dispatched from manager nodes
* Manager nodes also run services as worker nodes (by default), but can be configured to run manager tasks exclusively and be manager-only nodes
* An agent runs on the worker nodes and reports on the tasks assigned to it
* The worker node notifies the manager node of the current state of those tasks so the manager can handle the desired state of each worker

### Services and Tasks

* A **service** defines the tasks to execute on the manager or worker nodes
* When creating a service, the container image is specified, along with which commands to execute inside running containers (e.g. `docker service create alpine ping 8.8.8.8`)
* A **replicated service** is a scalable service instance in which the swarm manager distributes a defined number of replica tasks among the nodes based on the scale of the desired state (e.g. `docker service create --name hello --replicas 3 --publish 8080:80 nginx`)
* A **task** carries a container along with its commands to run within the container
* Manager nodes assign tasks to work nodes based on the number of replicas set in the service scale
* Tasks are immovable from one node to another once assigned

### Load Balancing

* The swarm manager uses **ingress load balancing** to expose the services you want externally available to the swarm
* Manager can auto-assign the service a **PublishedPort** in the `30000-32767` range, or it can be self-configured with any unused port
* External components (i.e. cloud load balancers) are able to access the service on the PublishedPort of any node within the cluster regardless of whether or not the node is currently running the service's task
* All nodes in the swarm route ingress connections to a running task instance
* Swarm mode has internal DNS that automatically assigns each service a DNS entry
* The swarm manager uses **internal load balancing** to distribute requests among services within the cluster based on the DNS name of the service

## Enabling Swarm

* Not enabled by default
* When running `docker swarm init`, the Docker Engine sets up the swarm by:
    * enabling swarm
    * creating a `default` swarm
    * designating the current node as a leader manager
    * naming the node with the machine hostname
    * configuring the manager to listen on the active network interface on port 2377
    * setting the current node to `Active` availability, allowing it to receive tasks from the scheduler
    * starting an internal distributed data store to maintain view consistency between services and any Engines participating in the swarm
    * generates a self-signed root CA, by default
    * generates tokens for worker and manager nodes to join the swarm, be default
    * creates an overlay network name `ingress` for publishing external service ports

```bash
$ docker swarm init
Swarm initialized: current node (dxn1zf6l61qsb1josjja83ngz) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join \
    --token SWMTKN-1-49nj1cmql0jkz5s954yi3oex3nedyz0fb0xx14ie39trti4wxv-8vxv8rssmk743ojnwacrr2e7c \
    192.168.99.100:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
```

## Advertise Address

Manager nodes utilize and advertise address to allow other nodes within the swarm to access the Swarmkit API and overlay networking. The other nodes must be able to access the manager node on its advertise address as well.

Without a specified advertise address, Docker will try to use the IP address listening on `2377` by default. If the system has multiple IP addresses, you must specify the correct `--advertise-addr` to enable inter-manager communication and overlay networking:

```bash
$ docker swarm init --advertise-addr <MANAGER-IP>
```

An example of this configuration is show in the demo being utilized in play.docker.com.

## View the join command/update a swarm join token

Nodes require a secret token to join a swarm. The token for worker nodes and manager nodes are not the same. Nodes only use the join-token at the moment they join the swarm. Rotating the token after joining does not affect a node's swarm membership. Rotation of a token ensures an old token cannot be used by new nodes attempting to join.

To retrieve the join command with the join token for work nodes:

```bash
$ docker swarm join-token worker

To add a worker to this swarm, run the following command:

    docker swarm join \
    --token SWMTKN-1-49nj1cmql0jkz5s954yi3oex3nedyz0fb0xx14ie39trti4wxv-8vxv8rssmk743ojnwacrr2e7c \
    192.168.99.100:2377

This node joined a swarm as a worker.
```

To view the join command and token for manager nodes:

```bash
$ docker swarm join-token manager

To add a worker to this swarm, run the following command:

    docker swarm join \
    --token SWMTKN-1-59egwe8qangbzbqb3ryawxzk3jn97ifahlsrw01yar60pmkr90-bdjfnkcflhooyafetgjod97sz \
    192.168.99.100:2377
```

Pass the `--quiet` flag to print the token only:

```bash
$ docker swarm join-token --quiet worker

SWMTKN-1-49nj1cmql0jkz5s954yi3oex3nedyz0fb0xx14ie39trti4wxv-8vxv8rssmk743ojnwacrr2e7c
```

## More info on Swarm mode

For more information on Swarm mode, the [Docker docs are your friend](https://docs.docker.com/engine/swarm/swarm-mode).

## Demo!

* Use play.docker.com
* Single instance for basic examples
* 3-node swarm
* Swarm Visualizer

### Initialize Swarm

_Note:_ Could copy/paste the `docker swarm join --token <token>`, or get it later as needed.

```bash
$ docker swarm init --advertise-addr <instance IP>
Swarm initialized: current node (98bw7xv33ryx0yzb05gksfiny) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join --token SWMTKN-1-5836ufdx8zspdccpirb3ezizt7eglbebqhapnzs4fikysl2pa3-36bcaxewlzgrxxv0enah0lg26 192.168.0.48:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
```

### Create Hello World service with 3 replicas using nginx published to port 8080

```bash
$ docker service create --name hello --replicas 3 --publish 8080:80 nginx
4zmjjn3ns97pm9v32zxtkb2y9
overall progress: 3 out of 3 tasks
1/3: running   [==================================================>]
2/3: running   [==================================================>]
3/3: running   [==================================================>]
verify: Service converged
```

### List running nodes

```bash
$ docker node ls
ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS      ENGINE VERSION
98bw7xv33ryx0yzb05gksfiny *   node1               Ready               Active              Leader              18.03.1-ce
```

### List service information

```bash
$ docker service ls
ID                  NAME                MODE                REPLICAS            IMAGE               PORTS
4zmjjn3ns97p        hello               replicated          3/3                 nginx:latest        *:8080->80/tcp
```

### List service tasks

```bash
# $ docker service ps [ID/NAME]
$ docker service ps hello
ID                  NAME                IMAGE               NODE                DESIRED STATE       CURRENT STATE            ERROR               PORTS
nyg72k0vw2kt        hello.1             nginx:latest        node1               Running             Running 10 minutes ago
kwqqd7g4nwlv        hello.2             nginx:latest        node1               Running             Running 10 minutes ago
qg0s1ooo3172        hello.3             nginx:latest        node1               Running             Running 10 minutes ago
```

### Create service with the alpine image that pings Google's DNS (8.8.8.8)

```bash
$ docker service create alpine ping 8.8.8.8
gz7521yo6tb4ocg2k5hlrcbb0
overall progress: 1 out of 1 tasks
1/1: running   [==================================================>]
verify: Service converged
```

### List service information

```bash
$ docker service ls
ID                  NAME                MODE                REPLICAS            IMAGE               PORTS
4zmjjn3ns97p        hello               replicated          3/3                 nginx:latest        *:8080->80/tcp
gz7521yo6tb4        relaxed_golick      replicated          1/1                 alpine:latest
```

### List service tasks

```bash
# $ docker service ps [ID/NAME]
$ docker service ps relaxed_golick
ID                  NAME                IMAGE               NODE                DESIRED STATE       CURRENT STATE           ERROR               PORTS
558e3zecikqt        relaxed_golick.1    alpine:latest       node1               Running             Running 4 minutes ago
```

### Update pinging service to be a replicated service with 3 replicas

```bash
# $ docker service update <ID> --replicas <NUM>
$ docker service update gz7521yo6tb4 --replicas 3
gz7521yo6tb4
overall progress: 3 out of 3 tasks
1/3: running   [==================================================>]
2/3: running   [==================================================>]
3/3: running   [==================================================>]
verify: Service converged
```

### Swarm help

```bash
$ docker swarm --help

Usage:  docker swarm COMMAND

Manage Swarm

Options:


Commands:
  ca          Display and rotate the root CA
  init        Initialize a swarm
  join        Join a swarm as a node and/or manager
  join-token  Manage join tokens
  leave       Leave the swarm
  unlock      Unlock swarm
  unlock-key  Manage the unlock key
  update      Update the swarm

Run 'docker swarm COMMAND --help' for more information on a command.

$ docker update --help
Usage:  docker update [OPTIONS] CONTAINER [CONTAINER...]

Update configuration of one or more containers

Options:
      --blkio-weight uint16        Block IO (relative weight), between 10 and 1000, or 0 to disable (default 0)
      --cpu-period int             Limit CPU CFS (Completely Fair Scheduler) period
      --cpu-quota int              Limit CPU CFS (Completely Fair Scheduler) quota
      --cpu-rt-period int          Limit the CPU real-time period in microseconds
      --cpu-rt-runtime int         Limit the CPU real-time runtime in microseconds
  -c, --cpu-shares int             CPU shares (relative weight)
      --cpus decimal               Number of CPUs
      --cpuset-cpus string         CPUs in which to allow execution (0-3, 0,1)
      --cpuset-mems string         MEMs in which to allow execution (0-3, 0,1)
      --kernel-memory bytes        Kernel memory limit
  -m, --memory bytes               Memory limit
      --memory-reservation bytes   Memory soft limit
      --memory-swap bytes          Swap limit equal to memory plus swap: '-1' to enable unlimited swap
      --restart string             Restart policy to apply when a container exits

$ docker service update --help # blue/green
Usage:  docker service update [OPTIONS] SERVICE

Update a service

Options:
      --args command                       Service command args
      --config-add config                  Add or update a config file on a service
      --config-rm list                     Remove a configuration file
      --constraint-add list                Add or update a placement constraint
      --constraint-rm list                 Remove a constraint
      --container-label-add list           Add or update a container label
      --container-label-rm list            Remove a container label by its key
      --credential-spec credential-spec    Credential spec for managed service account (Windows only)
  -d, --detach                             Exit immediately instead of waiting for the service to converge
      --dns-add list                       Add or update a custom DNS server
      --dns-option-add list                Add or update a DNS option
      --dns-option-rm list                 Remove a DNS option
      --dns-rm list                        Remove a custom DNS server
      --dns-search-add list                Add or update a custom DNS search domain
      --dns-search-rm list                 Remove a DNS search domain
      --endpoint-mode string               Endpoint mode (vip or dnsrr)
      --entrypoint command                 Overwrite the default ENTRYPOINT of the image
      --env-add list                       Add or update an environment variable
      --env-rm list                        Remove an environment variable
      --force                              Force update even if no changes require it
      --generic-resource-add list          Add a Generic resource
      --generic-resource-rm list           Remove a Generic resource
      --group-add list                     Add an additional supplementary user group to the container
      --group-rm list                      Remove a previously added supplementary user group from the container
      --health-cmd string                  Command to run to check health
      --health-interval duration           Time between running the check (ms|s|m|h)
      --health-retries int                 Consecutive failures needed to report unhealthy
      --health-start-period duration       Start period for the container to initialize before counting retries towards unstable (ms|s|m|h)
      --health-timeout duration            Maximum time to allow one check to run (ms|s|m|h)
      --host-add list                      Add a custom host-to-IP mapping (host:ip)
      --host-rm list                       Remove a custom host-to-IP mapping (host:ip)
      --hostname string                    Container hostname
      --image string                       Service image tag
      --isolation string                   Service container isolation mode
      --label-add list                     Add or update a service label
      --label-rm list                      Remove a label by its key
      --limit-cpu decimal                  Limit CPUs
      --limit-memory bytes                 Limit Memory
      --log-driver string                  Logging driver for service
      --log-opt list                       Logging driver options
      --mount-add mount                    Add or update a mount on a service
      --mount-rm list                      Remove a mount by its target path
      --network-add network                Add a network
      --network-rm list                    Remove a network
      --no-healthcheck                     Disable any container-specified HEALTHCHECK
      --no-resolve-image                   Do not query the registry to resolve image digest and supported platforms
      --placement-pref-add pref            Add a placement preference
      --placement-pref-rm pref             Remove a placement preference
      --publish-add port                   Add or update a published port
      --publish-rm port                    Remove a published port by its target port
  -q, --quiet                              Suppress progress output
      --read-only                          Mount the containers root filesystem as read only
      --replicas uint                      Number of tasks
      --reserve-cpu decimal                Reserve CPUs
      --reserve-memory bytes               Reserve Memory
      --restart-condition string           Restart when condition is met ("none"|"on-failure"|"any")
      --restart-delay duration             Delay between restart attempts (ns|us|ms|s|m|h)
      --restart-max-attempts uint          Maximum number of restarts before giving up
      --restart-window duration            Window used to evaluate the restart policy (ns|us|ms|s|m|h)
      --rollback                           Rollback to previous specification
      --rollback-delay duration            Delay between task rollbacks (ns|us|ms|s|m|h)
      --rollback-failure-action string     Action on rollback failure ("pause"|"continue")
      --rollback-max-failure-ratio float   Failure rate to tolerate during a rollback
      --rollback-monitor duration          Duration after each task rollback to monitor for failure (ns|us|ms|s|m|h)
      --rollback-order string              Rollback order ("start-first"|"stop-first")
      --rollback-parallelism uint          Maximum number of tasks rolled back simultaneously (0 to roll back all at once)
      --secret-add secret                  Add or update a secret on a service
      --secret-rm list                     Remove a secret
      --stop-grace-period duration         Time to wait before force killing a container (ns|us|ms|s|m|h)
      --stop-signal string                 Signal to stop the container
  -t, --tty                                Allocate a pseudo-TTY
      --update-delay duration              Delay between updates (ns|us|ms|s|m|h)
      --update-failure-action string       Action on update failure ("pause"|"continue"|"rollback")
      --update-max-failure-ratio float     Failure rate to tolerate during an update
      --update-monitor duration            Duration after each task update to monitor for failure (ns|us|ms|s|m|h)
      --update-order string                Update order ("start-first"|"stop-first")
      --update-parallelism uint            Maximum number of tasks updated simultaneously (0 to update all at once)
  -u, --user string                        Username or UID (format: <name|uid>[:<group|gid>])
      --with-registry-auth                 Send registry authentication details to swarm agents
  -w, --workdir string                     Working directory inside the container
```

### Perform a rogue removal of a container to see how the container orchestration is handled by Swarm

```bash
# $ docker container rm -f <NAME>.1.<ID>
$ docker container ls # <NAMES> field
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS               NAMES
8fa7faa344e3        alpine:latest       "ping 8.8.8.8"           8 minutes ago       Up 8 minutes                            relaxed_golick.3.plzginj3y2bersge03ip91k1o
aa90115a6375        alpine:latest       "ping 8.8.8.8"           8 minutes ago       Up 8 minutes                            relaxed_golick.2.ra9byjbz0ah8y6linvozx8cnh
0dd2d231ecd3        alpine:latest       "ping 8.8.8.8"           16 minutes ago      Up 16 minutes                           relaxed_golick.1.558e3zecikqtbump1t2oblcoi
6a46e803dc8e        nginx:latest        "nginx -g 'daemon of…"   38 minutes ago      Up 38 minutes       80/tcp              hello.2.kwqqd7g4nwlv58vltszx5qwk8
62f1626710df        nginx:latest        "nginx -g 'daemon of…"   38 minutes ago      Up 38 minutes       80/tcp              hello.3.qg0s1ooo3172rzaa0ab18fe58
b25c2de42e50        nginx:latest        "nginx -g 'daemon of…"   38 minutes ago      Up 38 minutes       80/tcp              hello.1.nyg72k0vw2ktrw0wlbc5jbzix

$ docker container rm -f hello.1.nyg72k0vw2ktrw0wlbc5jbzix
hello.1.nyg72k0vw2ktrw0wlbc5jbzix

$ docker service ls # if you're quick enough, shows 2/3 services
ID                  NAME                MODE                REPLICAS            IMAGE               PORTS
4zmjjn3ns97p        hello               replicated          2/3                 nginx:latest        *:8080->80/tcp
gz7521yo6tb4        relaxed_golick      replicated          3/3                 alpine:latest

$ docker service ls # relaunches a new replacement services within seconds
ID                  NAME                MODE                REPLICAS            IMAGE               PORTS
4zmjjn3ns97p        hello               replicated          3/3                 nginx:latest        *:8080->80/tcp
gz7521yo6tb4        relaxed_golick      replicated          3/3                 alpine:latest

$ docker service ps hello # historical representation of task
ID                  NAME                IMAGE               NODE                DESIRED STATE       CURRENT STATE            ERROR                         PORTS
teklprpfs5w0        hello.1             nginx:latest        node1               Running             Running 3 minutes ago
nyg72k0vw2kt         \_ hello.1         nginx:latest        node1               Shutdown            Failed 3 minutes ago     "task: non-zero exit (137)"
kwqqd7g4nwlv        hello.2             nginx:latest        node1               Running             Running 43 minutes ago
qg0s1ooo3172        hello.3             nginx:latest        node1               Running             Running 43 minutes ago
```

### Perform a proper service removal and see the orchestration queue in action

```bash
$ docker service rm hello
hello

$ docker service ls
ID                  NAME                MODE                REPLICAS            IMAGE               PORTS
gz7521yo6tb4        relaxed_golick      replicated          3/3                 alpine:latest

$ docker container ls # sometimes lags in the orchestration queue
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
8fa7faa344e3        alpine:latest       "ping 8.8.8.8"      15 minutes ago      Up 15 minutes                           relaxed_golick.3.plzginj3y2bersge03ip91k1o
aa90115a6375        alpine:latest       "ping 8.8.8.8"      15 minutes ago      Up 15 minutes                           relaxed_golick.2.ra9byjbz0ah8y6linvozx8cnh
0dd2d231ecd3        alpine:latest       "ping 8.8.8.8"      23 minutes ago      Up 23 minutes                           relaxed_golick.1.558e3zecikqtbump1t2oblcoi
```

### Create 3-node swarm

```bash
# node1
$ docker swarm init --advertise-addr <instance IP>
Swarm initialized: current node (j3yr0m1vlv6kcyqy8qige7xlq) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join --token SWMTKN-1-0qqupobcvwq9bxowqyf0smmdyx3tv6qfmehv8kptaauv00f8c9-512s10jh07e300inyb3x41wwr 192.168.0.48:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.

# copy-paste join-token to node2
$ docker swarm join --token SWMTKN-1-0qqupobcvwq9bxowqyf0smmdyx3tv6qfmehv8kptaauv00f8c9-512s10jh07e300inyb3x41wwr 192.168.0.48:2377
This node joined a swarm as a worker.

# node1: lists nodes with node1 manager status as "Leader"
$ docker node ls
ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS      ENGINE VERSION
j3yr0m1vlv6kcyqy8qige7xlq *   node1               Ready               Active              Leader              18.03.1-ce
my0nswmsmc2pq45vqs33hq5o8     node2               Ready               Active                                  18.03.1-ce

# node2: not a manager, can't do manager things
$ docker node ls
Error response from daemon: This node is not a swarm manager. Worker nodes cant be used to view or modify cluster state. Please run this command on a manager node or promote the current node to a manager.

# node1: update node2 to be a manager
$ docker node update --role manager node2
node2

# node2: lists nodes with node1 manager status as "Leader" and node2 manager status as "Reachable"
$ docker node ls
ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS      ENGINE VERSION
j3yr0m1vlv6kcyqy8qige7xlq     node1               Ready               Active              Leader              18.03.1-ce
my0nswmsmc2pq45vqs33hq5o8 *   node2               Ready               Active              Reachable           18.03.1-ce

# node1: get the join-token to add node3
$ docker swarm join-token manager
To add a manager to this swarm, run the following command:

    docker swarm join --token SWMTKN-1-0qqupobcvwq9bxowqyf0smmdyx3tv6qfmehv8kptaauv00f8c9-9ar6jnhoswqgx6oqzdztbqaag 192.168.0.48:2377

# node3: paste join-token
$ docker swarm join --token SWMTKN-1-0qqupobcvwq9bxowqyf0smmdyx3tv6qfmehv8kptaauv00f8c9-9ar6jnhoswqgx6oqzdztbqaag 192.168.0.48:2377
This node joined a swarm as a manager.

# node1: list nodes with node1 manager status as "Leader", and node2 and node3 manager statuses as "Reachable"
$ docker node ls
ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS      ENGINE VERSION
j3yr0m1vlv6kcyqy8qige7xlq *   node1               Ready               Active              Leader              18.03.1-ce
my0nswmsmc2pq45vqs33hq5o8     node2               Ready               Active              Reachable           18.03.1-ce
wjdr1inb3ahcc7uyygqwokl2o     node3               Ready               Active              Reachable           18.03.1-ce

# node1: create replicated swarm with 3 replicas using alpine image with the command to ping Google's DNS service (8.8.8.8)
$ docker service create --replicas 3 alpine ping 8.8.8.8
c6dx9i1x45crf429s44a87fs4
overall progress: 3 out of 3 tasks
1/3: running   [==================================================>]
2/3: running   [==================================================>]
3/3: running   [==================================================>]
verify: Service converged

# node1: list the service tasks currently running on the outputted container
$ docker node ps
ID                  NAME                 IMAGE               NODE                DESIRED STATE       CURRENT STATE                ERROR               PORTS
t1o7m94rdzng        gifted_blackwell.1   alpine:latest       node1               Running             Running about a minute ago

# node1: show containers running on node2
$ docker node ps node2
ID                  NAME                 IMAGE               NODE                DESIRED STATE       CURRENT STATE           ERROR               PORTS
liwv1yjog5i7        gifted_blackwell.2   alpine:latest       node2               Running             Running 2 minutes ago

# node1: list all tasks running, their current state, etc.
$ docker service ps gifted_blackwell
ID                  NAME                 IMAGE               NODE                DESIRED STATE       CURRENT STATE           ERROR               PORTS
t1o7m94rdzng        gifted_blackwell.1   alpine:latest       node1               Running             Running 3 minutes ago
liwv1yjog5i7        gifted_blackwell.2   alpine:latest       node2               Running             Running 3 minutes ago
jo7te1cofmkd        gifted_blackwell.3   alpine:latest       node3               Running             Running 3 minutes ago

# node1: scale the service
# $ docker service scale <SERVICE-ID>=<NUMBER>
$ docker service scale gifted_blackwell=5
gifted_blackwell scaled to 5
overall progress: 5 out of 5 tasks
1/5: running   [==================================================>]
2/5: running   [==================================================>]
3/5: running   [==================================================>]
4/5: running   [==================================================>]
5/5: running   [==================================================>]
verify: Service converged

# node1: list updated running tasks to see the scaled state
$ docker service ps gifted_blackwell
ID                  NAME                 IMAGE               NODE                DESIRED STATE       CURRENT STATE                ERROR               PORTS
t1o7m94rdzng        gifted_blackwell.1   alpine:latest       node1               Running             Running 9 minutes ago
liwv1yjog5i7        gifted_blackwell.2   alpine:latest       node2               Running             Running 9 minutes ago
jo7te1cofmkd        gifted_blackwell.3   alpine:latest       node3               Running             Running 9 minutes ago
pn3fp4he6468        gifted_blackwell.4   alpine:latest       node2               Running             Running about a minute ago
4i9won9100b1        gifted_blackwell.5   alpine:latest       node1               Running             Running about a minute ago

# node1: list running containers on the connected node
$ docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
6d2325535e5d        alpine:latest       "ping 8.8.8.8"      2 minutes ago       Up 2 minutes                            gifted_blackwell.5.4i9won9100b1t01xbh3wyl0le
bacc7f5b4603        alpine:latest       "ping 8.8.8.8"      10 minutes ago      Up 10 minutes                           gifted_blackwell.1.t1o7m94rdzngo36sre8ojeje7
```

### Swarm Visualizer

https://github.com/BretFisher/docker-swarm-visualizer

https://hub.docker.com/r/bretfisher/visualizer/


> Demo container that displays Docker services running on a Docker Swarm in a diagram.
>
> This works only with Docker swarm mode which was introduced in Docker 1.12. These instructions presume you are running on the master node and you already have a Swarm running.
>
> Each node in the swarm will show all tasks running on it. When a service goes down it'll be removed. When a node goes down it won't, instead the circle at the top will turn red to indicate it went down. Tasks will be removed.
>
> Occasionally the Remote API will return incomplete data, for instance the node can be missing a name. The next time info for that node is pulled, the name will update.

```bash
# node1: pull bretfisher/visualizer from Docker Hub
$ docker pull bretfisher/visualizer
Using default tag: latest
latest: Pulling from bretfisher/visualizer
88286f41530e: Pull complete
6a722742375f: Pull complete
7e9d2f284de4: Pull complete
da2fce695ca7: Pull complete
7465a3d10f0c: Pull complete
2b6c40acc71a: Pull complete
d00de06f06cc: Pull complete
0efb3b62dcd6: Pull complete
c6bbd7fdfa8b: Pull complete
fd9a75018b5a: Pull complete
1c6b95e5ce5e: Pull complete
713a6496787e: Pull complete
1b16a276e6da: Pull complete
Digest: sha256:479eb1f494e86a83c29a49ef9e763fde09490f1d4a8f1890808bd8fed8485de3
Status: Downloaded newer image for bretfisher/visualizer:latest

# node1: run in docker swarm
$ docker service create \
  --name=viz \
  --publish=8080:8080/tcp \
  --constraint=node.role==manager \
  --mount=type=bind,src=/var/run/docker.sock,dst=/var/run/docker.sock \
  dockersamples/visualizer
3gq3nyblr0va7gjfm65vgp4to
overall progress: 1 out of 1 tasks
1/1: running   [==================================================>]
verify: Service converged
```

![Swarm Visualizer](/visualizer.PNG "Swarm Visualizer Example")

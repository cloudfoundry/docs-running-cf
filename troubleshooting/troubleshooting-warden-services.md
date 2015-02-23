---
title: Troubleshooting Wardenized Services
---

## <a id="intro"></a>Introduction

This document shows how to debug a Wardenized service in a development or production environment.

## <a id="debug"></a>Debugging

This section includes the following topics, with corresponding examples:

* Checking the status of Warden server
* Checking the log and config of Warden
* Looking into a Warden container
* Service instance logs, data and localdb

### <a id="check-status"></a>Checking Status of Warden Server

The status of a Warden server is monitored by monit. You can see the Warden server status by using the `monit status` command:

<pre class="terminal">
$ monit status
The Monit daemon 5.2.4 uptime: 1d 1h 8m

Process 'warden'
status                            running
monitoring status                 monitored
pid                               4621
parent pid                        1
uptime                            1d 1h 7m
children                          8
memory kilobytes                  37112
memory kilobytes total            41708
memory percent                    0.2%
memory percent total              0.2%
cpu percent                       0.1%
cpu percent total                 0.1%
unix socket response time         0.000s to /tmp/warden.sock [DEFAULT]
data collected                    Tue Jan  8 07:11:52 2013
...
</pre>

If the Warden server is running normally, you can see the server status using `ps` and `grep`:

<pre class="terminal">
$ ps aux | grep warden
root      4621  0.1  0.2 125448 36644 ?        Sl   Jan07   2:37 ruby /var/vcap/data/packages/redis_node_ng/12.2-dev.1/warden/vendor/bundle/ruby/1.9.1/bin/rake warden:start[/var/vcap/jobs/redis_node_ng/config/warden.yml]
root      5342  0.0  0.0   5984   316 ?        S    06:43   0:00 /var/vcap/data/packages/redis_node_ng/12.2-dev.1/warden/src/oom/oom /sys/fs/cgroup/memory/instance-16io51f6jri
root      5592  0.0  0.0   5984   316 ?        S    06:43   0:00 /var/vcap/data/packages/redis_node_ng/12.2-dev.1/warden/src/oom/oom /sys/fs/cgroup/memory/instance-16io51f6jrj
root     15849  0.0  0.0   7688   844 pts/1    S+   07:10   0:00 grep --color=auto warden
</pre>

### <a id="check-log"></a>Checking the log and config of Warden

The config file of Warden is normally found at `/var/vcap/jobs/SERVICE-NODE/config/warden.yml`:

<pre class="terminal">
$ cat /var/vcap/jobs/redis_node_ng/config/warden.yml
---
server:
container_klass: Warden::Container::Linux
container_grace_time: ~
container_rootfs_path: /var/vcap/data/warden/rootfs
container_depot_path: /var/vcap/store/containers
unix_domain_permissions: 0777
container_rlimits:
  nofile: 2000
  # Redis-server forks a process to do dump,
  # so also increase the maximum number of processes,
  # the default limitation is 1024
  nproc: 2000
quota:
  disk_quota_enabled: false
logging:
file: /var/vcap/sys/log/warden/warden.log
syslog: vcap.services.redis.warden
level: debug
network:
pool_start_address: 10.254.0.0
pool_size: 4096
user:
pool_start_uid: 10000
pool_size: 4096
</pre>

The rootfs and container depot position is labelled in yellow and they could offer some information on debugging. The Warden log is labelled in green and it records whether the Warden server started correctly and how it handled incoming requests. The socket that the Warden server listens on is found at `/tmp/warden.sock`.

Warden uses [Steno](https://github.com/cloudfoundry/steno) for logging. You can use `steno-prettify` to make it more human readable. `steno-prettify` is a gem that is installed with Warden so you can use it without additional installation after going into the `warden` directory:

<pre class="terminal">
$ bundle exec steno-prettify /var/vcap/sys/log/warden/warden.log
2013-01-07 06:04:04.743379 Warden::Server pid=4621  tid=5798 fid=ee62 warden/server.rb/run!:240    DEBUG -- rlimit_nofile: 1024 => 32768
2013-01-07 06:04:05.372339 Warden::Container::Linux pid=4621  tid=5798 fid=ee62 container/spawn.rb/set_deferred_success:131    DEBUG -- Exited with status 0 (0.626s): [["/var/vcap/data/packages/redis_node_ng/12.2-dev.1/warden/src/closefds/closefds", "/var/vcap/data/packages/redis_node_ng/12.2-dev.1/warden/src/closefds/closefds"], "/var/vcap/data/packages/redis_node_ng/12.2-dev.1/warden/root/linux/setup.sh"]
2013-01-07 06:04:05.373243 Warden::Server pid=4621  tid=5798 fid=2add warden/server.rb/block (2 levels) in run!:282     INFO -- Listening on /tmp/warden.sock, and ready for action.
2013-01-07 06:04:21.776618 Warden::Server::Drainer pid=4621  tid=5798 fid=ee62 warden/server.rb/register_connection:53    DEBUG -- Connection registered: #<Warden::Server::ClientConnection:0x000000028fd7c8>
2013-01-07 06:04:21.777053 Warden::Server::Drainer pid=4621  tid=5798 fid=ee62 warden/server.rb/unregister_connection:60    DEBUG -- Connection unregistered: #<Warden::Server::ClientConnection:0x000000028fd7c8>
2013-01-07 06:04:31.784353 Warden::Server::Drainer pid=4621  tid=5798 fid=ee62 warden/server.rb/register_connection:53    DEBUG -- Connection registered: #<Warden::Server::ClientConnection:0x000000028e8df0>
2013-01-07 06:04:31.950572 Warden::Server::Drainer pid=4621  tid=5798 fid=ee62 warden/server.rb/unregister_connection:60    DEBUG -- Connection unregistered: #<Warden::Server::ClientConnection:0x000000028e8df0>
...
</pre>

You can also redirect the prettified log to a file to make it easier for navigation.

You way find the following script handy to set up an alias to run steno-prettify from any current directory:

<pre class="terminal">
#!/bin/bash

STENO_DIR=$(find /var/vcap/data/packages/ -name steno | grep vendor | head -n 1)
if [ $? -ne 0 ]; then
    echo "did not find steno in packages"
else
    export BUNDLE_GEMFILE=$(echo $STENO_DIR | sed s#/vendor/.*#/Gemfile# )
    if [ -f $BUNDLE_GEMFILE ]; then
        alias  steno-prettify="bundle exec steno-prettify"
        echo "ready to use steno-prettify alias, try steno-prettify on one of the following files:"
        find /var/vcap/sys/log/ -name "*.log" | egrep  -v "err|out|ctl" | xargs ls -al
    else
        echo "could not find Gemfile into ${STENO_DIR}:" `ls -al ${BUNDLE_GEMFILE}`
    fi
fi
</pre>

Save it to a file and source it, or add it to vcap bash profile:
<pre class="terminal">
# vi ~/setUpSteno.bash
[...]
# source ~/setUpSteno.bash
ready to use steno-prettify alias, try steno-prettify on one of the following files:
-rw-r--r-- 1 root root    2130 Feb 20 11:47 /var/vcap/sys/log/daylimit/daylimit.log
-rw-r--r-- 1 root root       0 Feb 20 10:45 /var/vcap/sys/log/rabbit_node/rabbit_logrotate_cron.log
-rw-r--r-- 1 root root 4179849 Feb 23 13:57 /var/vcap/sys/log/rabbit_node/rabbit_node.log
-rw-r--r-- 1 root root 9443178 Feb 23 13:41 /var/vcap/sys/log/rabbit/warden/warden.log

# steno-prettify /var/vcap/sys/log/rabbit/warden/warden.stdout.log
2015-02-20 10:41:29.678178 Warden::Server pid=3364  tid=6bfe fid=0ff4 warden/server.rb/run!:271 server={"unix_domain_path"=>"/tmp/warden.sock", "unix_domain_permissions"=>511, "container_klass"=>"Warden::Container::Linux", "container_grace_time"=>nil, "job_output_limit"=>10485760, "quota"=>{"disk_quota_enabled"=>false}, "allow_nested_warden"=>false, "container_rootfs_path"=>"/var/vcap/packages/rootfs_lucid64", "container_depot_path"=>"/var/vcap/data/rabbit/containers"},logging={"level"=>"debug", "file"=>"/var/vcap/sys/log/rabbit/warden/warden.log"},network={"pool_network"=>"10.254.0.0/18", "deny_networks"=>[], "allow_networks"=>[], "mtu"=>1500},port={"pool_start_port"=>61001, "pool_size"=>4000},user={"pool_start_uid"=>10000, "pool_size"=>4096}    INFO -- Configuration
[...]

# tail -f /var/vcap/sys/log/rabbit/warden/warden.stdout.log | steno-prettify
...
</pre>


### <a id="looking"></a>Looking into a Warden container

You can use the Warden client to talk to existing Warden containers and perform operations. The Warden client normally sits in `/path_to_warden_packages/bin/warden`:

<pre class="terminal">
$ ./bin/warden
warden> list
handles[0] : 16io51f6jri
handles[1] : 16io51f6jrj
</pre>

The Warden container is identified by its handle on a single node. You can run a script to get information in a Warden container:

<pre class="terminal">
$ ./bin/warden
warden> run --handle 16io51f6jri --script "echo hello world"
hello world
...
</pre>

In order to run operations with root privileges, you can use the `--privileged` option:

<pre class="terminal">
$ ./bin/warden
warden> run --handle 16io51f6jri --script "sudo echo hello world" --privileged
hello world
...
</pre>

The `help` option will give you more information on how to run a command or script within a Warden container.

<pre class="terminal">
$ ./bin/warden
warden> help

        copy_in       Copy files/directories into the container.
        copy_out      Copy files/directories out of the container.
        create        Create a container, optionally pass options.
        destroy       Shutdown a container.
        echo          Echo a message.
        info          Show metadata for a container.
        limit_cpu     Set or get the CPU limit in shares for the container.
        limit_disk    set or get the disk limit for the container.
        limit_memory  Set or get the memory limit for the container.
        link          Do blocking read on results from a job.
        list          List containers.
        net_in        Forward port on external interface to container.
        net_out       Allow traffic from the container to address.
        ping          Ping warden.
        run           Short hand for spawn(stream(cmd)) i.e. spawns a command, streams the result.
        spawn         Spawns a command inside a container and returns the job id.
        stop          Stop all processes inside a container.
        stream        Do blocking stream on results from a job.
        help          Show help.

Use --help with each command for more information.
warden> net_in --help
command: net_in
description: Forward port on external interface to container.
usage: net_in [options]

[options] can be one of the following:

        --handle <handle> (string)  # required
        [--host_port <host_port> (uint32)]  # optional
        [--container_port <container_port> (uint32)]  # optional
warden>

warden> net_in --handle 18fglrqcs23
host_port : 61013
container_port : 61013
</pre>

You may get additional networking details on warden containers virtual nic NAT, and received traffic:

<pre class="terminal">
$ iptables -t nat -L -v
[...]
Chain warden-instance-18fglrqcs23 (1 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 DNAT       tcp  --  any    any     anywhere             0.rabbit-node.services1.cf-services-contrib.bosh  tcp dpt:15009 to:10.254.4.170:10001
    0     0 DNAT       tcp  --  any    any     anywhere             0.rabbit-node.services1.cf-services-contrib.bosh  tcp dpt:25009 to:10.254.4.170:20001
    0     0 DNAT       tcp  --  any    any     anywhere             0.rabbit-node.services1.cf-services-contrib.bosh  tcp dpt:61003 to:10.254.4.170:61003
    0     0 DNAT       tcp  --  any    any     anywhere             0.rabbit-node.services1.cf-services-contrib.bosh  tcp dpt:61013 to:10.254.4.170:61013
</pre>


There are other ways to check whether the process is running. For example, given that you have a container with handle `16io51f6jri` running `redis-server`, you can find the corresponding process tree:

<pre class="terminal">
$ ps auxf | less
... [ search for 16io51f6jri:] /16io51f6jri
root      5291  0.0  0.0  14564   496 ?        S    06:43   0:00 wshd: 16io51f6jri
10000     5351  0.0  0.0  17708  1220 ?        Ss   06:43   0:00  \_ /bin/bash
10000     5353  0.1  0.0  36220  1772 ?        Sl   06:43   0:12      \_ /var/vcap/packages/redis26/bin/redis-server /var/vcap/store/redis/instances/9407af2c-0628-484b-a5f8-5c8634421a35/redis.conf
...
</pre>

and

<pre class="terminal">
$ pstree -p 5291
wshd(5291)───bash(5351)───redis-server(5353)─┬─{redis-server}(5362)
                                             └─{redis-server}(5363)
</pre>

Its corresponding PID of `wshd` will be put into the corresponding Warden depot directory:

<pre class="terminal">
$ cat /var/vcap/store/containers/16io51f6jri/run/wshd.pid
5291
</pre>

### <a id="entering"></a>Entering into a Warden container

Use wsh to enter within the container and interactively run commands within the container (e.g. curl from same NIC, send signals ...)

<pre class="terminal">

# cd /var/vcap/data/rabbit/containers/18fglrqcs23
# ./bin/wsh --user vcap

vcap@18fglrqcs23:~$ ps aux --cols=2000
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root         1  0.0  0.0   1124   292 ?        S s  Feb20   0:00 wshd: 18fglrqcs23
vcap        35  0.0  0.0  17712  1180 ?        S s  Feb20   0:00 /bin/bash
vcap        36  0.8  1.0  68568 44316 ?        S l  Feb20  36:05  \_ /var/vcap/packages/erlang/lib/erlang/erts-6.2/bin/beam -W w -K true -A30 -P 1048576 -- -root /var/vcap/packages/erlang/lib/erlang -progname erl -- -home /home/vcap -- -pa /var/vcap/packages/rabbitmq-3.0/sbin/../ebin -noshell -noinput -s rabbit boot -sname f80a348e-e937-4e4d-96c9-d65ef35ed960@localhost -boot start_sasl -config /var/vcap/store/rabbit/instances/f80a348e-e937-4e4d-96c9-d65ef35ed960/config/rabbitmq -kernel inet_default_connect_options [{nodelay,true}] -rabbit tcp_listeners [{"auto",10001}] -sasl errlog_type error -sasl sasl_error_logger false -rabbit error_logger {file,"/var/vcap/sys/service-log/rabbit/f80a348e-e937-4e4d-96c9-d65ef35ed960/f80a348e-e937-4e4d-96c9-d65ef35ed960@localhost.log"} -rabbit sasl_error_logger {file,"/var/vcap/sys/service-log/rabbit/f80a348e-e937-4e4d-96c9-d65ef35ed960/f80a348e-e937-4e4d-96c9-d65ef35ed960@localhost-sasl.log"} -rabbit enabled_plugins_file "/var/vcap/store/rabbit/instances/f80a348e-e937-4e4d-96c9-d65ef35ed960/config/enabled_plugins" -rabbit plugins_dir "/var/vcap/packages/rabbitmq-3.0/sbin/../plugins" -rabbit plugins_expand_dir "/var/vcap/store/rabbit/instances/f80a348e-e937-4e4d-96c9-d65ef35ed960/plugins" -os_mon start_cpu_sup false -os_mon start_disksup false -os_mon start_memsup false -mnesia dir "/var/vcap/store/rabbit/instances/f80a348e-e937-4e4d-96c9-d65ef35ed960/mnesia" -smp disable
vcap       117  0.0  0.0  10792   356 ?        S s  Feb20   0:00      \_ inet_gethost 4
vcap       118  0.0  0.0  17112   800 ?        S    Feb20   0:00          \_ inet_gethost 4
vcap        55  0.0  0.0  10824   332 ?        S    Feb20   0:01 /var/vcap/packages/erlang/lib/erlang/erts-6.2/bin/epmd -daemon
vcap     14969  0.0  0.0  17892  1804 pts/0    S s  13:36   0:00 /bin/bash
vcap     14993  0.0  0.0  15036  1004 pts/0    R +  13:36   0:00  \_ ps auxf --cols=2000

</pre>

### <a id="service-data"></a>Service instance logs, data and localdb

The data of the service sits in `/var/vcap/store/SERVICE-NAME/instances/UUID`:

<pre class="terminal">
$ ls /var/vcap/store/redis/instances/
2b4e0275-7da1-4da7-88c1-7c907b6b1b8c  5e8bbae6-2c3d-429e-bec7-a5b695d2c4f4
</pre>

The instance data directory normally contains the service PID, data directory and the instance specific config file. The log file of the instance sits in `/var/vcap/sys/service-log/SERVICE-NAME/UUID`:

<pre class="terminal">
$ ls /var/vcap/sys/service-log/redis/
2b4e0275-7da1-4da7-88c1-7c907b6b1b8c  5e8bbae6-2c3d-429e-bec7-a5b695d2c4f4
</pre>

Besides the service log, this directory also contains the `warden_service_ctl.log` and `warden_service_ctl.err.log`. `warden_service_ctl` is a script that controls the start, stop and status of the service instance.

The local sqlite database also includes valuable information for debugging issues with a service instance. The local sqlite database sits in `/var/vcap/store/SERVICE-NAME/SERVICE-NODE`. You can use the pre-installed `sqlite3` binary to look into the sqlite database:

<pre class="terminal">
$ /var/vcap/packages/sqlite/bin/sqlite3 /var/vcap/store/redis/redis_node.db
SQLite version 3.7.5
Enter ".help" for instructions
Enter SQL statements terminated with a ";"

sqlite> .table
vcap_services_redis_node_provisioned_services

sqlite> PRAGMA table_info(vcap_services_redis_node_provisioned_services);
0|name|VARCHAR(50)|1||1
1|port|INTEGER|0||0
2|password|VARCHAR(50)|1||0
3|plan|INTEGER|1||0
4|pid|INTEGER|0||0
5|memory|INTEGER|0||0
6|container|VARCHAR(50)|0||0
7|ip|VARCHAR(50)|0||0
8|version|VARCHAR(50)|0||0

sqlite> select * from vcap_services_redis_node_provisioned_services;
5e8bbae6-2c3d-429e-bec7-a5b695d2c4f4|5000|0508e659-c6d0-4d6f-b64f-4ab1049984c7|1|0|20|16j9dohgvks|10.254.0.2|2.6
2b4e0275-7da1-4da7-88c1-7c907b6b1b8c|5001|cd189a0e-144c-44fa-8e36-281c38e52e5c|1|0|20|16j9dohgvkt|10.254.0.6|2.6
</pre>

Some useful operations are listed above to show the table name, schema and values with the table that is used by the service node. The start, stop, and check\_status operations on the instance are all done by the `warden_service_ctl` script. On the node, it sits in `/var/vcap/store/SERVICE-NAME_common/bin`:

<pre class="terminal">
$ ls /var/vcap/store/redis_common/bin/
utils.sh  warden_service_ctl
</pre>

Service node code will use the `warden_service_ctl` script to start, stop, and ping the status of the service instance. You can also use the script manually with Warden client to start and stop the instance but it is dangerous to do that manually.

## <a id="services"></a>Service specific issue

### <a id="services-mongo"></a>MongoDB proxy

MongoDB has a corresponding proxy and they won’t be able to work without it. Mongodb will have their proxy running within Warden:

<pre class="terminal">
$ cat /var/vcap/store/containers/16iod87tbpe/run/wshd.pid
2398
$ pstree -p 2398
wshd(2398)───bash(2457)───mongod(2459)─┬─proxyctl(2470)─┬─{proxyctl}(2472)
                                       │                ├─{proxyctl}(2473)
                                       │                ├─{proxyctl}(2474)
                                       │                └─{proxyctl}(2475)
                                       ├─{mongod}(2497)
                                       ├─{mongod}(2499)
                                       ├─{mongod}(2500)
                                       ├─{mongod}(2501)
                                       ├─{mongod}(2502)
                                       ├─{mongod}(2503)
                                       ├─{mongod}(2504)
                                       └─{mongod}(2505)
</pre>

### <a id="services-rabbit"></a>RabbitMQ daylimit daemon

RabbitMQ has a daylimit daemon to monitor the bandwidth outside Warden container (monitored by monit) and it will take care of the bandwidth of each RabbitMQ instance. A RabbitMQ instance will be able to work without the daylimit daemon, while the daylimit daemon will block an instance when the throughput reaches its day limit.

Note that RabbitMQ from cf-services-contrib-release is community supported and has some known short comings, search vcap-dev@ for more details and how to improve it.



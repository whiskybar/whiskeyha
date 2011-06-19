Overview
========

There are six machines in the test. Their names all end with .barton. They 
resemble the production structure as close as I understand it.

In addition, any machine can become unavailable yet no information is lost.
Most traffic waits when a situation is detected and only a very tiny fraction
of it may get lost.


redis1
------

This machine runs two redis servers. I understand there are like ten
redis servers on your redis machine. This is just to emulate this. They
are running on the ports 9001 and 9002.

redis2
------

This machine is a copy of redis1.barton. We can say it is a slave of 
redis1.barton but this may no longer be true. This is how it started;
once redis1 failed, the roles were switched. One cannot be sure what is
slave of what because the switchover have happened many times. 

By slave I mean all redis servers on one machine are slaves to their 
respective redis servers on the other machine.

haproxy1
--------

This machine has a TCP proxy which relays its traffic to the master
redis server and, to the slave redis server if the master is unavailable.
It knows which redis is the master at the moment. 

It listens on 8001 which gets redirected to 9001 and so with 8002 and 9002.

By nature of haproxy itself, it can detect when the master (or slave) goes 
down. We can set this detection to whatever time we choose. Now it gives up
after three unsuccessful attempts each in 5 seconds. I am not sure but I think
this is the time when a request may get lost; that is only this one while
the rest of the requests wait.

It is configured in such a way the failed server never gets used again. In 
the meantime a monitoring software finds out and takes action.

haproxy2
--------

This server is an exact copy of haproxy1. It is a backup server for haproxy1.

app
---

This server represents many of the clients which use redis at Whiskymedia. It
runs a local haproxy which connects to haproxy1 and haproxy2 -- the latter 
one is configured as a backup for haproxy1.

The local proxy listens on 7001 and 7002 to make situation clear. There is a 
very simple bash script which can sort of test how smoothly a situation is
overcome. Run and keep it running like this::

	redis-test 7001


monit
-----

This server controls everything. It monitors redis1, redis2, haproxy1, and 
haproxy2, and is able to come up with a new machine which connects to the 
existing structure if that goes down.


Connections
===========

The easiest path looks like this::

    app:7001 --> haproxy1:8001 --> redis1:9001


The whole path with backup nodes looks like this::

                                           |-> redis1:9001
                       |-> haproxy1:8001 --
                       |                   |-> redis2:9001
                       |
    haproxy@app:7001 --
                       |
                       |                   |-> redis1:9001
                       |-> haproxy2:8001 --
                                           |-> redis2:9001


Failover
========

Failure detection
-----------------

monit checks haproxies and redises on regular basis. It runs a failover script
``/usr/local/sbin/failover`` if there is an outage three times in a row.

monit polls the redis servers directly; in a real world scenario, it would be
useful if it checked haproxies for redis failures. HAProxy performs health
checks on its own for its own sake. One could connect to haproxy and find out
from its status. I have used the simplest tool available (i.e. monit) which
was not able to read the response easily.

It seems natural to replace monit with Jeff's collectd or a different
monitoring tool when it's ready. I believe I have used monit to its full
potential; monit is rather a simple tool.

Recovery
--------

I am taking a very simple approach to fixing failures:

    * "unmonitor" the resource
    * unregister the resource from whatever service
    * **take the machine down**
    * **spin up a new machine from a stored AMI**
    * configure the new machine
    * connect the machine to the existing infrastructure
    * start monitoring the new machine with its resources

All logic is in the bash script on the monit server. One would replace with
commands in Diplomatico; nearly half of the script deals with EC2 
peculiarities.

The script is pretty self-explanatory but let's go into the real scenarios.

Recovery from redisX failure
----------------------------

This is ``failed_redis`` in the bash script.

1. Unmonitor the whole machine with the failed redis.
2. Unregister the redis from both haproxy1 and haproxy2.
3. Make the remaining redis master.
4. Kill the machine with the failed redis.
5. Launch a new machine from the AMI with the redis copy.
6. Make the new redis slave of the existing redis.
7. Register the new redis with both haproxies.
8. Start monitoring the new redis.

The algorithm works for the redis master and the redis slave.

Recovery from haproxyX failure
------------------------------

This is ``failed_haproxy`` in the bash script.

1. Unmonitor the failed machine.
2. Kill it.
3. Launch a new machine from the AMI with unconfigured haproxy.
4. Configure the haproxy the same way as the running haproxy.
5. Enter the new machine's IP into DNS.
6. Start monitoring the new haproxy.

The step #5 has to be carried out manually. This means adding the new IP
address to the haproxy on the app machine by hand and reloading the haproxy.

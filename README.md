Practice
========

NOTE: This is targetted to run on a dedicated docker host with more than 10 of
RAM.

Command reference
-----------------

    tools/cassie-client            Opens a shell used for running client tools like cassandra-stress
    tools/cassie-nodetool $NODE    Runs nodetool on $NODE
    tools/cassie-shell $NODE       Opens a shell on $NODE
    tools/cassie-start $NODE       Starts $NODE
    tools/cassie-stop $NODE        Stops $NODE
    tools/prep                     Prepares cluster environment

Starting up
-----------
To get started, run

    tools/prep

This will do several things:

1. Setup a common directory to share files between the 3 cassandra instances.
1. Setup working directories for each of 3 cassandra instances.
1. Create the 172.18.27.0/24 network for cassandra to run on. .11,.12,.13 are
for cassandra instances.
1. Pull down the cassandra 3.7 image to work with.

Once that's complete you can start up your cluster by starting up each instance.

    tools/cassie-start 01
    tools/cassie-start 02
    tools/cassie-start 03

In general each node will be represented in the wrapper tools with 01, 02, or
03.

To stop a node:

    tools/cassie-stop $NODENUMBER

After that, you will want to prepopulate data:

    tools/cassie-client
    cassandra-stress user profile=/common/stress.yaml ops\(insert=1\) no-warmup cl=ONE -node 172.18.27.11,172.18.27.12,172.18.27.13

Two things to note:
* This will also tell you that the environment is up and running
* Look at the performance numbers - rows/s

nodetool
--------
The cli tool for working with a cassandra node is nodetool. It is a dispatch
tool for a series of operations. E.g.:

    nodetool compactionstats              Print statistics on compactions
    nodetool describecluster              Print the name, snitch, partitioner and schema version of a cluster
    nodetool flush                        Flush one or more tables
    nodetool listsnapshots                Lists all the snapshots along with the size on disk and true size.
    nodetool rebuild                      Rebuild data by streaming from other nodes (similarly to bootstrap)
    nodetool repair                       Repair one or more tables
    nodetool status                       Print cluster information (state, load, IDs, ...)
    nodetool tablestats                   Print statistics on tables

This practice provides a convience wrapper around nodetool to easiy reference
the 3 nodes 01, 02, and 03. There is a convience wrapper to nodetool, so the
simpliest way to get status is:

    tools/cassie-nodetool $NODENUMBER status

Failure and Replication
-----------------------
To get a feel for how replication and consistency levels work, let's step
through a simple node fault scenario under different setups.

We're going to drive work to this cluster using the cassandra-stress tool.

The initial setup has a replication factor of 1, so there isn't much to
test consistency level.

Start a simple load:

    tools/cassie-client
    cassandra-stress user profile=/common/stress.yaml ops\(read1=1\) no-warmup cl=ONE -node 172.18.27.11

While that's running, open another window, and kill the 03 node:

    tools/cassie-stop 03

Cancel (CTRL-C cassandra-stress if it hasn't died already) that, and bring 03
back online:

    tools/cassie-start 03

Let's increase the replication factor. Back in cassie-client:

    cqlsh 172.18.27.11
    ALTER KEYSPACE stress WITH replication = {'class': 'SimpleStrategy', 'replication_factor': 3};

Now, the keyspace is configured for triple replication, but the data doesn't
move on it's own. You need to run repairs to force the copy to sync:

    tools/cassie-nodetool 01 repair -pr
    tools/cassie-nodetool 02 repair -pr
    tools/cassie-nodetool 03 repair -pr

Now, consistency level comes into play. Let's try with cl=ALL:

    cassandra-stress user profile=/common/stress.yaml ops\(read1=1\) no-warmup cl=ALL -node 172.18.27.11

and kill 03 again:

    tools/cassie-stop 03

As expected, with needing to query all of them, this fails. Let's try to relax
our consistency requirements with cl=QUORUM:

    cassandra-stress user profile=/common/stress.yaml ops\(read1=1\) no-warmup cl=ONE

Success!

*What are some of the impacts here?*

More resilient - yay!

But have to start watching performance.

How does throughput/latency compare between cl=ONE and cl=QUORUM?

    cassandra-stress user profile=/common/stress.yaml ops\(read1=1\) no-warmup cl=ONE -node 172.18.27.11
    
    cassandra-stress user profile=/common/stress.yaml ops\(read1=1\) no-warmup cl=QUORUM -node 172.18.27.11

Under cl=QUORUM, stop and restart a node, what happens with throughput/latency?
Open up top and see what's happening with the cassandra processes.

Compactions
-----------
Compactions happen on Cassandra's time, not on yours unless you want to manage
them yourself. However, you can force the memtable to drop to disk, which will
eventually cause compaction.

First, observe

    watch -n 1 "ls -l /home/cassie01/data/stress/stress-*"

And separately start flushing the memtables:

    tools/cassie-nodetool 01 flush
    sleep 2
    tools/cassie-nodetool 01 flush
    sleep 2
    tools/cassie-nodetool 01 flush
    sleep 2
    tools/cassie-nodetool 01 flush
    sleep 2
    tools/cassie-nodetool 01 flush

Keep repeating until you see compression in action.

This can also be observed with compactionstats:

    tools/cassie-nodetool 01 compactionstats

Backups
-------
With sstables being immutable, backups can be magical. Cassandra relies on
the sstables never changing and so points live snapshots to the existing data
files.

First, take a snapshot:

    tools/cassie-nodetool 01 snapshot

And list out snapshots:

    tools/cassie-nodetool 01 listsnapshots

And take a look at the file structure now:

    ls /home/cassie01/data/stress/stress-*/backup/*

stat one of those files - notice the "Links" field. Cassandra uses hard links
to back up the files. You can restore those if you want or 

Node loss
---------
There are many ways for a node to fail. You can have a total system failure, or
just a disk failure, or other parts. You can recover the node in a variety of
ways, but being able to replace a node is key.

Fake a node failure:

    tools/cassie-stop 03
    sudo rm -rf /home/cassie03/*

And try to add it back in:

    tools/casssie-start 03
    ...
    docker ps

Notice that 03 has stopped. Check the log for what has happened:

    tail /home/cassie03/logs/system.log

    java.lang.RuntimeException: A node with address /172.18.27.11 already exists, cancelling join. Use cassandra.replace_address if you want to replace this node.

The cluster already thinks there an 01 node, so we need to tell the cluster to
get rid of that.

    tools/cassie-nodetool 02 assassinate 172.18.27.13

And now clean up and start up 03 node:

    tools/cassie-stop 03  # clean up the container
    sudo rm -rf /home/cassie03/*
    tools/cassie-start 03

This one will stay up, so check the ring:

    tools/cassie-nodetool 03 status

It shows up, but the load is way off. We brought the node back, but it
currently is empty. We can perform a repair, but that'll be a bit suboptimal.
Cassandra has a rebuild mechanism for fully pulling the data back:

    tools/cassie-nodetool 03 rebuild
    tools/cassie-nodetool 03 status

Lots of tombstones
------------------
Coming soon


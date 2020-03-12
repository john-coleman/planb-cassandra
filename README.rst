================
Plan B Cassandra
================

Bootstrap and update a Cassandra cluster on STUPS_/AWS.

Planb deploys Cassandra by means of individual EC2 instances running Taupage_ & Docker with the latest
Cassandra version 3.0.x (default; the new 'tick-tock' releases 3.x and older 2.x versions
are still available).

Features:

* internal to a VPC or span multiple AWS regions
* fully-automated setup including Elastic IPs (when needed), EC2 security groups, SSL certs
* multi-region replication available (using Ec2MultiRegionSnitch_)
* encrypted inter-node communication (SSL/TLS)
* `EC2 Auto Recovery`_ enabled
* Jolokia_ agent to expose JMX metrics via HTTP

Non-Features:

* dynamic cluster sizing


Prerequisites
==============

* Python 3.5+
* Python dependencies (``sudo pip3 install -r requirements.txt``)
* Java 8 with ``keytool`` in your ``PATH`` (required to generate SSL certificates)
* Latest Stups tooling installed and configured
* You have created a dedicated AWS IAM user for auto-recovery.  The policy
  document for this user should look like the following::

    {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Effect": "Allow",
                "Action": [
                    "ec2:DescribeInstanceRecoveryAttribute",
                    "ec2:RecoverInstances",
                    "ec2:DescribeInstanceStatus",
                    "ec2:DescribeInstances",
                    "cloudwatch:PutMetricAlarm"
                ],
                "Resource": [
                    "*"
                ]
            }
        ]
    }
* You have a ``planb_autorecovery`` section in your AWS credentials file
  (``~/.aws/credentials``) with the access key of the auto-recovery user::

    [planb_autorecovery]
    aws_access_key_id = THEKEYID
    aws_secret_access_key = THESECRETKEY

  These credentials are only used to create the auto-recovery alarm.  When
  triggered by the failing system status check, the recovery action is
  performed by this dedicated user.

  .. note::

     The access keys for the auto-recovery user can be rotated or made
     inactive at any time, without impacting its ability to perform the
     recovery action.  The user still needs to be there, however.


Usage
=====

Create a new cluster
--------------------

To create a cluster named "mycluster" in two regions with 3 nodes per region
(the default size, enough for testing):

.. code-block:: bash

    $ zaws login  # get temporary AWS credentials
    $ ./planb.py create --cluster-name mycluster --use-dmz eu-west-1 eu-central-1

The above example requires Elastic IPs to be allocated in every region (this
might require to increase the AWS limits for Elastic IPs).

To create a cluster in a single region, using private IPs only, see
the following example:

.. code-block:: bash

    $ ./planb.py create --cluster-name mycluster eu-central-1

It is possible to use Public IPs even with a single region, for
example, if your application(s) connect from different VPC(s).  This
is currently **not recommended**, though, as there is no provision for
client-to-server encryption.

Available options are:

===========================  ============================================================================
--cluster-name               Not actually an option, you must specify the name of a cluster to create
--cluster-size               Number of nodes to create per AWS region.  Default: 3
--dc-suffix                  Optional "DC suffix".
--num-tokens                 Number of virtual nodes per node.  Default: 256
--instance-type              AWS EC2 instance type to use for the nodes.  Default: t2.medium
--volume-type                Type of EBS data volume to create for every node.  Default: gp2 (General Purpose SSD).
--volume-size                Size of EBS data volume in GB for every node.  Default: 16
--volume-iops                Number of provisioned IOPS for the volumes, used only for volume type of io1.  Default: 100 (when applicable).
--no-termination-protection  Don't protect EC2 instances from accidental termination.  Useful for testing and development.
--use-dmz                    Deploy the cluster into DMZ subnets using Public IPs (required for multi-region setup).
--hosted-zone                Specify this to create SRV records for every region, listing all nodes' private IP addresses in that region.  This is optional.
--scalyr-key                 API Key for writing logs to Scalyr (optional).
--scalyr-region              Scalyr account region, such as 'eu' (optional).
--artifact-name              Override Pierone artifact name.  Default: planb-cassandra-3.0
--docker-image               Override default Docker image.
--environment, -e            Extend/override environment section of Taupage user data.
--sns-topic                  Amazon SNS topic name to use for notifications about Auto-Recovery.
--sns-email                  Email address to subscribe to Amazon SNS notification topic.  See below for details.
===========================  ============================================================================

In order to be able to receive notification emails in case instance
recovery is triggered, provide either SNS topic name in
``--sns-topic``, or email to subscribe in ``--sns-email`` (or both).

If only the email address is specified, then SNS topic name defaults
to ``planb-cassandra-system-event``.  An SNS topic will be created (if
it doesn't exist) in each of the specified regions.  If email is
specified, then it will be subscribed to the topic.

If you use the Hosted Zone parameter, a full name specification is
required e.g.: ``--hosted-zone myzone.example.com.`` (note the
trailing dot.)

After the create command finishes successfully, follow the on-screen
instructions to create the admin superuser, set replication factors for
system_auth keyspace and then create your application user and the data
keyspace.

The generated administrator password is available inside the docker
container in an environment variable ``ADMIN_PASSWORD``.

The list of private IP contact points for the application can be
obtained with the following snippet:

.. code-block:: bash

    $ aws ec2 describe-instances --region $REGION --filter 'Name=tag:Name,Values=planb-cassandra' | grep PrivateIp | sed s/[^0-9.]//g | sort -u

Update of a cluster
-------------------

.. important::

   The Jolokia port 8778 should be accessible from the Odd host. Ensure the
   ingress rule for your clusters security group allows connections from the Odd
   host.

To update the Docker image or AMI you should ensure that you are logged in to
your account and have SSH access to your Odd host. The following commands will
allow you to update the Docker image on all nodes of the cluster `mycluster`.
If an action is interrupted the next call will resume with the last action on
the last used node.

.. code-block:: bash

    $ zaws re $ACCOUNT  # for longer updates run `zaws login -r` in background
    $ piu re -O $ODDHOST $ODDHOST  # for longer updates add `-t 180` or bigger
    $ ./planb.py update \
        --region eu-central-1 \
        --odd-host $ODDHOST \
        --cluster-name mycluster \
        --docker-image registry.opensource.zalan.do/stups/planb-cassandra-3.0:cd-69 \
        --sns-topic planb-cassandra-system-event \
        --sns-email test@example.com

Available options for update:

===================  ========================================================
--region             The region where the update should be applied (required)
--odd-host           The Odd host in the region of your VPC (required)
--cluster-name       The name of your cluster (required)
--filters            Additional AWS resource filters (in JSON format)
--force-termination  Disable termination protection for the duration of update
--no-prompt          Don't prompt before updating every node.
--docker-image       The full specified name of the Docker image
--taupage-ami-id     The full specified name of the AMI
--instance-type      The type of instance to deploy each node on (e.g. t2.medium)
--scalyr-key         API Key for writing logs to Scalyr (optional).
--scalyr-region      Scalyr account region, such as 'eu' (optional).
--environment, -e    Extend/override environment section of Taupage user data.
--sns-topic          Amazon SNS topic name to use for notifications about Auto-Recovery.
--sns-email          Email address to subscribe to Amazon SNS notification topic.  See description of ``create`` subcommand above for details.
===================  ========================================================

The cluster name parameter is used to list all EC2 instances in the region
with the matching ``Name`` tag.  This parameter may contain wildcards (``*``).
For example, if you have multiple virtual data centers in a cluster, this
allows to update all nodes of all DCs by running only one command.

Any additional resource filters supported by AWS may be provided (only JSON
format is supported, though).  For example, to limit the update operation to a
specific Availability Zone, add the following parameter: ``--filters
'[{"Name":"availability-zone","Values":["eu-central-1c"]}]'``.

By default, ``update`` is an interactive command which operates on one node at a time.
It will prompt before starting update of each node.  It starts by draining the
target node and then terminates the EC2 instance that is running it.  Then a new
EC2 instance is created with the same private and public IP addresses (if any),
and potentially different configuration as specified by the options.  The new
instance is expected to attach the EBS volume that was previously utilized by the
node.  This keeps all the node's data and identification within the cluster intact.

The command will wait for the replacement node to be back UP.  You should still
monitor the status of the cluster to verify that all other nodes also see the new
node as UP before proceeding.

If you're confident enough in using this command, you may opt in for "fire and
forget" behavior, by specifying the ``--no-prompt`` flag.

While performing the update, which destroys the running EC2 instance and creates a
blank one, the command keeps the current state in the tags of the EBS data volume.

If interrupted by some unexpected problems, the command resumes the update sequence
by using the information in the EBS volume tags.  This relies however on an assumption
that the command is ran again with essentially the same parameters on the same machine,
since some of the state is stored in a temporary file, named after the EBS volume id.

If the command enters `failed` state, as a safety precaution it will not try to proceed
further, even if started again.  The operator is then responsible for analysing the
failure reason and removing the failed state tag from the related EBS volume before
starting the command again.  One common source of failed state is forgetting to use
`--force-termination` flag on a cluster which was deployed with termination protection
enabled.

No provisions are made by the command to detect if a concurrent update operation is
in progress for a given cluster.  It makes sense to ensure that only one operator is
using the command as part of routine maintenance at any given time.

Extend an existing cluster
--------------------------

There are a number of scenarios requiring to extend an existing cluster.  The
possible use-cases are::

* Add a new "virtual data center"
* Add a new region
* Add more nodes to existing data center

Available options for extend:

==============================  ============================================================================
--from-region                   Name of AWS region where a cluster is already running.
--to-region                     Name of AWS region where a new data center should be created.  This can be the same as "from region", in this case a virtual data center is created.
--cluster-name                  The name of a cluster to extend.
--ring-size                     Number of nodes to create in the new data center.
--dc-suffix                     Optional "DC suffix".  When creating a virtual data center be sure to specify a new suffix for each virtual data center you create!
--num-tokens                    Number of virtual nodes per node.  Default: 256
--allocate-tokens-for-keyspace  Use new token allocation algorithm, available starting with version 3.0.
--instance-type                 AWS EC2 instance type to use for the nodes.  Default: t2.medium
--volume-type                   Type of EBS data volume to create for every node.  Default: gp2 (General Purpose SSD).
--volume-size                   Size of EBS data volume in GB for every node.  Default: 16
--volume-iops                   Number of provisioned IOPS for the volumes, used only for volume type of io1.  Default: 100 (when applicable).
--no-termination-protection     Don't protect EC2 instances from accidental termination.  Useful for testing and development.
--use-dmz                       Deploy the new data center into DMZ subnets using Public IPs (required for multi-region setup).
--hosted-zone                   Specify this to create the SRV record for the new data center.  This is optional.
--artifact-name                 Override Pierone artifact name.  Default: planb-cassandra-3.0
--docker-image                  Override default Docker image.
--environment, -e               Extend/override environment section of Taupage user data.
--sns-topic                     Amazon SNS topic name to use for notifications about Auto-Recovery.
--sns-email                     Email address to subscribe to Amazon SNS notification topic.  See description of ``create`` subcommand above for details.
==============================  ============================================================================

-------------------------------
Add a new "virtual data center"
-------------------------------

To add a new virtual data center in the same region where your existing
cluster is running run the extend command like this:

.. code-block:: bash

    $ planb.py extend \
        --from-region eu-central-1 \
        --to-region eu-central-1 \
        --cluster-name mycluster \
        --ring-size 3 \
        --dc-suffix _new \
        --hosted-zone myzone.example.com.

.. important::

   In this mode the new nodes are created with ``auto_bootstrap: false``.
   When creating a new virtual data center in the same region, you **must**
   specify the DC suffix which doesn't exist in the region yet!  Otherwise you
   risk adding a number of empty nodes to the cluster, which will be serving
   read requests and your client applications will suffer from apparent data
   loss.

After the command has run successfully, you need to login to each of the nodes
in the new data center and run ``nodetool rebuild $existing_dc_name``.

On version 3.0 or later it is possible to request use of the new token
allocation algorithm.  For that, start by including the to-be-deployed virtual
DC in the replication settings of the data keyspace, by running a CQL
statement like the following one on one of the existing cluster nodes:

.. code-block::

   cqlsh> ALTER KEYSPACE mydata WITH replication = {
       'class': 'NetworkTopologyStrategy',
       'eu-central': 3,
       'eu-central_new': 3
   };

Then run the extend command, specifying the
``--allocate-tokens-for-keyspace=mydata`` as one of the options.

With the new token allocation algorithm it makes sense to use a much smaller
number of tokens than the default 256.  E.g. 16 tokens are generally enough to
achieve balanced ownership distribution.  Use the ``--num-tokens`` option to
set the desired number of tokens per node.

.. important::

   In order for the token allocation algorithm to be actually used, the
   ``auto_bootstrap`` parameter has to be set to ``true``.  This is done
   automatically by the deployment script.  Due to this, before you can run
   ``nodetool rebuild`` command on the nodes of the newly deployed ring, you
   have to run manually the following CQL command on every new node:
   ``TRUNCATE system.available_ranges``.

----------------
Add a new region
----------------

To extend a cluster to a new AWS region, run the command like this:

.. code-block:: bash

    $ planb.py extend \
        --from-region eu-central-1 \
        --to-region eu-west-1 \
        --cluster-name mycluster \
        --ring-size 3 \
        --use-dmz \
        --hosted-zone myzone.example.com.

The DC suffix is optional in this case, unless you already have a cluster with
this name in the target region.  You must specify the DMZ option, and the
existing cluster must already be running in the DMZ: otherwise the new and
existing nodes will not be able to communicate with each other.

--------------------------------------
Add more nodes to existing data center
--------------------------------------

This is currently unsupported, due to the use of `auto_bootstrap: false` when
creating new nodes.  In general, it should be possible to override this option
and add the nodes one by one to the existing data center, but care should be
taken while doing so.

Running commands remotely on Cassandra nodes
============================================

There is a command group called ``remote`` that allows you to run arbitrary
shell commands on all nodes of a given Cassandra cluster.  This can be useful
when applying a configuration change, e.g. setting compaction throughput:

.. code-block:: bash

    $ planb.py remote \
        --region eu-west-1 \
        --odd-host $ODDHOST \
        --cluster-name mycluster \
        --piu "setting cassandra compaction throughput" \
        nodetool \
        -- \
        setcompactionthroughput 50

The following options are available for the ``remote`` command:

==============  ==================================================
--region        AWS region.
--odd-host      Odd host name for the first SSH hop.
--cluster-name  The name of the cluster (Name tag on the EC2 instances).
--filters       Additional AWS resource filters (in JSON format)
--piu           Run ``piu`` first with this parameter as reason.
--echo          Print the command before running it.
--no-prompt     Don't prompt before running the command.
--no-wait       Don't wait for the command to exit.
--ip-label      Label all output from the node with its IP address.
--help          Show this message and exit.
==============  ==================================================

There are 3 subcommands in the ``remote`` command group:

========  ==============================
shell     Run an arbitrary shell command.
nodetool  Run a nodetool command.
cqlsh     Run an administrative CQL shell command.
========  ==============================

The most basic is ``shell`` which allows to run any command on the server.
Two shorthand commands for running ``nodetool`` and ``cqlsh -u admin -p
$ADMIN_PASSWORD`` are also provided.

Client configuration for Public IPs setup
=========================================

When configuring your client application to talk to a Cassandra
cluster deployed in AWS using Public IPs, be sure to enable address
translation using EC2MultiRegionAddressTranslator_.  Not only it saves
costs when communicating within single AWS region, it also prevents
availability problems when security group for your Cassandra is not
configured to allow client access on Public IPs (via the region's NAT
instances addresses).

Even if your client connects to the ring using Private IPs, the list
of peers it gets from the first Cassandra node to be contacted only
consists of Public IPs in such setup.  Should that node go down at a
later time, the client has no chance of reconnecting to a different
node if the client traffic on Public IPs is not allowed.  For the same
reason the client won't be able to distribute load efficiently, as it
will have to choose the same coordinator node for every request it
sends (namely, the one it has first contacted via the Private IP).


Troubleshooting
===============

To watch the cluster's node status (e.g. joining during initial bootstrap):

.. code-block:: bash

    $ # on Taupage instance
    $ watch docker exec -it taupageapp nodetool status

The output should look something like this (freshly bootstrapped cluster):

::

    Datacenter: eu-central
    ======================
    Status=Up/Down
    |/ State=Normal/Leaving/Joining/Moving
    --  Address        Load       Tokens  Owns (effective)  Host ID                               Rack
    UN  52.29.137.93   66.59 KB   256     34.8%             62f50c2c-cb0f-4f62-a518-aa7b1fd04377  1a
    UN  52.28.11.187   66.43 KB   256     31.1%             69d698a9-7357-46b2-93b8-6c038155f0c1  1b
    UN  52.29.41.128   71.79 KB   256     35.0%             b76e7ed7-78de-4bbc-9742-13adbbcfd438  1a
    Datacenter: eu-west
    ===================
    Status=Up/Down
    |/ State=Normal/Leaving/Joining/Moving
    --  Address        Load       Tokens  Owns (effective)  Host ID                               Rack
    UN  52.49.209.129  91.29 KB   256     34.8%             140bc7de-9973-46fd-af8c-68148bf20524  1b
    UN  52.49.192.149  81.16 KB   256     32.1%             cb45fc4c-291d-4b2b-b50f-3a11048f0211  1c
    UN  52.49.128.58   81.22 KB   256     32.1%             8a270de3-b419-4baf-8449-f4bc65c51d0d  1a


Scaling up instance
===================

The following manual process may be applied whenever there is a need
to scale up EC2 instances or update Taupage AMI.

For every node in the cluster, one by one:

#. Stop a node (``nodetool drain; nodetool stopdaemon``).
#. Terminate EC2 instance, **take note of its IP address(es)**.  Simply stopping will not work as the private IP will be still occupied by the stopped instance.
#. Use the 'Launch More Like This' menu in AWS web console on one of the remaining nodes.
#. **Use the latest available Taupage AMI version.  Older versions are subject to data loss race conditions when attaching EBS volumes.**
#. Be sure to reuse the private IP of the node you just terminated on the new node.
#. In the 'Instance Details' section, edit 'User Data' to add ``erase_on_boot: false`` flag under ``mounts: /var/lib/cassandra``.  See documentation of Taupage_ for detailed description and syntax example.  The docker image version being used can also be updated in this section, however, it is recommended to avoid changing multiple things at a time.  Also, docker image can be updated without terminating the instance, by stopping and starting it with updated 'User Data' instead.
#. While the new instance is spinning up, attach the (now detached) data volume to the new instance.  Use ``/dev/sdf`` as the device name.
#. Log in to node, check application logs, if it didn't start up correctly: ``docker restart taupageapp``.
#. Repair the node with ``nodetool repair`` (optional: if the node was down for less than ``max_hint_window_in_ms``, which is by default 3 hours, hinted hand off should take care of streaming the changes from alive nodes).
#. Check status with ``nodetool status``.

Proceed with other nodes as long as the current one is back and
everything looks OK from nodetool and application points of view.


Scaling out cluster
===================

It is possible to manually scale out already deployed cluster by
following these steps:

#. Increase replication factor of ``system_auth`` keyspace (if needed)
   in every region affected.  Don't set RFs to be more than 5 per region
   or virtual DC.

   For example, if you run in two regions and want to scale to 5 nodes
   per region, issue the following CQL command on any of the nodes:

   ``ALTER KEYSPACE system_auth WITH replication = {'class': 'NetworkTopologyStrategy', 'eu-central': 5, 'eu-west': 5};``

#. *For public IPs setup only:* pre-allocate Elastic IPs for the new
   nodes in every region, then update security groups in every region
   to include all newly allocated Elastic IP addresses.

   For example, if scaling from 3 to 5 nodes in two regions you will
   need 2 new IP addresses in every region and both security groups
   need to be updated to include a total of 4 new addresses.

#. Choose a private IP for the new instance, that is not already taken by any
   other EC2 instance in the VPC.  You will need it on further steps.

#. Create a new EBS volume of appropriate type and size (normally you want to
   have the same settings as for the rest of the cluster).  EBS encryption is
   not recommended as it might prevent auto-recovery.

#. Create a ``Name`` tag on the volume in the format:
   ``<cluster-name>-<private-ip>``.

#. Create an additional tag on the newly created **empty EBS volume:**
   the tag name should be ``Taupage:erase-on-boot`` and the value ``True``.

#. Use the 'Launch More Like This' menu in the AWS web console on one
   of the running nodes.

#. Choose appropriate subnet for the new node: ``internal-...``
   vs. ``dmz-...`` for public IPs setup.  The subnet need to match your
   private IP, which should also be assigned manually on the same page.

#. Make sure that under 'Instance Details' the setting 'Auto-assign
   Public IP' is set to 'Disable'.

#. **Review UserData.** Make sure that ``AUTO_BOOTSTRAP`` environment variable
   is set to ``true`` or not present.  Update the referenced EBS volume to:
   ``<cluster-name>-<private-ip>``

#. Launch the instance.

#. *For public IPs setup:* while the instance is starting up,
   associate one of the pre-allocated Elastic IP addresses with it.

   **Caution!** For multi-region setup the nodes are started in DMZ
   subnet and thus don't have internet traffic before you give them a
   public IP.  Be sure to do this before anything else, or the new
   node won't be able to ship its logs and you won't be able to ssh
   into it (restarting the node should help if it was too late).

#. Monitor the logs of the new instance and ``nodetool status`` to
   track its progress in joining the ring.

#. Use the 'CloudWatch Monitoring' > 'Add/Edit Alarms' to add an
   auto-recovery alarm for the new instance.

   Check '[x] Take the action: [*] Recover this instance' and leave
   the rest of parameters at their default values.  It is also
   recommended to set up a notification SNS topic for actual recovery
   events.

Only when the new node has fully joined, proceed to add more nodes.
After all new nodes have joined, issue ``nodetool cleanup`` command on
every node in order to free up the space that is still occupied by the
data that the node is no longer responsible for.

.. _STUPS: https://stups.io/
.. _Odd: http://docs.stups.io/en/latest/components/odd.html
.. _Taupage: http://docs.stups.io/en/latest/components/taupage.html
.. _Ec2MultiRegionSnitch: http://docs.datastax.com/en/cassandra/2.1/cassandra/architecture/architectureSnitchEC2MultiRegion_c.html
.. _EC2MultiRegionAddressTranslator: https://datastax.github.io/java-driver/manual/address_resolution/#ec2-multi-region
.. _EC2 Auto Recovery: https://aws.amazon.com/blogs/aws/new-auto-recovery-for-amazon-ec2/
.. _Jolokia: https://jolokia.org/
.. _Più: http://docs.stups.io/en/latest/components/piu.html

Upgrade your cluster from Cassandra 2.1 -> 3.0.x
===================

In order to upgrade your Cluster you should run the following steps. You should have in mind that this process is a rolling update, which means applying the changes for each node in your cluster one by one.
After upgrading the last node in your cluster you are done.

**Disclaimer: Before you actually start, you should:**
  1. Read the [Datastax guide](https://docs.datastax.com/en/latest-upgrade/upgrade/cassandra/upgrdCassandraDetails.html) and consider the upgrade restrictions.
  2. Check if your client applications driver actually support V4 of the cql-protocol


1. Check for the latest Plan-B Cassandra image version: 
  `curl https://registry.opensource.zalan.do/teams/stups/artifacts/planb-cassandra-3.0/tags | jq '.[-1].name'`
2. Connect to the instance where you want to run the upgrade and enter your docker container. 
3. Run `nodetool upgradesstables` and `nodetool drain`. The latter command will flush the memtables and speed up the upgrade process later on. *This command is mandatory and cannot be skipped.*
   Excerpt from the manual `Cassandra stops listening for connections from the client and other nodes. You need to restart Cassandra after running nodetool drain.`
4. Remove the docker container by running on the host `docker rm -f taupageapp`
5. If you are running cassandra with the old folder structure where the data is directly located in __mounts/var/lib/cassandra/__ do the following. **If not go on with step 6.** 
  1. Move all keyspaces to __/mounts/var/lib/cassandra/data/data__
  2. Move the folder  commit_logs to __/mounts/var/lib/cassandra/data/commitlog__ 
  3. Move the folder saved_caches to __/mounts/var/lib/cassandra/data/__
  4. Set owner of data folders to application
    Example:
    ```
    **Before Move**

    /mounts/var/lib/cassandra$ ls
    commit_logs  keyspace_1 saved_caches  system_auth  system_traces 


    **After Move**

    /mounts/var/lib/cassandra$ ls -la
    total 28
    drwxrwxrwx 4 application application  4096 Oct 10 12:21 .
    drwxr-xr-x 3 root        root         4096 Aug 25 13:27 ..
    drwxrwxr-x 5 application mpickhan     4096 Oct 10 12:21 data

    /mounts/var/lib/cassandra$ ls -la data/
    total 36
    drwxrwxr-x 5 application mpickhan     4096 Oct 10 12:21 .
    drwxrwxrwx 4 application application  4096 Oct 10 12:21 ..
    drwxr-xr-x 2 application root        20480 Oct 10 12:15 commitlog
    drwxrwxr-x 9 application mpickhan     4096 Oct 10 12:19 data
    drwxr-xr-x 2 application root         4096 Oct 10 10:52 saved_caches

    /mounts/var/lib/cassandra$ ls -la data/data/
    total 36
    drwxrwxr-x  9 application mpickhan 4096 Oct 10 12:19 .
    drwxrwxr-x  5 application mpickhan 4096 Oct 10 12:21 ..
    drwxr-xr-x 10 application root     4096 Aug 25 14:29 keyspace_1
    drwxr-xr-x 19 application root     4096 Aug 25 13:27 system
    drwxr-xr-x  5 application root     4096 Aug 25 13:27 system_auth
    drwxr-xr-x  4 application root     4096 Aug 25 13:27 system_traces
    ```
6. **Stop** the ec2-Instance and change the user details `Go to Actions -> Instance Settings -> View/Change User Details` Change the "source" entry to the version you want to upgrade to:
    **Important:** Use the stop command and __not__ terminate.
    ```
    Example:

    From: "source: registry.opensource.zalan.do/stups/planb-cassandra:cd89"
    To: "source: registry.opensource.zalan.do/stups/planb-cassandra-3.0:cd105"
    ```
7. Start the instance and connect to it. At this point your node should be working and serving reads and writes. Login to the docker container and finish the upgrade by running `nodetool upgradesstables`.
   Check the logs for errors and warnings. (__Note:__ For the size of ~12GB SSTables it takes approximately one hour to convert them to the new format.)
8. Proceed with each node in your cluster.

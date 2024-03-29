# Running Couchbase Server 2.2 under CoreOS Fleet

This is a Docker image and set of CoreOS unit/cloud-config files which makes it easy to fire up a Couchbase Server 2.2 cluster that looks like:

![architecture diagram](http://tleyden-misc.s3.amazonaws.com/blog_images/couchbase-coreos-onion.png)

## Launch CoreOS instances via AWS Cloud Formation

Click the "Launch Stack" button to launch your CoreOS instances via AWS Cloud Formation:

[<img src="https://s3.amazonaws.com/cloudformation-examples/cloudformation-launch-stack.png">](https://console.aws.amazon.com/cloudformation/home?region=us-east-1#cstack=sn%7ECouchbase-CoreOS%7Cturl%7Ehttp://tleyden-misc.s3.amazonaws.com/couchbase-coreos/coreos-stable-pv.template)

*NOTE: this is hardcoded to use the us-east-1 region, so if you need a different region, you should edit the URL accordingly*

Use the following parameters in the form:

* **ClusterSize**: 3 nodes (default)
* **Discovery URL**:  as it says, you need to grab a new token from https://discovery.etcd.io/new and paste it in the box.
* **KeyPair**:  use whatever you normally use to start EC2 instances.  For this discussion, let's assumed you used `aws`, which corresponds to a file you have on your laptop called `aws.cer`

## ssh into a CoreOS instance

Go to the AWS console under EC2 instances and find the public ip of one of your newly launched CoreOS instances.  

Choose any one of them (it doesn't matter which), and ssh into it as the **core** user with the cert provided in the previous step:

```
$ ssh -i aws.cer -A core@ec2-54-83-80-161.compute-1.amazonaws.com
```

## Sanity check

```
$ fleetctl list-units
```

Should return an empty list, but no errors of any kind.

## Download cluster-init script

```
$ wget https://raw.githubusercontent.com/tleyden/couchbase-server-coreos/master/2.2/scripts/cluster-init.sh
$ chmod +x cluster-init.sh
```

## Launch cluster

```
$ ./cluster-init.sh -n 3 -u "user:passw0rd"
```

Where:

* **-n** the total number of couchbase nodes to start -- should correspond to number of ec2 instances (eg, 3)
* **-u** the username and password as a single string, delimited by a colon (:) 

Replace `user:passw0rd` with a sensible username and password.  It **must** be colon separated, with no spaces.  The password itself must be at least 6 characters.

Once this command completes, your cluster will be in the process of launching.

## Verify 

To check the status of your cluster, run:

```
$ fleetctl list-units
```

You should see four units, all as active.

```
UNIT						MACHINE				ACTIVE	SUB
couchbase_bootstrap_node.service                375d98b9.../10.63.168.35	active	running
couchbase_bootstrap_node_announce.service       375d98b9.../10.63.168.35	active	running
couchbase_node.1.service                        8cf54d4d.../10.187.61.136	active	running
couchbase_node.2.service                        b8cf0ed6.../10.179.161.76	active	running
```

## Rebalance Couchbase Cluster

**Login to Couchbase Server Web Admin**

* Find the public ip of any of your CoreOS instances via the AWS console
* In a browser, go to `http://<instance_public_ip>:8091`
* Login with the username/password you provided above

After logging in, your Server Nodes tab should look like this:

![screenshot](http://tleyden-misc.s3.amazonaws.com/blog_images/couchbase_admin_ui_prerebalance.png)

**Kick off initial rebalance**

* Click server nodes
* Click "Rebalance"

After the rebalance is complete, you should see:

![screenshot](http://tleyden-misc.s3.amazonaws.com/blog_images/couchbase_admin_ui_post_rebalance.png)

# References

* https://registry.hub.docker.com/u/ncolomer/couchbase/
* https://github.com/lifegadget/docker-couchbase
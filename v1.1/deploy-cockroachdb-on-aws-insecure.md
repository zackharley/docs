---
title: Deploy CockroachDB on AWS EC2 (Insecure)
summary: Learn how to deploy CockroachDB on Amazon's AWS EC2 platform.
toc: false
toc_not_nested: true
---

<div class="filters filters-big clearfix">
  <a href="deploy-cockroachdb-on-aws.html"><button class="filter-button">Secure</button>
  <button class="filter-button current"><strong>Insecure</strong></button></a>
</div>

This page shows you how to manually deploy an insecure multi-node CockroachDB cluster on Amazon's AWS EC2 platform, using AWS's managed load balancing service to distribute client traffic.

{{site.data.alerts.callout_danger}}If you plan to use CockroachDB in production, we strongly recommend using a secure cluster instead. Select <strong>Secure</strong> above for instructions.{{site.data.alerts.end}}

<div id="toc"></div>

## Requirements

You must have SSH access ([key pairs](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-key-pairs.html)/[SSH login](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/AccessingInstancesLinux.html)) to each machine with root or sudo privileges. This is necessary for distributing binaries and starting CockroachDB.

## Recommendations

- If you plan to use CockroachDB in production, we recommend using a [secure cluster](deploy-cockroachdb-on-aws.html) instead. Using an insecure cluster comes with risks:
  - Your cluster is open to any client that can access any node's IP addresses.
  - Any user, even `root`, can log in without providing a password.
  - Any user, connecting as `root`, can read or write any data in your cluster.
  - There is no network encryption or authentication, and thus no confidentiality.

- For guidance on cluster topology, clock synchronization, cache and SQL memory size, and file descriptor limits, see [Recommended Production Settings](recommended-production-settings.html).

- All instances running CockroachDB should be members of the same Security Group.

- Decide how you want to access your Admin UI:
	- Only from specific IP addresses, which requires you to set firewall rules to allow communication on port `8080` *(documented on this page)*
	- Using an SSH tunnel, which requires you to use `--http-host=localhost` when starting your nodes

## Step 1. Configure your network

CockroachDB requires TCP communication on two ports:

- `26257` for inter-node communication (i.e., working as a cluster), for applications to connect to the load balancer, and for routing from the load balancer to nodes
- `8080` for exposing your Admin UI

You can create these rules using [Security Groups' Inbound Rules](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-network-security.html#adding-security-group-rule).

#### Inter-node and load balancer-node communication

| Field | Recommended Value |
|-------|-------------------|
| Type | Custom TCP Rule |
| Protocol | TCP |
| Port Range | **26257** |
| Source | The name of your security group (e.g., *sg-07ab277a*) |

#### Admin UI

| Field | Recommended Value |
|-------|-------------------|
| Type | Custom TCP Rule |
| Protocol | TCP |
| Port Range | **8080** |
| Source | Your network's IP ranges |

#### Application data

| Field | Recommended Value |
|-------|-------------------|
| Type | Custom TCP Rules |
| Protocol | TCP |
| Port Range | **26257** |
| Source | Your application's IP ranges |

## Step 2. Create instances

[Create an instance](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/launching-instance.html) for each node you plan to have in your cluster. We [recommend](recommended-production-settings.html#cluster-topology):

- Running at least 3 CockroachDB nodes to ensure survivability.
- Selecting the same continent for all of your instances for best performance.

## Step 3. Set up load balancing

Each CockroachDB node is an equally suitable SQL gateway to your cluster, but to ensure client performance and reliability, it's important to use TCP load balancing:

- **Performance:** Load balancers spread client traffic across nodes. This prevents any one node from being overwhelmed by requests and improves overall cluster performance (queries per second).

- **Reliability:** Load balancers decouple client health from the health of a single CockroachDB node. In cases where a node fails, the load balancer redirects client traffic to available nodes.

AWS offers fully-managed load balancing to distribute traffic between instances.

1. [Add AWS load balancing](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-increase-availability.html). Be sure to:
	- Set forwarding rules to route TCP traffic from the load balancer's port **26257** to port **26257** on the node Droplets.
	- Configure health checks to use HTTP port **8080** and path `/health`.
2. Note the provisioned **IP Address** for the load balancer. You'll use this later to test load balancing and to connect your application to the cluster.

{{site.data.alerts.callout_info}}If you would prefer to use HAProxy instead of AWS's managed load balancing, see <a href="manual-deployment-insecure.html">Manual Deployment</a> for guidance.{{site.data.alerts.end}}

## Step 4. Start the first node

1. SSH to your instance:

	~~~ shell
	$ ssh -i <path to AWS .pem> <username>@<node1 external IP address>
	~~~

2. Install the latest CockroachDB binary:

	~~~ shell
	# Get the latest CockroachDB tarball.
	$ wget https://binaries.cockroachdb.com/cockroach-{{ page.release_info.version }}.linux-amd64.tgz

	# Extract the binary.
	$ tar -xf cockroach-{{ page.release_info.version }}.linux-amd64.tgz  \
	--strip=1 cockroach-{{ page.release_info.version }}.linux-amd64/cockroach

	# Move the binary.
	$ sudo mv cockroach /usr/local/bin
	~~~

3. Start a new CockroachDB cluster with a single node:

	~~~
	$ cockroach start --insecure \
    --advertise-host=<node1 internal IP address> \
    --cache=25% \
    --max-sql-memory=25% \
    --background
	~~~

	This commands starts an insecure node and identifies the address at which other nodes can reach it. It also increases the node's cache and temporary SQL memory size to 25% of available system memory to improve read performance and increase capacity for in-memory SQL processing (see [Recommended Production Settings](recommended-production-settings.html#cache-and-sql-memory-size-changed-in-v1-1) for more details).

	Otherwise, it uses all available defaults. For example, the node stores data in the `cockroach-data` directory, listens for internal and client communication on port 26257, and listens for HTTP requests from the Admin UI on port 8080. To set these options manually, see [Start a Node](start-a-node.html).

## Step 5. Add nodes to the cluster

At this point, your cluster is live and operational but contains only a single node. Next, scale your cluster by setting up additional nodes that will join the cluster.

1. SSH to another instance:

	~~~
	$ ssh -i <path to AWS .pem> <username>@<additional node external IP address>
	~~~

2. Install CockroachDB from our latest binary:

	~~~ shell
	# Get the latest CockroachDB tarball.
	$ wget https://binaries.cockroachdb.com/cockroach-{{ page.release_info.version }}.linux-amd64.tgz

	# Extract the binary.
	$ tar -xf cockroach-{{ page.release_info.version }}.linux-amd64.tgz  \
	--strip=1 cockroach-{{ page.release_info.version }}.linux-amd64/cockroach

	# Move the binary.
	$ sudo mv cockroach /usr/local/bin
	~~~

3. Start a new node that joins the cluster using the first node's internal IP address:

	~~~
	$ cockroach start --insecure \
	--join=<node1 internal IP address>:26257 \
    --cache=25% \
    --max-sql-memory=25% \
    --background
    ~~~

    The only difference when adding a node is that you connect it to the cluster with the `--join` flag, which takes the address and port of the first node. Otherwise, it's fine to accept all defaults; since each node is on a unique machine, using identical ports won't cause conflicts.

4. Repeat these steps for each instance you want to use as a node.

## Step 6. Test your cluster

CockroachDB replicates and distributes data for you behind-the-scenes and uses a [Gossip protocol](https://en.wikipedia.org/wiki/Gossip_protocol) to enable each node to locate data across the cluster.

To test this, use the [built-in SQL client](use-the-built-in-sql-client.html) as follows:

1. SSH to your first node:

	~~~ shell
	$ ssh -i <path to AWS .pem> <username>@<node1 external IP address>
	~~~

2. Launch the built-in SQL client and create a database:

	~~~ shell
	$ cockroach sql --insecure
	~~~
	~~~ sql
	> CREATE DATABASE insecurenodetest;
	~~~

3. In another terminal window, SSH to another node:

	~~~ shell
	$ ssh -i <path to AWS .pem> <username>@<node3 external IP address>
	~~~

4. Launch the built-in SQL client:

	~~~ shell
	$ cockroach sql --insecure
	~~~

5. View the cluster's databases, which will include `insecurenodetest`:

	~~~ sql
	> SHOW DATABASES;
	~~~
	~~~
	+--------------------+
	|      Database      |
	+--------------------+
	| crdb_internal      |
	| information_schema |
	| insecurenodetest   |
	| pg_catalog         |
	| system             |
	+--------------------+
	(5 rows)
	~~~

6. Use **CTRL + D**, **CTRL + C**, or `\q` to exit the SQL shell.

## Step 7. Test load balancing

The AWS load balancer created in [step 3](#step-3-set-up-load-balancing) can serve as the client gateway to the cluster. Instead of connecting directly to a CockroachDB node, clients can connect to the load balancer, which will then redirect the connection to a CockroachDB node.

To test this, install CockroachDB locally and use the [built-in SQL client](use-the-built-in-sql-client.html) as follows:

1. [Install CockroachDB](install-cockroachdb.html) on your local machine, if it's not there already.

2. Launch the built-in SQL client, with the `--host` flag set to the load balancer's IP address:

	~~~ shell
	$ cockroach sql --insecure \
	--host=<load balancer IP address> \
	--port=26257
	~~~

3. View the cluster's databases:

	~~~ sql
	> SHOW DATABASES;
	~~~
	~~~
	+--------------------+
	|      Database      |
	+--------------------+
	| crdb_internal      |
	| information_schema |
	| insecurenodetest   |
	| pg_catalog         |
	| system             |
	+--------------------+
	(5 rows)
	~~~

	As you can see, the load balancer redirected the query to one of the CockroachDB nodes.

4. Use the `node_id` [session variable](show-vars.html) to check which node you were redirected to:

    {% include copy-clipboard.html %}
    ~~~ sql
    > SHOW node_id;
    ~~~

	~~~
	+---------+
	| node_id |
	+---------+
	|       3 |
	+---------+
	(1 row)
	~~~

5. Use **CTRL + D**, **CTRL + C**, or `\q` to exit the SQL shell.

## Step 8. Monitor the cluster

View your cluster's Admin UI by going to `http://<any node's external IP address>:8080`.

On this page, verify that the cluster is running as expected:

1. Click **View nodes list** on the right to ensure that all of your nodes successfully joined the cluster.

2. Click the **Databases** tab on the left to verify that `insecurenodetest` is listed.

{% include prometheus-callout.html %}

## Step 9. Use the database

Now that your deployment is working, you can:

1. [Implement your data model](sql-statements.html).
2. [Create users](create-and-manage-users.html) and [grant them privileges](grant.html).
3. [Connect your application](install-client-drivers.html). Be sure to connect your application to the AWS load balancer, not to a CockroachDB node.

## See Also

- [Google Cloud Platform GCE Deployment](deploy-cockroachdb-on-google-cloud-platform.html)
- [Digital Ocean Deployment](deploy-cockroachdb-on-digital-ocean.html)
- [Azure Deployment](deploy-cockroachdb-on-microsoft-azure.html)
- [Manual Deployment](manual-deployment.html)
- [Orchestration](orchestration.html)
- [Start a Local Cluster](start-a-local-cluster.html)

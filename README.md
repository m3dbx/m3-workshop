# M3 Day Workshop
 
## About

This repository contains all necessary bits to run the M3 workshop stack locally on either Mac or Windows. Before getting started with the workshop, make sure you have met the below prerequisites. During the workshop, we will be following the steps using a three-node M3DB cluster. If you don't have the capacity available for this, we have created a single-node version to follow along with as well. 

### Prerequisites 

- Download [Docker Desktop](https://www.docker.com/products/docker-desktop) for Mac or Windows. [Docker Compose](https://docs.docker.com/compose) will be automatically installed. On Linux, make sure you have the latest version of [Compose](https://docs.docker.com/compose/install/). 

- Adjust "Resources" in Docker to have at least 8 GB of memory. (If using the single-node M3DB cluster, setting it to 3 GB will be sufficient.) 

## Stack overview

The stack consists of the latest versions of these components:

- A standalone M3DB node or a cluster of 3 nodes;
- [M3 Coordinator](https://m3db.io/docs/m3coordinator/) and [M3 Query](https://m3db.io/docs/m3query/) instances to interact w/ the [M3DB](https://m3db.io/docs/m3db/) nodes;
- Two [Prometheus](https://prometheus.io/docs/introduction/overview/) instances;
- A single [Grafana](https://grafana.com/) instance.

Both Prometheus instances are configured to scrape themselves, and a slightly different sets of services. During the 
workshop, we'll demonstrate how querying the data on separate Prometheus instances will lead to different results. 

The M3 Coordinator instance will take all read and write requests, and then distributes them to the M3DB cluster. It will also implement [Prometheus Remote Read and Write HTTP endpoints](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#remote_write), which we'll use when modifying the Prometheus instances during the workshop.

Finally, the M3 Query instance will be used to query all data from the M3DB cluster via Grafana.

From the start of the workshop, Grafana will be configured with 3 data sources - Prometheus01, Prometheus02, and M3Query: 

- [Prometheus instance 1](http://localhost:9090)
- [Prometheus instance 2](http://localhost:9091)
- [M3 Query endpoint](http://localhost:7221)

![Architecture diagram](./m3-workshop-schema.png)

**List of container instances**

Below is a list of the container instances with descriptions and names to be used throughout the workshop:

| Container   | Endpoints 	| Notes		|
| ----------- | ----------- |-----------|
| prometheus01| [http://localhost:9090](http://localhost:9090)|The first Prometheus instance, scrapes itself and all M3 services, except M3 Query|
| prometheus02| [http://localhost:9091](http://localhost:9091)|The second Prometheus instance, scrapes all M3 services in the stack|
| grafana| [http://localhost:3000](http://localhost:3000)||
| m3db_seed	  | localhost:2379; localhost:909[0-2]| M3DB instance, running built-in etcd service (2379 TCP port). Runs in both single-node and cluster modes|
| m3db_data01 | localhost:909[0-2]  | This M3DB node runs in cluster mode only |
| m3db_data02 | localhost:909[0-2]  | This M3DB node runs in cluster mode only |
| m3coordinator01| 0.0.0.0:7201 | Exposes Prometheus Remote Read and Write API on TCP 7201 port |
| m3query01 	| 0.0.0.0:7221  | Exposes Prometheus Remote Read API on TCP 7221 port, used as a Grafana data source to query data in the M3DB cluster|
| provisioner | N/A | Prepares M3DB cluster on startup (creates M3DB namespace, placements)|

## Instructions for the workshop

### Step 1: Go to the M3 Workshop repo and clone the repo onto your local machine: 

Link to repo: https://github.com/m3dbx/m3-workshop

### Step 2: Start up the stack via Docker-Compose

For the three node M3DB cluster, run the following command:

```$:~ docker-compose up```

Once you see the following output (with code 0 at the end), the stack is configured and ready to be used (this will take ~30 seconds): 

```
provisioner_1      | Waiting until shards are marked as available
provisioner_1      | Provisioning is done.
provisioner_1      | Prometheus available at http://localhost:9090
provisioner_1      | Prometheus replica is available at http://localhost:9091
provisioner_1      | Grafana available at http://localhost:3000
m3-workshop_provisioner_1 exited with code 0
```

Logs of the `provisioner` process can be seen either by following the output of the `docker-compose up` or by running the following command: ```docker-compose logs provisioner```

** If wanting to run the **single M3DB node**, run the following command instead:

```$:~ docker-compose -f single-node.yml up```

### Step 3: Open up Grafana 

Once the stack is up and running, login into [Grafana](http://localhost:3000) using `admin:admin` credentials. Press "skip screen" in Grafana when prompting you to set a password. 

Once in the Grafana UI, go to the [Explore](http://localhost:3000/explore) tab. From here, you will see a dropdown at the top of the page with the three data sources - Prometheus01, Prometheus02, and M3Query. 

### Step 4: Explore the Prometheus data sources in Grafana via Explore and Dashboard tabs

**Explore:** In the [Explore](http://localhost:3000/explore) tab of Grafana, you will see three data sources - Prometheus01, Prometheus02, and M3Query. Only the 2 Prometheus instances will be emitting metrics (scraping themselves), as remote read and write HTTP endpoints have not been enabled yet for your M3DB cluster. 

Try switching between the two Prometheus data sources, and running the query command: 'up{}'. Change the resolution of your graph to "Last 5 minutes" to get results. When switching between the two Prometheus instances, you will notice that each of the Prometheus instances are emitting slightly different sets of metrics (look under the "Table" section). In order to combine the metris from both instances, we will need to add the Remote Read and Write HTTP endpoints to our Prometheus configurations. This will allow metrics to then be sent to our M3DB cluster. 

**Dashboard:** If you want to see multiple metrics for your data sources in one place, go to the [Dashboard](http://localhost:3000/?orgId=1) tab in Grafana. There will be some default dashboards already created, such as `Prometheus Stats`. Within the Dashboard, if you want to explore a certain metric (e.g. `prometheus_target_interval_length_seconds`) then click on the dropdown at the top of the metric and go to `Explore`. This will direct you to a more detailed look at that particular metric. Check it out! 

### Step 5: Sending Prometheus metrics to the M3DB cluster

To start sending metrics from the two Prometheus instances to the M3DB cluster, we need to enable remote write functionality:

- In your code editor (e.g. VSCode), go to the Prometheus folder under Config. There will be two `yml` config files there. At the bottom of each config file, there will be a Remote Read and Write section commented out (in green). Uncomment both `remote_read` and `remote_write` blocks in [./config/prometheus/prometheus01.yml](./config/prometheus/prometheus01.yml) and [./config/prometheus/prometheus02.yml](./config/prometheus/prometheus02.yml) config files. Once this is done, save your changes locally. 
- Then run `docker-compose restart prometheus01 prometheus02`
- Once they're reloaded, head back to the [Explore](http://localhost:3000/explore) tab in Grafana, and switch to the `M3 Query` data source to run PromQL queries (e.g. `up{}`). By doing so, you will now see that metrics across both Prometheus instances are being emitted to the `M3 Query` data source - creating a single point of query across your instances. 

### Step 6: Spin down one of the M3DB nodes (if running 3 node cluster) and query Prometheus metrics 

- When performing reads or writes, M3 Coordinator and M3 Query utilize quorum logic in order to successfully complete requests (i.e. in a three node M3DB cluster, the Coordinator must successfully write at least 2 of the 3 copies of data to M3DB). In order to demonstrate this, we will be spinning down one of the M3DB nodes (**only if using 3 node cluster**), and querying against the remaining two DB nodes. 
- For the workshop, we will spin down `m3db_data01` by running the following command (**Note:** Do NOT stop the `m3db_seed` instance): 

```$:~ docker-compose stop m3db_data01```

- Once this is down, return to your `M3Query` data source in Grafana, and run a query command (e.g. `up{}`). You will see that the single node has been dropped, but that all remaining instances are still being successfully queried. 

**Note:** If you want to re-start `m3db_data01`, run the following command:

```$:~ docker-compose start m3db_data01```

### Step 7: Spinning down the stack

Press `Ctrl+C` to interrupt the already running `docker-compose up` process, or run the following command:

```$:~ docker-compose down```

**Recommended step:** it leaves container disks intact. If you want to get rid of the data as well, run the following command:

```$:~ docker-compose down -v```

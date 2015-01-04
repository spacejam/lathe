Lathe
=====
Lathe provides modular abstractions for running code in distributed environments.  You provide the business logic, and Lathe can handle deployment, monitoring, dynamic reconfiguration, and persistence.  It supports low-latency synchronous services, asynchronous data processing systems, persistence, caching, or any mix that you need.  It comes with some high level interfaces if you want to use them, but it makes no assumptions about how you need to use the core functionality of the system: service discovery and scheduling.  This is what makes it a micro-framework: it provides some modular tools but does not force "the one true way".  It is inspired by [Samza](https://github.com/apache/incubator-samza), [Storm](https://github.com/apache/storm), [Consul](https://github.com/hashicorp/consul), [groupcache](https://github.com/golang/groupcache) and [Circuit](https://github.com/gocircuit/circuit).
###### Requirements 
1. Mesos
2. Zookeeper
3. Kafka is optional for asynchronous flows and persistent state replication

### Core Components
#### Scheduling
You can tag your tasks and provide affinities.  For instance, if you want to run several databases on your cluster, but want to guarantee that only one will ever be scheduled on a particular machine, you may tag each task with "heavy_disk" and set the task's bias for "heavy_disk" to "never".  Other choices are "always", "prefer", "avoid", and "neutral".  "neutral" is the default if you do not specify a bias on either task.  If you want a service to be colocated with a database, set the bias on either or both tasks for the other to a positive number.  When you submit a configuration, if the constraints that you specify are impossible to satisfy, you will be given an error and no action will be taken.
#### Service Discovery
A configurable number of metadata registry instances share the responsibilities of liveness monitoring, reliable configuration change broadcasting, and serving of a REST API for requesting current configuration details as well as sending metadata to a functional group of tasks.  There is also an agent daemon that you may optionally use for providing a large number of external hosts with a local copy of the metadata state.  The local agent uses a combination of gossip and periodic polling of the registries to receive updates.  This is useful if you have a number of services in your Lathe cluster that are relied on by external services.  For instance, if you run a cache in Lathe that your external web servers make requests to, and the task for the cache is restarted on a different host or port, you may configure the agent to execute a script that updates the web application's configuration when the cache membership changes.
### Optional Abstractions
#### Monitoring
Similar to how you may configure external agents to execute specified scripts when certain types of events happen, you may also configure the metadata registry to do this.  For example, if you already have a timeseries database that you run nagios checks against periodically, you may have one of the metadata registries execute a script that writes a value into the database whenever a task dies.  You may then have nagios check whether more than 5 tasks have died in the last hour, and generate an alert when the threshold is crossed.
#### Connection Abstractions
When a task dies and is restarted, this provides runtime reconfiguration.  Traffic cutover, naive load balancing and double-writing become configuration details rather than modifications to your code.
#### Persistence
Inspired by Samza: creates a local RocksDB instance that is optionally replicated to Kafka to be inherited by the tasks replacement should it die.
#### Caching
Based on groupcache.

my_topology.toml:
```toml
[task.http.lb]
crate = "http_loadbalancer"  # required
version = "0.8.2"            # required
task = "my::LoadBalancer"    # required
tasks = 3                    # required
tags = [ "lb", "net" ]       # optional, assists scheduler properly distribute
biases = [                   # optional scheduler biases based on tags
  ["lb", "frontpage"],
  [
    -10,  # avoid scheduling multiple loadbalancers on the same machine
    7     # prefer colocating with frontpage tasks to improve locality
  ]                   
]
backend_distribution = [     # you can provide your own keys to be accessed from the task
  ["task.http.frontpage.v4-2-1", "task.http.frontpage.v4-2-2"],
  [90, 10]
]

[task.http.frontpage.v4-2-1]
crate = "http_frontpage"
version = "4.2.1"
tasks = 5
task = "my::FrontPage"
tags = [ "frontpage" ]

[task.http.frontpage.v4-2-2]
crate = "http_frontpage"
version = "4.2.2"
tasks = 5
task = "my::FrontPage"
tags = [ "frontpage" ]
```
source for http_loadbalancer:
```rust
extern crate lathe;
use std::rand;
use std::collections::TreeMap;
use lathe::http::{HTTPTask,HTTPResult};
use lathe::config::{task,registry};
use lathe::task::{TaskRef};

fn gethost() -> TaskRef {
  let destinations = registry.lookup_tasks("backend_distribution").unwrap();
  let mut total = 0i;
  // don't be this lazy in production
  let mut map = TreeMap::new();
  for (task, weight) in destinations {
    total += weight;
    map.insert(weight, task);
  }
  let index = rand::random() * total;
  registry.get_ref(map.find(index));  // this is cached locally, lazily populated on miss, and actively refreshed
}

struct LoadBalancer;
impl HTTPTask for LoadBalancer {
  fn handle(request: HTTPRequest) -> Future<HTTPResult> {
    request.forward(gethost())
  }
}
```
source for one of the http_frontpage's:
```rust
extern crate lathe;
use lathe::http::{HTTPTask,HTTPResult,HTTPResponse};

struct FrontPage;
impl HTTPTask for FrontPage {
  fn handle(request: HTTPRequest) -> Future<HTTPResult> {
    match request.path {
      ["get", "index"] => Future.from_value(HTTPResponse(200, ":)")),
      _ => Future.from_value(HTTPResponse(400, "invalid request")),
    }
  }
}
```

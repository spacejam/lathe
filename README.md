Lathe
=====
Lathe is a topology manager and service discovery mechanism.  It supports low-latency synchronous services, asynchronous data processing systems, or a mixture of each.  It includes a metadata registry that allows you to notify your topologies about configuration changes external to the cluster, and to reliably deliver configuration changes of the cluster to external machines.  It is inspired by [Samza](https://github.com/apache/incubator-samza), [Storm](https://github.com/apache/storm), [Consul](https://github.com/hashicorp/consul), [groupcache](https://github.com/golang/groupcache) and [Circuit](https://github.com/gocircuit/circuit).
* Requirements: YARN and Zookeeper.  Kafka is optional for asynchronous flows and persistent state replication.
* Components:  metadata registry, tasks (similar to a Storm bolt or a Samza task), external metadata agents.
* The metadata registry provides a REST api for reading and writing metadata.
* Zookeeper stores metadata state, but the registry is responsible for dissemination of changes.  This keeps load on Zookeeper low, and allows the metadata registry to be killed without worry.
* The client agent allows external nodes to be reliably notified of membership or metadata changes using a mixture of gossip and polling of the registry.
* Tries to keep your code running, and will restart failed nodes.
* Rewires data flows across functional groups and reliably notifies client agents of membership changes.
* Out of the box client agent support for notifying Varnish and HAProxy of topology changes.  You configure the client agent with actions corresponding to changes to specific subtrees of metadata.
* Asynchronous flows may use an external Kafka cluster for transport.
* You can pass arbitrary metadata to a functional group through the registry.  This is useful if you rely on an external server or cluster that can change membership, without having to restart your lathe topology.
* Persistent state API: local rocksdb with optional double writing to Kafka for recovery.
* Distributed cache API: based on groupcache.
* Slow-rollout upgrade strategies for tasks.
* In the sense that Storm and Samza are processing frameworks, this may be viewed as a micro-framework.  It imposes less doctrine, and gives users more flexibility.  You can block your thread, you can create other threads, but you can also use an abstraction with more batteries included (see HTTPTask below).

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
  let destinations = task.lookup("backend_distribution").unwrap();
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

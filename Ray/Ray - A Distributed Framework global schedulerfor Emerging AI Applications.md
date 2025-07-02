
- Lineage-based fault tolerance for tasks and actors
- Replication-based fault tolerance for the metadata store
- Ray is not a substitute for generic data-parallel frameworks, such as Spark as it currently lacks the rich functionality and APIs (e.g., straggler mitigation, query optimization)
- We design and build the first distributed framework that unifies training, simulation, and serving— necessary components of emerging RL applications.
- To support these workloads, we unify the actor and task-parallel abstractions on top of a dynamic task execution engine
-  To achieve scalability and fault tolerance, we propose a system design principle in which control state is stored in a sharded metadata store and all other system components are stateless.
-  To achieve scalability, we propose a bottom-up distributed scheduling strategy
## Architecture
### System Layer
![[Screenshot 2025-05-19 at 11.16.34 AM.png]]
#### Global Control Store (GCS)(Metadata store)
- Maintains the entire control state of the system
- A key-value store with pub-sub functionality
- Scale -> sharding, Fault tolerance -> per-shard chain replication
- Need to dynamically spawn millions of tasks per second -> fault tolerance and low latency
- Enables every component in the system to be stateless
- Lineage-based fault tolerance
Originally GCS use a redis instance Before ray1.9...
#### Bottom-Up Distributed Scheduler
Ray needs to dynamically schedule millions of tasks per second, tasks which may take as little as a few milliseconds. None of the cluster schedulers we are aware of meet these requirements.
- Most cluster computing frameworks, such as Spark, CIEL , and Dryad implement a centralized scheduler, which can provide locality but at latencies in the tens of ms
- Distributed schedulers such as work stealing, Sparrow & Canary can achieve high scale, but they either don’t consider data locality, or assume tasks belong to independent jobs, or assume the computation graph is known.
- Ray design a two-level hierarchical scheduler consisting of a global scheduler and per-node local schedulers
- Tasks created -> submit to node’s local scheduler -> local scheduler schedules tasks locally unless the node is overloaded (i.e., its local task queue exceeds a predefined threshold), or it cannot satisfy a task’s requirements (e.g., lacks a GPU). -> forward to global scheduler if a local scheduler decides not to schedule a task locally. **Bottom-Up Distributed Scheduler is named by this since it attempts to schedule tasks locally first.** 
![[Screenshot 2025-05-19 at 2.19.19 PM.png]]
- Global scheduler making scheduling decisions based on: each node’s load, task’s constraints.
- More precisely, global scheduler identifies the set of nodes that have enough resources of the type requested by the task, and from these nodes selects the node which provides the lowest estimated waiting time.
  At a given node, this time is the sum of (i) the estimated time the task will be queued at that node (i.e., task queue size times average task execution), and (ii) the estimated transfer time of task’s remote inputs (i.e., total size of remote inputs divided by average bandwidth).
- Global scheduler gets the queue size at each node and the node resource availability via **heartbeats**, and the location of the task’s inputs and their sizes from **GCS**
- If the global scheduler becomes a bottleneck, we can **instantiate more replicas** all sharing the same information via GCS. This makes our scheduler architecture **highly scalable.**

#### In-Memory Distributed Object Store
- Purpose: To minimize task latency, implemented an in-memory distributed storage system to store the inputs and outputs of every task, or stateless computation. 
- zero-copy data sharing between tasks running on the same node
- Data format : Apache Arrow
- If a task’s inputs are not local, the inputs are replicated to the local object store before execution. Also, a task writes its outputs to the local object store.
- For low latency, we keep objects entirely in memory and evict them as needed to disk using an LRU policy.
- Just as existing cluster computing frameworks, such as Spark and Dryad, Ray objects are immutable. This simplifies support for fault tolerance. Ray recovers any needed objects through lineage re-execution using lineage stored in the GCS that tracks stateless tasks during initial execution.
- Ray object store does not support distributed objects, i.e., each object fits on a single node.
#### Implementation


![[Screenshot 2025-05-19 at 5.51.32 PM.png]]
0. Remote function add() is automatically registered with the GCS upon initialization and distributed to every worker in the system
1. Driver submits add(a, b) to the local scheduler
2. Forwards it to a global scheduler
3. Global scheduler looks up the locations of add(a, b)’s arguments in the GCS (step 3) 
4. Global schedule decides to schedule the task on node N2 which stores argument b
5. Local scheduler at node N2 checks whether the local object store contains add(a, b)’s arguments 
6. Local store doesn’t have object a, it looks up a’s location in the GCS
7. Learning that a is stored at N1, N2’s object store replicates it locally
8. As all arguments of add() are now stored locally, the local scheduler invokes add() at a local worker
9. Local worker execute with arguments in its object store.

![[Screenshot 2025-05-19 at 5.51.42 PM.png]]
1. Driver checks the local object store for the value c, using the future idc returned by add()
2. local object store doesn’t store c, it looks up its location in the GCS.At this time, there is no entry for c, as c has not been created yet. As a result, N1’s object store registers a callback with the Object Table to be triggered when c’s entry has been created
3. Meanwhile, at N2, add() completes its execution, stores the result c in the local object store
4. Which in turn adds c’s entry to the GCS 
5. GCS triggers a callback to N1’s object store with c’s entry
6. N1 replicates c from N2
7. Return result form N1's local obj store

## Reference
https://www.anyscale.com/blog/redis-in-ray-past-and-future
https://www.usenix.org/system/files/osdi18-moritz.pdf
https://zhuanlan.zhihu.com/p/14525928746
https://stephanie-wang.github.io/pdfs/nsdi21-ownership.pdf
## 前言

這是初探ray core 的第一篇，原本在寫的時候想要先從更higher level 寫一篇overview，但不小心看太深入，所以決定先直接把GCS寫完，等所有componenets都寫完再回頭寫

作為Ray Core中最關鍵的component之一，GCS (Global Control Service) 負責控制整個Ray cluster的行為並存儲cluster的metadata。
GCS提供了以下核心功能：

- Node Management - 節點生命周期管理
- Resource Management - cluster資源調度與分配
- Actor Management - Actor life cycle與狀態管理
- Placement Group Management 
- Worker Management 
- Metadata Store - cluster的元數據儲存
- Runtime Environment - 運行時環境管理
## GCS Overview
雖然GCS看起來很複雜，不過一言以蔽之，GCS server = gRPC server + many managers  + optional back store (Redis)
![](attachments/Screenshot%202025-07-17%20at%202.04.23%20PM.png)
幾乎都是用callback 來達成async
## GCS Entry Point
`src/ray/gcs/gcs_server/gcs_server_main.cc`
- Parse Flags e.g.(redis_address, gcs_server_port, etc.)
- Install signal handler
- Init Ray Event
- Pass config and Instantiate GCS Server
- Start ASIO event loop (Create main service io context) 大部分GCS中的service都是共用DefaultIOContext，除了一些resource intensive的service 例如：GCS task manager
## GCS Server
`src/ray/gcs/gcs_server/gcs_server.cc`
- Create Redis Client if `REDIS_PERSIST` is set
- Init GCS Publisher
- Init Managers including
- Install Event listeners
- Collect metrics  every `RayConfig::instance().metrics_report_interval_ms() /2 ` ms by putting the task into asio event loop
- Start gRPC server (after)
### Init 

### Managers

#### Node manager
![](attachments/Screenshot%202025-07-17%20at%202.06.04%20PM.png)
基本上就是維護這幾個map ＆ 維護有哪些listener 在乎 node added or removed
```cpp
// NodeId <-> ip:port of raylet
using NodeIDAddrBiMap =
boost::bimap<boost::bimaps::unordered_set_of<NodeID, std::hash<NodeID>>,
boost::bimaps::unordered_multiset_of<std::string>>;
absl::flat_hash_map<NodeID, std::shared_ptr<rpc::GcsNodeInfo>> alive_nodes_;
absl::flat_hash_map<NodeID, std::shared_ptr<rpc::autoscaler::DrainNodeRequest>>draining_nodes_;
absl::flat_hash_map<NodeID, std::shared_ptr<rpc::GcsNodeInfo>> dead_nodes_;
```
GCS 也會藉由定期向raylets heart checks 來監控node的狀態，並且也會把這些資訊broadcast給所有node，這樣可以讓個node的raylet得知全域得node 資訊，如果有node failed他們就可以clean up 關於failed node的資源，e.g. 某個task可能是被failed node submit的。
#### Cluster Task Manager
<!---
這是raylet的 不小心看太深入...
-->
負責把task schedule 到cluster 中的其中一個node，根據以下邏輯：
1. 先把tasks queue起來
2. 選擇有足夠資源的節點去運行
3. 如果infeasible 放到infeasible queue -> 跟GCS report -> notify auto scaler -> start new node
每次被rpc invoke `ScheduleAndDispatchTasks` 時都會 `ScheduleAndDispatchTasks` 並嘗試看看infeasible queue中的task 能否被schedule了(每個在infeasible queue 的 schedule class 都會被拿去問 `cluster_resource_scheduler_` 一次)要是feasible了就把他丟回 pending task queue
infeasible queue & pending task queue 每個deque都只會驗第一個能不能被schedule 如果不feasible就會提早結束（因為一個schedule class 中的queue中的task shape都一樣

#### Gcs Resource Manager
[[core] Introduce pull based health check to GCS.](https://github.com/ray-project/ray/pull/29442)

雖然是GCS "server" 但卻是 pull base
`absl::flat_hash_map<NodeID, HealthCheckContext *> health_check_contexts_;` 
維護 node Id 對應HealthCheckContext 的 map, Add/Remove/Fail(動詞) node 時 會改動這個map.
HealthCheckContext自己是在GcsHealthCheckManager的一個class，裡面存了
- nodeID
- `lastest_known_healthy_timestamp_` 
- bool `stopped_`,
- rpc request HealthCheckRequest
- 剩餘 health check 次數 health_check_remaining_ 一開始該是多少由`manager->failure_threshold_`決定
- `timer_` `boost::asio::deadline_timer`  timer for this context -> 用來控制多久跑一次health check
- grpc stub
因為constructor 裡面定義 HealthCheckContext 被create 時就會自動起好timer 並設置 第一次health check 的delay
```cpp
  class HealthCheckContext {
   public:
    HealthCheckContext(const std::shared_ptr<GcsHealthCheckManager> &manager,
                       const std::shared_ptr<grpc::Channel> &channel,
                       NodeID node_id)
        : manager_(manager),
          node_id_(node_id),
          timer_(manager->io_service_),
          health_check_remaining_(manager->failure_threshold_) {
      request_.set_service(node_id.Hex());
      stub_ = grpc::health::v1::Health::NewStub(channel);
      timer_.expires_from_now(
          boost::posix_time::milliseconds(manager->initial_delay_ms_));
      timer_.async_wait([this](auto) { StartHealthCheck(); });
    }
```
然後在`StartHealthCheck()` 中再更改設置timer -> 把這個funcction放入asio loop 以實現定時執行。
StartHealthCheck 裡面就是 grpc 去call node 然後看status 失敗的話就health_check_remaining_ -1 ，再把自己丟到asio loop，到直到remaining < 0 ，就認定這個node 死了

另外 ray syncer 大概也可以順便知道health情形，所以可以把最後的 `lastest_known_healthy_timestamp_` 往後推，`if (now <= next_check_time)`  就把現在這個return 然後 重新開一個timer

#### GCS Job Manager
這個感覺比較有趣
負責管理Job 的狀態以及通知關心job finish 的listener

// 什麼會被放在GCS table -> 無法被歸類在任一種manager下 e.g. Job table 除會被 job manager使用到 GCS client 也會發rpc 要這些資料 ->後面會講 （JobInfoAccessor）(待商榷)
#### Placement Group
Bundle
> a collection of “resources”. It could be a single resource, `{"CPU": 1}`, or a group of resources, `{"CPU": 1, "GPU": 4}`. A bundle is a unit of reservation for placement groups. “Scheduling a bundle” means we find a node that fits the bundle and reserve the resources specified by the bundle. A bundle must be able to fit on a single node on the Ray cluster. For example, if you only have an 8 CPU node, and if you have a bundle that requires `{"CPU": 9}`, this bundle cannot be scheduled.

Placement group -> list of bundle

##### Scheduler
-  Two-Phase Commit -> Preparing & Commiting 
- callback async
##### Manager
控制placement group 的 life cycle
```cpp
absl::btree_multimap<int64_t, std::pair<ExponentialBackoff, std::shared_ptr<GcsPlacementGroup>>> pending_placement_groups_;
```
- Tracks placement group states (PENDING → PREPARED → CREATED → REMOVED)
#### Actor 
和placement group 一樣，actor 在GCS中也被拆分為scheduler & manager
##### Scheduler
GCS提供了兩種actor scheduling 的方式，GCS-Based Scheduling 及 Raylet-Based Scheduling， 可以設定`gcs_actor_scheduling_enabled` 決定要用哪一種（預設為false），選用GCS作scheduling的話會把actor creation task 丟給 cluster task manager 去queuing。
```cpp
if (RayConfig::instance().gcs_actor_scheduling_enabled() && !actor->GetCreationTaskSpecification().GetRequiredResources().IsEmpty()) {
    ScheduleByGcs(actor);
  } else {
    ScheduleByRaylet(actor);
}
```

如果選raylet scheduling 的話，會把actor creation 的task forward 給raylet 去決定要schedule到哪裡，forward時會盡量forward到 actor 的owner 上，沒owner的actor則會randomly 從alive node 選一個然後把task 綁到node上。
> The owner of an actor is the worker that originally created the actor by calling `ActorClass.remote()`. [Detached actors](https://docs.ray.io/en/latest/ray-core/actors/named-actors.html#actor-lifetimes) do not have an owner process and are cleaned up when the Ray cluster is destroyed.

兩個的差別是 如果用GCS去作scheduling的話，可以consider cluster-wide的資源是否足夠，可以做出比較好的scheduling decision，raylet的話則是可以把loading distributed一點。

決定好要把actor 放在哪裡後會向該node的worker租借（lease）資源，成功lease之後便會在worker上create 該actor。
```cpp
void GcsActorScheduler::CreateActorOnWorker(std::shared_ptr<GcsActor> actor,std::shared_ptr<GcsLeasedWorker> worker) {
  auto request = std::make_unique<rpc::PushTaskRequest>();
  request->set_intended_worker_id(worker->GetWorkerID().Binary());
  request->mutable_task_spec()->CopyFrom(
      actor->GetCreationTaskSpecification().GetMessage());
  google::protobuf::RepeatedPtrField<rpc::ResourceMapEntry> resources;
  for (auto resource : worker->GetLeasedResources()) {
    resources.Add(std::move(resource));
  }
  request->mutable_resource_mapping()->CopyFrom(resources);

  auto client = worker_client_pool_.GetOrConnect(worker->GetAddress());
  client->PushNormalTask(request, some_callback);
```

##### Manager 
```cpp
absl::flat_hash_map<ActorID, std::vector<std::function<void(Status)>>> actor_to_register_callbacks_;

absl::flat_hash_map<ActorID, std::vector<RestartActorForLineageReconstructionCallback>> 
actor_to_restart_for_lineage_reconstruction_callbacks_;

absl::flat_hash_map<ActorID, std::vector<CreateActorCallback>> actor_to_create_callbacks_;

absl::flat_hash_map<ActorID, std::shared_ptr<GcsActor>> registered_actors_;

absl::flat_hash_map<ActorID, std::shared_ptr<GcsActor>> destroyed_actors_;

std::list<std::pair<ActorID, int64_t>> sorted_destroyed_actor_list_;

absl::flat_hash_map<std::string, absl::flat_hash_map<std::string, ActorID>> named_actors_;

absl::flat_hash_map<NodeID, absl::flat_hash_map<WorkerID, 
absl::flat_hash_set<ActorID>>> unresolved_actors_;

std::vector<std::shared_ptr<GcsActor>> pending_actors_;
absl::flat_hash_map<NodeID, absl::flat_hash_map<WorkerID, ActorID>> created_actors_;

absl::flat_hash_map<NodeID, absl::flat_hash_map<WorkerID, Owner>> owners_;
```
負責控制每個actor的 life cycle，跟其他manager很像，也是有存各actor的狀態以及其他需要的資訊（owners, callbacks等等），不過這篇沒打算太深入探討actor，之後預計會寫一篇專們說明actor的文章。
#### Worker Manager
Worker manager 主要就作這幾件事：
- 管理`gcs_table_storage_`中的Worker Table狀態
- Handle Worker Add, Dead 以及其他component request get worker
- 如果有 worker failure就通知listeners
#### GCS Task Manager
控制task的life cycle ＆ transition
有別於其他的map，因為task event可能會有很多，因此超出limit時(`RAY_CONFIG(int64_t, task_events_max_num_task_in_gcs, 100000)`)當數量GCS task manager 會根據priority + LRU 去evict掉一些tasks.
Priority Strategy:
```
Priority Strategy:
Finished tasks (priority 0): Evicted first
Actor tasks (priority 1): Medium retention
Running tasks (priority 2): Highest retention
```

task數量一大query起來也會變慢，因此其中也開了多個map，分別處理不同的query 需求。
```cpp
 // Primary index from task attempt to the locator.
absl::flat_hash_map<TaskAttempt, std::shared_ptr<TaskEventLocator>> primary_index_;
// Secondary indices for retrieval.
absl::flat_hash_map<TaskID,absl::flat_hash_set<std::shared_ptr<TaskEventLocator>>>task_index_;
absl::flat_hash_map<JobID,absl::flat_hash_set<std::shared_ptr<TaskEventLocator>>>job_index_;
absl::flat_hash_map<WorkerID,absl::flat_hash_set<std::shared_ptr<TaskEventLocator>>>worker_index_;
```

值得一提的是GCS Task Manager擁有自己專屬的io_context and io_thread （沒有和其他GCS services一起使用）我想是因為GCS是被design要處理millions of task per second，所以讓他自己有一個thread才不會把其他的service block住。

--- 

你會發現其實這些managers講白了就是在管理特定資源的狀態，並控制各個狀態間的transition. 

## GCS Client
### Accessor

## Memory Store
## 賺到的PR
在trace GCS code時，意外賺到了幾個PR。
推薦大家在讀code的時候可以想看看哪裡是可以improve的，然後發支PR。
> 摸蛤兼洗褲，凡走過必留下PR。

![](attachments/Screenshot%202025-07-16%20at%201.41.37%20PM.png)
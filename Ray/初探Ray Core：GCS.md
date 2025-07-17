## 前言

這是初探ray core 的第一篇，原本在寫的時候想要先從更higher level 寫一篇overview，但不小心看太深入，所以決定先直接把GCS寫完，等所有components都寫完再回頭寫整個ray core。

--- 
作為Ray Core中最關鍵的component之一，GCS (Global Control Service) 負責控制整個Ray cluster的行為並存儲cluster的metadata。
GCS提供了以下重要功能：

- Node Management - node life cycle管理
- Resource Management - cluster資源scheduling與分配
- Actor Management - Actor life cycle與狀態管理
- Placement Group Management 
- Worker Management 
- Metadata Store - cluster的元數據儲存
## GCS Overview
雖然GCS看起來很複雜，不過一言以蔽之，GCS server = gRPC server + many managers  + Table store & optional back store (Redis)
![](attachments/Screenshot%202025-07-17%20at%202.04.23%20PM.png)
## GCS Entry Point
`src/ray/gcs/gcs_server/gcs_server_main.cc`

- Parse command line flags（如 redis_address、gcs_server_port 等）
- Install signal handler（SIGTERM）處理gracefully shutdown 等
- Initialize Ray Event 用於事件追蹤
- 傳遞config 並 Instantiate GCS Server
- 建立並啟動主要的 ASIO event loop（大部分 GCS service 都共用 DefaultIOContext，除了一些 resource intensive 的 service 如 GCS Task Manager）
## GCS Server
`src/ray/gcs/gcs_server/gcs_server.cc`
GCS Server 的核心初始化會做這些事：
1. Storage 初始化：根據config create Redis Client 或 In-Memory storage
2. GCS Publisher 初始化
3. 初始化 Managers
4. Event Listeners 安裝：建立各組件間的事件通信
5. 定期任務設置：設置 metrics 收集、debug message輸出等定期任務
6. 啟動gRPC Server 
### Managers
GCS server提供了不同資源的manager，你會發現其實這些managers講白了就是在管理特定資源的狀態，並控制各個狀態間的transition. 
#### Node manager
![](attachments/Screenshot%202025-07-17%20at%202.06.04%20PM.png)
基本上就是維護這幾個map ＆ 維護有哪些listener 在乎 node added or removed
```cpp
// NodeId <-> ip:port of raylet
using NodeIDAddrBiMap =
boost::bimap<boost::bimaps::unordered_set_of<NodeID, std::hash<NodeID>>,
boost::bimaps::unordered_multiset_of<std::string>>;

absl::flat_hash_map<NodeID, std::shared_ptr<rpc::GcsNodeInfo>> alive_nodes_;

absl::flat_hash_map<NodeID, std::shared_ptr<rpc::autoscaler::DrainNodeRequest>> draining_nodes_;

absl::flat_hash_map<NodeID, std::shared_ptr<rpc::GcsNodeInfo>> dead_nodes_;
```
GCS 也會藉由定期向raylet heart checks 來監控node的狀態，並且也會把這些資訊broadcast給所有node，這樣可以讓個node的raylet得知全域的node 資訊，如果有node failed他們就可以clean up 關於failed node的資源，e.g. 某個task可能是被failed node submit的。
#### Cluster Task Manager
負責把task schedule 到cluster 中的其中一個node，根據以下邏輯：
1. 先把tasks queue起來
2. 選擇有足夠資源的節點去運行
3. 如果infeasible 放到infeasible queue -> 跟GCS report -> notify auto scaler -> start new node
每次被rpc invoke `ScheduleAndDispatchTasks` 時都會 `ScheduleAndDispatchTasks` 並嘗試看看infeasible queue中的task 能否被schedule了(每個在infeasible queue 的 schedule class 都會被拿去問 `cluster_resource_scheduler_` 一次)要是feasible了就把他丟回 pending task queue。

infeasible queue & pending task queue 每個deque都只會驗第一個能不能被schedule，如果不feasible就會提早結束（因為一個schedule class 中的queue中的task shape都一樣
#### Gcs Health Check Manager
[[core] Introduce pull based health check to GCS.](https://github.com/ray-project/ray/pull/29442)
`absl::flat_hash_map<NodeID, HealthCheckContext *> health_check_contexts_;` 
維護 node Id 對應HealthCheckContext 的 map, Add/Remove/Fail(這邊fail是動詞) node 時 會改動這個map.

HealthCheckContext自己是在GcsHealthCheckManager的一個class，裡面存了
- nodeID
- rpc request HealthCheckRequest
- 剩餘 health check 次數 health_check_remaining_ 一開始該是多少由`manager->failure_threshold_`決定
- `timer_` `boost::asio::deadline_timer`  timer for this context -> 用來控制多久跑一次health check
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
然後在`StartHealthCheck()` 中再更改設置timer -> 把這個function放入asio loop 以實現定時執行。

StartHealthCheck 裡面就是 grpc 去call node 然後看status 失敗的話就health_check_remaining_ -1 ，再把自己丟到asio loop，到直到remaining < 0 ，就認定這個node 死了。

另外 ray syncer 大概也可以順便知道health情形，所以可以把最後的 `latest_known_healthy_timestamp_` 往後推，如果 `(now <= next_check_time)`  就把現在這個return，然後重新開一個timer推到 loop 中。

#### GCS Job Manager
這個感覺比較有趣
負責管理Job 的狀態以及通知關心job finish 的listener

// 什麼會被放在GCS table -> 無法被歸類在任一種manager下 e.g. Job table 除會被 job manager使用到 GCS client 也會發rpc 要這些資料 ->後面會講 （JobInfoAccessor）(待商榷)
#### Placement Group Manager

這邊貼一下 ray document 對於bundle的定義：
> a collection of “resources”. It could be a single resource, `{"CPU": 1}`, or a group of resources, `{"CPU": 1, "GPU": 4}`. A bundle is a unit of reservation for placement groups. “Scheduling a bundle” means we find a node that fits the bundle and reserve the resources specified by the bundle. A bundle must be able to fit on a single node on the Ray cluster. For example, if you only have an 8 CPU node, and if you have a bundle that requires `{"CPU": 9}`, this bundle cannot be scheduled.

而Placement group 基本上就是 list of bundle

Ray 支援幾種不同的 placement strategies：
- PACK: 盡量把 bundles 放在最少的節點上（適合有大量網路傳輸使用）
- SPREAD: 盡量把 bundles 分散到不同節點上
- STRICT_PACK: 強制所有 bundles 必須在同一個節點上
- STRICT_SPREAD: 強制每個 bundle 都在不同節點上
GCS 中把 placement group 的部分拆分為 Scheduler & Manager，分離了schedule的邏輯和placement group的狀態管理：
##### Scheduler
-  利用兩階段提交 (Two-Phase Commit)確定所有發到node上的prepare bundle request都是成功的，如果沒有就把資源都是放掉
  ```cpp
enum class LeasingState {
  /// The first phase of 2PC. It means requests to nodes are sent to prepare resources.
  PREPARING,
  /// The second phase of 2PC. It means that all prepare requests succeed, and GCS is
  /// committing resources to each node.
  COMMITTING,
  /// Placement group has been removed, and this leasing is not valid.
  CANCELLED
};
```
1. PREPARING 階段：
- 根據 placement strategy 選擇合適的節點
- 對每個合適的節點發送 `PrepareBundleResources` RPC
- 節點檢查是否有足夠資源，如果有就暫時預留（但不實際分配）
- 等待所有節點回覆
2. COMMITTING 階段：
- 如果所有節點都成功預留資源，發送 `CommitBundleResources` RPC
- 節點將預留的資源正式分配給該 placement group
- 如果任何一個節點失敗，發送 `CancelResourceReserve` 到所有節點釋放預留
##### Manager
負責管理、控制placement group 的 life cycle
```cpp
absl::flat_hash_map<PlacementGroupID, std::shared_ptr<GcsPlacementGroup>>
registered_placement_groups_;

absl::btree_multimap<int64_t, std::pair<ExponentialBackoff, std::shared_ptr<GcsPlacementGroup>>> pending_placement_groups_;

std::deque<std::shared_ptr<GcsPlacementGroup>> infeasible_placement_groups_;
```

- Tracks placement group states (PENDING → PREPARED → CREATED → REMOVED)
#### Actor Manager
和placement group 一樣，actor 在GCS中也被拆分為scheduler & manager。
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

## GCS Client
GCS client提供了一系列的accessors來存取GCS server中的service。
### Accessor
-  ActorInfoAccessor
- JobInfoAccessor
- NodeInfoAccessor 
- NodeResourceInfoAccessor
- ErrorInfoAccessor
- TaskInfoAccessor
- WorkerInfoAccessor
- PlacementGroupInfoAccessor
- InternalKVAccessor
- RuntimeEnvAccessor
- AutoscalerStateAccessor
- PublisherAccessor
其中提供了數個同步及異步的methods，用來對特定資料進行CRUD以及subscribe特定資料的event。
e.g. JobInfoAccessor
```cpp
class JobInfoAccessor {
	virtual Status AsyncAdd(const std::shared_ptr<rpc::JobTableData> &data_ptr,
		const StatusCallback &callback);
	virtual Status AsyncMarkFinished(const JobID &job_id, const StatusCallback &callback);
	virtual Status AsyncSubscribeAll(
	    const SubscribeCallback<JobID, rpc::JobTableData> &subscribe,
	    const StatusCallback &done);
	virtual Status GetAll(const std::optional<std::string> &job_or_submission_id,
                        bool skip_submission_job_info_field,
                        bool skip_is_running_tasks_field,
                        std::vector<rpc::JobTableData> &job_data_list,
                        int64_t timeout_ms);
    virtual void AsyncResubscribe();
    virtual Status AsyncGetNextJobID(const ItemCallback<JobID> &callback);
```
也有提供簡單的cache。
e.g.
```cpp
class NodeInfoAccessor {
  /* Omitted */
  /// A cache for information about all nodes.
  absl::flat_hash_map<NodeID, rpc::GcsNodeInfo> node_cache_;
}
```
### GlobalStateAccessor
如同`src/ray/gcs/gcs_client/global_state_accessor.h` 的註解：
```txt
/// `GlobalStateAccessor` is used to provide synchronous interfaces to access data in GCS
/// for the language front-end (e.g., Python's `state.py`).
```
GlobalStateAccessor其實就是一個為了提供給其他語言使用的一個同步的interface，把上面accessors提供的異步method包成同步的版本，這樣可以在實作中有async的效率，又可以在使用上很簡單得去使用這些methods。

實作上利用C++的promise & future 來把異步轉為同步。
```cpp
std::vector<std::string> GlobalStateAccessor::GetAllJobInfo(...) {
  std::vector<std::string> job_table_data;
  std::promise<bool> promise;
  {
    absl::ReaderMutexLock lock(&mutex_);
    RAY_CHECK_OK(gcs_client_->Jobs().AsyncGetAll(
        /*...*/,
        TransformForMultiItemCallback<rpc::JobTableData>(job_table_data, promise),
        /*timeout_ms=*/-1));
  }
  promise.get_future().get();  // Block until async operation completes
  return job_table_data;
}
```
## Memory Store
GCS 支援兩種 storage backend：
- **In-Memory Storage**
- **Redis Storage**: 提供持久化和 fault tolerance

GCS 會根據config決定使用哪種 storage：
```cpp
GcsServer::StorageType GcsServer::GetStorageType() const {
  if (RayConfig::instance().gcs_storage() == kInMemoryStorage) {
    if (!config_.redis_address.empty()) {
      return StorageType::REDIS_PERSIST;
    }
    return StorageType::IN_MEMORY;
  }
  if (RayConfig::instance().gcs_storage() == kRedisStorage) {
    return StorageType::REDIS_PERSIST;
  }
  return StorageType::UNKNOWN;
}
```
## 結語
GCS 作為 Ray 的控制中心，在其中拆分成了 Manager + Table Store 的架構，並利用了許多async 的操作讓GCS能夠handle大量的request和事件，也提供了Redis作為其back store來達成fault toletance。

在接下來的文章中，將進一步探討 Ray Core 的components，包括 Core Worker、Raylet、Object Store等等。

--- 
## 後記
### 賺到的PR
在trace GCS code時，意外賺到了幾個PR。
推薦大家在讀code的時候可以想看看哪裡是可以improve的，然後發支PR。
> 俗話說：凡走過必留下PR。
![](attachments/Pasted%20image%2020250718021751.png)
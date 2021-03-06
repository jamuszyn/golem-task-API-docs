---
title: Golem Task API

language_tabs: # must be one of https://git.io/vQNgJ
  - protobuf

toc_footers:
  - <a href='https://docs.golem.network'>Main documentation</a>
  - <a href='https://golem.network'>Website</a>
  - <a href='https://chat.golem.network'>Chat</a>
  - <a href='https://twitter.com/golemproject'>Twitter</a>
  - <a href='https://www.reddit.com/r/GolemProject'>Reddit</a>
  - <a href='https://www.facebook.com/golemproject/'>Facebook</a>
  - <a href='https://blog.golem.network'>Blog</a>

search: true
---

# Task-api
### Golem - application communication interface

<aside class="notice">This work is in it's alpha stage and under heavy development so the interface may change frequently.</aside>

This repository contains the interface that the Golem compatible application should implement as well as constants used by the protocol. The interface and the constants are defined under the `golem_task_api/proto` directory in the [Protocol Buffers](https://developers.google.com/protocol-buffers/) files.

This repository also contains programming language specific packages of the `gRPC` protocol which may be used for concrete implementation. This is for the ease of development of the application but it's not required to use them in the application.
If you don't see a programming language you're interested in, feel free to create an issue or even a pull request and we will add it.


# The API

The API is divided into two independent parts - **requestor** and **provider**.

## Requestor
> Requestor

```proto
service RequestorApp {
  rpc CreateTask (CreateTaskRequest) returns (CreateTaskReply) {}
  rpc NextSubtask (NextSubtaskRequest) returns (NextSubtaskReply) {}
  rpc Verify (VerifyRequest) returns (VerifyReply) {}
  rpc DiscardSubtasks (DiscardSubtasksRequest) returns (DiscardSubtasksReply) {}
  rpc RunBenchmark (RunBenchmarkRequest) returns (RunBenchmarkReply) {}
  rpc HasPendingSubtasks (HasPendingSubtasksRequest) returns (HasPendingSubtasksReply) {}
  rpc AbortTask (AbortTaskRequest) returns (AbortTaskReply) {}
  rpc AbortSubtask (AbortSubtaskRequest) returns (AbortSubtaskReply) {}

  rpc Shutdown (ShutdownRequest) returns (ShutdownReply) {}
}
```

For requestor the app should implement a long running RPC service which implements the `RequestorApp` interface from the proto files. The app should assume it will have access to a single directory (let's call it `work_dir`). Each task will have its own separate working directory under the main `work_dir`. You can assume that for a given `task_id` the first call will always be `CreateTask` and the following directories will exist under `work_dir` and they will be empty:

- `{task_id}`
- `{task_id}/{constants.TASK_INPUTS_DIR}`
- `{task_id}/{constants.SUBTASK_INPUTS_DIR}`
- `{task_id}/{constants.TASK_OUTPUTS_DIR}`
- `{task_id}/{constants.SUBTASK_OUTPUTS_DIR}`

### RPC methods

### CreateTask

```proto
rpc CreateTask (CreateTaskRequest) returns (CreateTaskReply) {}

message CreateTaskRequest {
  string task_id = 1;
  int32 max_subtasks_count = 2;
  string task_params_json = 3;
}

message CreateTaskReply {
  string env_id = 1;
  string prerequisites_json = 2;
  Infrastructure inf_requirements = 3;
}

message Infrastructure {
  float min_memory_mib = 1;
}
```
  - Takes three arguments: `task_id`, `max_subtasks_count`, and `task_params_json`.
  - Should treat `{work_dir}/{task_id}` as the working directory for the given task.
  - `task_params_json` is a JSON string containing app-specific task parameters. Format of these parameters is entirely up to the application developer.
  - Will only be called once with given `task_id`.
  - Can assume that `task_id` is unique per node.
  - Can assume `{task_id}/{constants.TASK_INPUTS_DIR}` contains all the resources provided by task creator.
  - Returns `env_id` and `prerequisites_json` specifying the environment and prerequisites required for providers to compute the task. See environments section for details.



### NextSubtask

```proto
rpc NextSubtask (NextSubtaskRequest) returns (NextSubtaskReply) {}

message NextSubtaskRequest {
  string task_id = 1;
  string subtask_id = 2;
  string opaque_node_id = 3;
}

message NextSubtaskReply {
  // workaround for the lack of null values in proto3
  // where "one of" means "at most one of"
  oneof subtask_oneof {
      SubtaskReply subtask = 1;
  }
}

message SubtaskReply {
  string subtask_params_json = 1;
  repeated string resources = 2;
}
```
  - Takes three arguments: `task_id`, `subtask_id` and `opaque_node_id`.
  - `opaque_node_id` is an identifier of the node which is going to compute the requested subtask. 'Opaque' means that the identifier doesn't allow to obtain any further information about the node (e.g. public key, IP address).
  - Can assume that `subtask_id` is unique per node.
  - Can assume `CreateTask` was called earlier with the same `task_id`.
  - Can return an empty message meaning that the app refuses to assign a subtask to the provider node (for whatever reason).
  - Returns `subtask_params_json` which is the JSON string containing subtask specific parameters.
  - Also returns `resources` which is a list of names of files required for computing the subtask. Files with these names are required to be present in `{task_id}/{constants.SUBTASK_INPUTS_DIR}` directory.



### Verify

```proto
rpc Verify (VerifyRequest) returns (VerifyReply) {}

message VerifyRequest {
  string task_id = 1;
  string subtask_id = 2;
}

message VerifyReply {
  enum VerifyResult {
    SUCCESS = 0;
    FAILURE = 1;
    AWAITING_DATA = 2;
    INCONCLUSIVE = 3;
  }
  VerifyResult result = 1;
  string reason = 2;
}
```  
  - Takes two arguments: `task_id` and `subtask_id` which specify which subtask results should be verified.
  - Will be called with only valid `task_id` and `subtask_id` values.
  - Returns `result` which is one of the defined verification result statuses:
    - `SUCCESS` - the subtask was computed correctly,
    - `FAILURE` - the subtask was computed incorrectly,
    - `INCONCLUSIVE` - cannot determine whether the subtask was computed correctly,
    - `AWAITING_DATA` - cannot perform verificaton until results of other subtasks are available.
  - Also returns `reason` which is a string providing more detail about the result.
  - For successfully verified subtasks it can also perform merging of the partial results into the final one.



### DiscardSubtasks

```proto
rpc DiscardSubtasks (DiscardSubtasksRequest) returns (DiscardSubtasksReply) {}

message DiscardSubtasksRequest {
  string task_id = 1;
  repeated string subtask_ids = 2;
}

message DiscardSubtasksReply {
  repeated string discarded_subtask_ids = 1;
}
```  
  - Takes two arguments: `task_id` and `subtask_ids`.
  - Should discard results of given subtasks and any dependent subtasks.
  - Returns list of subtask IDs that have been discarded.
  - In a simple case where subtasks are independent from each other it will return the same list as it received.


### Benchmark

```proto
rpc RunBenchmark (RunBenchmarkRequest) returns (RunBenchmarkReply) {}

message RunBenchmarkRequest {
}

message RunBenchmarkReply {
  float score = 1;
}
``` 
 - Takes no arguments.
  - Returns a score which indicates how efficient the machine is for this type of tasks.
  - Shouldn't take much time (preferably less than a minute for medium range machines).




### HasPendingSubtasks

```proto
rpc HasPendingSubtasks (HasPendingSubtasksRequest) returns (HasPendingSubtasksReply) {}

message HasPendingSubtasksRequest {
  string task_id = 1;
}

message HasPendingSubtasksReply {
  bool has_pending_subtasks = 1;
}
```
  - Takes one argument `task_id`.
  - Returns a boolean indicating whether there are any more pending subtasks waiting for computation at given moment.
  - In case when it returns `true`, the next `NextSubtask` call should return successfully (although it can still return an empty message).



### AbortTask

```proto
rpc AbortTask (AbortTaskRequest) returns (AbortTaskReply) {}

message AbortTaskRequest {
  string task_id = 1;
}

message AbortTaskReply {
}
```
  - Takes one argument: `task_id`.
  - Will be called when the task is aborted by the user or timed out. Should stop all running subtask verifications for this task and perform any other necessary cleanup.



### AbortSubtask

```proto
rpc AbortSubtask (AbortSubtaskRequest) returns (AbortSubtaskReply) {}

message AbortSubtaskRequest {
  string task_id = 1;
  string subtask_id = 2;
}

message AbortSubtaskReply {
}

```
  - Takes two arguments: `task_id` and `subtask_id`.
  - Will be called when the subtask is aborted by the user or timed out. Should stop verification of the subtask (if it's running) and perform any other necessary cleanup.



### Shutdown

```proto
rpc Shutdown (ShutdownRequest) returns (ShutdownReply) {}

mmessage ShutdownRequest {
}

message ShutdownReply {
}
``` 
  - takes no arguments
  - should gracefully terminate the service

When the last subtask is successfully verified on the requestor's side, the `work_dir/task_id/constants.RESULTS` directory should contain all result files and nothing else.

---

### Environments
Both provider and requestor apps run on top of Golem's execution environments. Environment for requestor is specified in the application definition and cannot vary. Provider environment is specified by the return value of `CreateTask` call. A single application could use different environments for different types of tasks, it could also use different environment for requestor and provider. Environments have their unique IDs and prerequisites formats. Prerequisites are additional requirements for the environment to run the app (e.g. Docker environment prerequisites specify image). Environment IDs and prerequisites formats are listed in [envs.proto](https://github.com/golemfactory/task-api/blob/master/golem_task_api/proto/envs.proto) file.

Currently the following environments are supported:
- `docker_cpu` - standard Docker environment  
  Prerequisites format:
  ```json
  {
      "image": "...",
      "tag":   "..."
  }
  ```
- `docker_gpu` - GPU-enabled Docker environment, Linux only  
  Prerequisites format: same as `docker_cpu`

---

## Provider

```proto
service ProviderApp {
  rpc Compute (ComputeRequest) returns (ComputeReply) {}
  rpc RunBenchmark (RunBenchmarkRequest) returns (RunBenchmarkReply) {}

  rpc Shutdown (ShutdownRequest) returns (ShutdownReply) {}
}

```
Provider app should implement a short-lived RPC service which implements the `ProviderApp` interface from the proto files. Short-lived means that there will be only one request issued per service instance, i.e. the service should shutdown automatically after handling the first and only request.


### RPC commands

### Compute

```proto
rpc Compute (ComputeRequest) returns (ComputeReply) {}

message ComputeRequest {
  string task_id = 1;
  string subtask_id = 2;
  string subtask_params_json = 3;
}

message ComputeReply {
  string output_filepath = 1;
}
```
`Compute`
  - Gets a single working directory `task_work_dir` to operate on.
  - Different subtasks of the same task will have the same `task_work_dir`.
  - Takes `task_id`, `subtask_id`, `subtask_params_json` as arguments.
  - Can assume the `{task_work_dir}/{constants.SUBTASK_INPUTS_DIR}` directory exists.
  - Can assume that under `{task_work_dir}/{constants.SUBTASK_INPUTS_DIR}` are the resources specified in the corresponding

### NextSubtask call.

```proto

```
  - Returns a filepath (relative to the `task_work_dir`) of the result file which will be sent back to the requestor with unchanged name.

### Benchmark

```proto
rpc RunBenchmark (RunBenchmarkRequest) returns (RunBenchmarkReply) {}

message RunBenchmarkRequest {
}

message RunBenchmarkReply {
  float score = 1;
}
```
  - Takes no arguments.
  - Returns a score which indicates how efficient the machine is for this type of tasks.
  - Shouldn't take much time (preferably less than a minute for medium range machines).

### Shutdown

```proto
rpc Shutdown (ShutdownRequest) returns (ShutdownReply) {}

mmessage ShutdownRequest {
}

message ShutdownReply {
}
```
- Takes no arguments.
  - Should gracefully terminate the service.
  - Can be called in case the provider wants to interrupt task computation or benchmark.






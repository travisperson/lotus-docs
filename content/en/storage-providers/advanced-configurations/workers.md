---
title: "Workers"
description: "Lotus has a lot of advanced configurations you can tune to optimize your storage provider setup. This guide explains the advanced configuration options for seal workers and PoSt workers"
lead: "Lotus has a lot of advanced configurations you can tune to optimize your storage provider setup. This guide explains the advanced configuration options for seal workers and PoSt workers"
draft: false
menu:
    storage-providers:
        parent: "storage-providers-advanced-configurations"
weight: 505
toc: true
---

While the Lotus workers have very reasonable default settings, some storage providers might want to fine tune some advanced configurations according to their setup.

## Advanced seal worker configurations

Although the default settings for the seal workers are reasonable, you can configure some advanced settings when running the workers. These settings should be tested for local optimizations of your hardware.

### Resource allocation in seal workers

Each **Seal Worker** can potentially run multiple tasks in available slots. Each slot is called a _window_. The number of available windows per worker is determined by the requirements of the sealing tasks being allocated to the worker and its available system resources, including:

- Number of CPU threads the task will use.
- Minimum amount of RAM required for good performance.
- Maximum amount of RAM required to run the task, where the system can swap out part of the RAM to disk, and the performance won't be affected too much.
- Whether the system has a GPU.

### Task resource table

The default resource table lives in [resources.go](https://github.com/filecoin-project/lotus/blob/master/extern/sector-storage/storiface/resources.go) and can be edited to tune the scheduled behavior to fit specific sealing clusters better.

Here is the default resource value table. There values are conservative:

| Sector size | Task Type  | Threads | Min RAM | Min disk space| GPU     |
|-------------|------------|---------|---------|------------|------------|
|     32G     | AddPiece   | 92%    | 4G      | 4G         |            |
|             | PreCommit1 | 1*     | 56G     | 64G        |            |
|             | PreCommit2 | 92%**  | 15G     | 15G        | If Present |
|             | Commit1    | 0***   | 1G      | 1G         |            |
|             | Commit2    | 92%**  | 32G+30G | 32G+150G   | If Present |
|     64G     | AddPiece   | 92%    | 8G      | 8G         |            |
|             | PreCommit1 | 1*     | 112G    | 128G       |            |
|             | PreCommit2 | 92%**  | 30G     | 30G        | If Present |
|             | Commit1    | 0***   | 1G      | 1G         |            |
|             | Commit2    | 92%**  | 64G+60G | 64G+190G   | If Present |

\* When used with the `FIL_PROOFS_USE_MULTICORE_SDR=1` environment variable, `PreCommit1` can use multiple cores, up to the number of cores sharing L3 caches.
\** Depending on the number of available threads, this value means:

```plaintext
 12  * 0.92 = 11
 16  * 0.92 = 14
 24  * 0.92 = 22
 32  * 0.92 = 29
 64  * 0.92 = 58
 128 * 0.92 = 117
```

\*** The Commit1 step is very cheap in terms of CPU time and blocks the Commit2 step. Allocating it to zero threads makes it more likely it will be scheduled with higher priority.

The Unseal task has the same resource use as the PreCommit1 task.

{{< alert icon="info" >}}
The default and custom configurations in the task resource table can be overridden using environment variables on a per worker basis. See the [environment variables]({{< relref "#control-groups" >}}) section for information. You can also gather these details using the `lotus-worker resources --default` command.
{{< /alert >}}

### Resource windows

The scheduler uses the concept of resource windows to prevent _resource starvation_ of tasks requiring larger amounts of resources by tasks with smaller resource requirements.

A resource window is simply a bucket of sealing tasks that can be run by a given worker in parallel based on the resources the worker has available when no tasks are running.

In the scheduler, each worker has:

- Scheduling windows - Two resource windows used to assign tasks to execute from the global queue.
- Preparing window - One resource window in which tasks are prepared to execute. For example, sector data is fetched if needed.
- Executing window - One resource window for currently executing tasks.

When tasks arrive in the global scheduling queue, the scheduler will look for empty scheduling windows. The scheduler may assign tasks to the scheduling window based on several factors, such as: 

- whether the worker already has direct access to sector data.
- task types supported by the worker.
- whether the worker has disk space for sector data.
- task priority.

After the scheduler fills the scheduling window, the schedule is sent to the worker for processing. The worker will pull tasks out of the scheduling window and start preparing them in the preparing window. After the preparing step is done, the task will be executed in the executing window.

After the worker has fully processed a scheduling window, it's sent back to the global scheduler to get more sealing tasks.

### Task priorities

When the scheduler decides which tasks to run, it takes into account the priority of running a specific task.

There are two priority tiers - high priority, for tasks that are cheap to execute but block other actions, and normal priority for all other tasks. Default priorities are defined in the table below. The lower the number, the higher the priority. Negative numbers have the highest priority. So `-2` has a higher priority than `1`.

| Task Type           | Priority |
|---------------------|----------|
| RegenSectorKey      | 10       |
| AddPiece            | 9        |
| ReplicaUpdate       | 8        |
| ProveReplicaUpdate2 | 7        |
| ProveReplicaUpdate1 | 6        |
| PreCommit1          | 5        |
| PreCommit2          | 4        |
| Commit2             | 3        |
| Commit1             | 2        |
| Unseal              | 1        |
| Fetch               | -1       |
| Finalize            | -2       |


When comparing task priority:

- High priority tasks are considered first.
- Sectors with deals are considered second. Additional deals increase the priority.
- If the above is equal, tasks are selected based on priorities in the table.
- If the above is equal, sectors with lower sector numbers are selected (this can optimize gas usage slightly when submitting messages to the chain).

### Control groups

Control groups (cgroups) is a Linux Kernel feature that limits, accounts for, and isolates the resource usage of a collection of processes. If cgroups are in use on the host, the `lotus-worker` will honor the cgroup memory limits configured on the host. There are 6 cgroup variables for each sealing stage.

| Cgroup variable    |  Type    |  Explanation                                                                                                |
|--------------------|----------|-------------------------------------------------------------------------------------------------------------|
|  MinMemory         | uint64   | Minimum RAM used for decent performance.                                                                     |
|  MaxMemory         | uint64   | Maximum memory (swap+RAM) usage during task execution.                                                        |
|  GPUUtilization    | float64  | Number of GPUs a task can use.                                                                              |
|  MaxParallelism    | int      | Number of CPU cores a task can use when GPU is not in use.                                                   |
|  MaxParallelismGPU | int      | Number of CPU cores a task can use when GPU is in use. Inherits value from `MaxParallelism` when set to 0.  |
|  BaseMinMemory     | uint64   | Minimum RAM used for decent performance. This is shared between the treads, unlike `MinMemory`. |

These are the default variables for resource allocation tuning. These values override settings in the resource allocation table.

{{< details "32 GB environment" >}}
```plaintext
AP_32G_BASE_MIN_MEMORY=1073741824
AP_32G_GPU_UTILIZATION=0
AP_32G_MAX_CONCURRENT=0
AP_32G_MAX_MEMORY=4294967296
AP_32G_MAX_PARALLELISM=1
AP_32G_MAX_PARALLELISM_GPU=0
AP_32G_MIN_MEMORY=4294967296
C1_32G_BASE_MIN_MEMORY=1073741824
C1_32G_GPU_UTILIZATION=0
C1_32G_MAX_CONCURRENT=0
C1_32G_MAX_MEMORY=1073741824
C1_32G_MAX_PARALLELISM=0
C1_32G_MAX_PARALLELISM_GPU=0
C1_32G_MIN_MEMORY=1073741824
C2_32G_BASE_MIN_MEMORY=34359738368
C2_32G_GPU_UTILIZATION=1
C2_32G_MAX_CONCURRENT=0
C2_32G_MAX_MEMORY=161061273600
C2_32G_MAX_PARALLELISM=-1
C2_32G_MAX_PARALLELISM_GPU=6
C2_32G_MIN_MEMORY=32212254720
DC_32G_BASE_MIN_MEMORY=1073741824
DC_32G_GPU_UTILIZATION=0
DC_32G_MAX_CONCURRENT=0
DC_32G_MAX_MEMORY=4294967296
DC_32G_MAX_PARALLELISM=1
DC_32G_MAX_PARALLELISM_GPU=0
DC_32G_MIN_MEMORY=4294967296
GET_32G_BASE_MIN_MEMORY=0
GET_32G_GPU_UTILIZATION=0
GET_32G_MAX_CONCURRENT=0
GET_32G_MAX_MEMORY=1048576
GET_32G_MAX_PARALLELISM=0
GET_32G_MAX_PARALLELISM_GPU=0
GET_32G_MIN_MEMORY=1048576
GSK_32G_BASE_MIN_MEMORY=1073741824
GSK_32G_GPU_UTILIZATION=0
GSK_32G_MAX_CONCURRENT=0
GSK_32G_MAX_MEMORY=4294967296
GSK_32G_MAX_PARALLELISM=1
GSK_32G_MAX_PARALLELISM_GPU=0
GSK_32G_MIN_MEMORY=4294967296
PC1_32G_BASE_MIN_MEMORY=10485760
PC1_32G_GPU_UTILIZATION=0
PC1_32G_MAX_CONCURRENT=0
PC1_32G_MAX_MEMORY=68719476736
PC1_32G_MAX_PARALLELISM=1
PC1_32G_MAX_PARALLELISM_GPU=0
PC1_32G_MIN_MEMORY=60129542144
PC2_32G_BASE_MIN_MEMORY=1073741824
PC2_32G_GPU_UTILIZATION=1
PC2_32G_MAX_CONCURRENT=0
PC2_32G_MAX_MEMORY=16106127360
PC2_32G_MAX_PARALLELISM=-1
PC2_32G_MAX_PARALLELISM_GPU=6
PC2_32G_MIN_MEMORY=16106127360
PC2_64G_MAX_MEMORY=32212254720
PC2_64G_MIN_MEMORY=32212254720
PR1_32G_BASE_MIN_MEMORY=1073741824
PR1_32G_GPU_UTILIZATION=0
PR1_32G_MAX_CONCURRENT=0
PR1_32G_MAX_MEMORY=1073741824
PR1_32G_MAX_PARALLELISM=0
PR1_32G_MAX_PARALLELISM_GPU=0
PR1_32G_MIN_MEMORY=1073741824
PR2_32G_BASE_MIN_MEMORY=34359738368
PR2_32G_GPU_UTILIZATION=1
PR2_32G_MAX_CONCURRENT=0
PR2_32G_MAX_MEMORY=161061273600
PR2_32G_MAX_PARALLELISM=-1
PR2_32G_MAX_PARALLELISM_GPU=6
PR2_32G_MIN_MEMORY=32212254720
RU_32G_BASE_MIN_MEMORY=1073741824
RU_32G_GPU_UTILIZATION=0
RU_32G_MAX_CONCURRENT=0
RU_32G_MAX_MEMORY=4294967296
RU_32G_MAX_PARALLELISM=1
RU_32G_MAX_PARALLELISM_GPU=0
RU_32G_MIN_MEMORY=4294967296
UNS_32G_BASE_MIN_MEMORY=10485760
UNS_32G_GPU_UTILIZATION=0
UNS_32G_MAX_CONCURRENT=0
UNS_32G_MAX_MEMORY=68719476736
UNS_32G_MAX_PARALLELISM=1
UNS_32G_MAX_PARALLELISM_GPU=0
UNS_32G_MIN_MEMORY=60129542144
WDP_32G_BASE_MIN_MEMORY=34359738368
WDP_32G_GPU_UTILIZATION=1
WDP_32G_MAX_CONCURRENT=0
WDP_32G_MAX_MEMORY=103079215104
WDP_32G_MAX_PARALLELISM=-1
WDP_32G_MAX_PARALLELISM_GPU=6
WDP_32G_MIN_MEMORY=32212254720
WNP_32G_BASE_MIN_MEMORY=34359738368
WNP_32G_GPU_UTILIZATION=1
WNP_32G_MAX_CONCURRENT=0
WNP_32G_MAX_MEMORY=1073741824
WNP_32G_MAX_PARALLELISM=-1
WNP_32G_MAX_PARALLELISM_GPU=6
WNP_32G_MIN_MEMORY=1073741824
```
{{< /details >}}

{{< details "512 MB Environment" >}}
```plaintext
AP_512M_BASE_MIN_MEMORY=1073741824
AP_512M_GPU_UTILIZATION=0
AP_512M_MAX_CONCURRENT=0
AP_512M_MAX_MEMORY=1073741824
AP_512M_MAX_PARALLELISM=1
AP_512M_MAX_PARALLELISM_GPU=0
AP_512M_MIN_MEMORY=1073741824
C1_512M_BASE_MIN_MEMORY=1073741824
C1_512M_GPU_UTILIZATION=0
C1_512M_MAX_CONCURRENT=0
C1_512M_MAX_MEMORY=1073741824
C1_512M_MAX_PARALLELISM=0
C1_512M_MAX_PARALLELISM_GPU=0
C1_512M_MIN_MEMORY=1073741824
C2_512M_BASE_MIN_MEMORY=10737418240
C2_512M_GPU_UTILIZATION=1
C2_512M_MAX_CONCURRENT=0
C2_512M_MAX_MEMORY=1610612736
C2_512M_MAX_PARALLELISM=1
C2_512M_MAX_PARALLELISM_GPU=0
C2_512M_MIN_MEMORY=1073741824
DC_512M_BASE_MIN_MEMORY=1073741824
DC_512M_GPU_UTILIZATION=0
DC_512M_MAX_CONCURRENT=0
DC_512M_MAX_MEMORY=1073741824
DC_512M_MAX_PARALLELISM=1
DC_512M_MAX_PARALLELISM_GPU=0
DC_512M_MIN_MEMORY=1073741824
GET_512M_BASE_MIN_MEMORY=0
GET_512M_GPU_UTILIZATION=0
GET_512M_MAX_CONCURRENT=0
GET_512M_MAX_MEMORY=1048576
GET_512M_MAX_PARALLELISM=0
GET_512M_MAX_PARALLELISM_GPU=0
GET_512M_MIN_MEMORY=1048576
GSK_512M_BASE_MIN_MEMORY=1073741824
GSK_512M_GPU_UTILIZATION=0
GSK_512M_MAX_CONCURRENT=0
GSK_512M_MAX_MEMORY=1073741824
GSK_512M_MAX_PARALLELISM=1
GSK_512M_MAX_PARALLELISM_GPU=0
GSK_512M_MIN_MEMORY=1073741824
PC1_512M_BASE_MIN_MEMORY=1048576
PC1_512M_GPU_UTILIZATION=0
PC1_512M_MAX_CONCURRENT=0
PC1_512M_MAX_MEMORY=1073741824
PC1_512M_MAX_PARALLELISM=1
PC1_512M_MAX_PARALLELISM_GPU=0
PC1_512M_MIN_MEMORY=805306368
PC2_512M_BASE_MIN_MEMORY=1073741824
PC2_512M_GPU_UTILIZATION=0
PC2_512M_MAX_CONCURRENT=0
PC2_512M_MAX_MEMORY=1610612736
PC2_512M_MAX_PARALLELISM=-1
PC2_512M_MAX_PARALLELISM_GPU=0
PC2_512M_MIN_MEMORY=1073741824
PR1_512M_BASE_MIN_MEMORY=1073741824
PR1_512M_GPU_UTILIZATION=0
PR1_512M_MAX_CONCURRENT=0
PR1_512M_MAX_MEMORY=1073741824
PR1_512M_MAX_PARALLELISM=0
PR1_512M_MAX_PARALLELISM_GPU=0
PR1_512M_MIN_MEMORY=1073741824
PR2_512M_BASE_MIN_MEMORY=10737418240
PR2_512M_GPU_UTILIZATION=1
PR2_512M_MAX_CONCURRENT=0
PR2_512M_MAX_MEMORY=1610612736
PR2_512M_MAX_PARALLELISM=1
PR2_512M_MAX_PARALLELISM_GPU=0
PR2_512M_MIN_MEMORY=1073741824
RU_512M_BASE_MIN_MEMORY=1073741824
RU_512M_GPU_UTILIZATION=0
RU_512M_MAX_CONCURRENT=0
RU_512M_MAX_MEMORY=1073741824
RU_512M_MAX_PARALLELISM=1
RU_512M_MAX_PARALLELISM_GPU=0
RU_512M_MIN_MEMORY=1073741824
UNS_512M_BASE_MIN_MEMORY=1048576
UNS_512M_GPU_UTILIZATION=0
UNS_512M_MAX_CONCURRENT=0
UNS_512M_MAX_MEMORY=1073741824
UNS_512M_MAX_PARALLELISM=1
UNS_512M_MAX_PARALLELISM_GPU=0
UNS_512M_MIN_MEMORY=805306368
WDP_512M_BASE_MIN_MEMORY=10737418240
WDP_512M_GPU_UTILIZATION=1
WDP_512M_MAX_CONCURRENT=0
WDP_512M_MAX_MEMORY=1610612736
WDP_512M_MAX_PARALLELISM=1
WDP_512M_MAX_PARALLELISM_GPU=0
WDP_512M_MIN_MEMORY=1073741824
WNP_512M_BASE_MIN_MEMORY=10737418240
WNP_512M_GPU_UTILIZATION=1
WNP_512M_MAX_CONCURRENT=0
WNP_512M_MAX_MEMORY=2048
WNP_512M_MAX_PARALLELISM=1
WNP_512M_MAX_PARALLELISM_GPU=0
WNP_512M_MIN_MEMORY=2048
```
{{< /details >}}

{{< details "64 GB Environment" >}}
```plaintext
AP_64G_BASE_MIN_MEMORY=1073741824
AP_64G_GPU_UTILIZATION=0
AP_64G_MAX_CONCURRENT=0
AP_64G_MAX_MEMORY=8589934592
AP_64G_MAX_PARALLELISM=1
AP_64G_MAX_PARALLELISM_GPU=0
AP_64G_MIN_MEMORY=8589934592
C1_64G_BASE_MIN_MEMORY=1073741824
C1_64G_GPU_UTILIZATION=0
C1_64G_MAX_CONCURRENT=0
C1_64G_MAX_MEMORY=1073741824
C1_64G_MAX_PARALLELISM=0
C1_64G_MAX_PARALLELISM_GPU=0
C1_64G_MIN_MEMORY=1073741824
C2_64G_BASE_MIN_MEMORY=68719476736
C2_64G_GPU_UTILIZATION=1
C2_64G_MAX_CONCURRENT=0
C2_64G_MAX_MEMORY=204010946560
C2_64G_MAX_PARALLELISM=-1
C2_64G_MAX_PARALLELISM_GPU=6
C2_64G_MIN_MEMORY=64424509440
DC_64G_BASE_MIN_MEMORY=1073741824
DC_64G_GPU_UTILIZATION=0
DC_64G_MAX_CONCURRENT=0
DC_64G_MAX_MEMORY=8589934592
DC_64G_MAX_PARALLELISM=1
DC_64G_MAX_PARALLELISM_GPU=0
DC_64G_MIN_MEMORY=8589934592
GET_64G_BASE_MIN_MEMORY=0
GET_64G_GPU_UTILIZATION=0
GET_64G_MAX_CONCURRENT=0
GET_64G_MAX_MEMORY=1048576
GET_64G_MAX_PARALLELISM=0
GET_64G_MAX_PARALLELISM_GPU=0
GET_64G_MIN_MEMORY=1048576
GSK_64G_BASE_MIN_MEMORY=1073741824
GSK_64G_GPU_UTILIZATION=0
GSK_64G_MAX_CONCURRENT=0
GSK_64G_MAX_MEMORY=8589934592
GSK_64G_MAX_PARALLELISM=1
GSK_64G_MAX_PARALLELISM_GPU=0
GSK_64G_MIN_MEMORY=8589934592
PC1_64G_BASE_MIN_MEMORY=10485760
PC1_64G_GPU_UTILIZATION=0
PC1_64G_MAX_CONCURRENT=0
PC1_64G_MAX_MEMORY=137438953472
PC1_64G_MAX_PARALLELISM=1
PC1_64G_MAX_PARALLELISM_GPU=0
PC1_64G_MIN_MEMORY=120259084288
PC2_64G_BASE_MIN_MEMORY=1073741824
PC2_64G_GPU_UTILIZATION=1
PC2_64G_MAX_CONCURRENT=0
PC2_64G_MAX_MEMORY=32212254720
PC2_64G_MAX_PARALLELISM=-1
PC2_64G_MAX_PARALLELISM_GPU=6
PC2_64G_MIN_MEMORY=32212254720
PR1_64G_BASE_MIN_MEMORY=1073741824
PR1_64G_GPU_UTILIZATION=0
PR1_64G_MAX_CONCURRENT=0
PR1_64G_MAX_MEMORY=1073741824
PR1_64G_MAX_PARALLELISM=0
PR1_64G_MAX_PARALLELISM_GPU=0
PR1_64G_MIN_MEMORY=1073741824
PR2_64G_BASE_MIN_MEMORY=68719476736
PR2_64G_GPU_UTILIZATION=1
PR2_64G_MAX_CONCURRENT=0
PR2_64G_MAX_MEMORY=204010946560
PR2_64G_MAX_PARALLELISM=-1
PR2_64G_MAX_PARALLELISM_GPU=6
PR2_64G_MIN_MEMORY=64424509440
RU_64G_BASE_MIN_MEMORY=1073741824
RU_64G_GPU_UTILIZATION=0
RU_64G_MAX_CONCURRENT=0
RU_64G_MAX_MEMORY=8589934592
RU_64G_MAX_PARALLELISM=1
RU_64G_MAX_PARALLELISM_GPU=0
RU_64G_MIN_MEMORY=8589934592
UNS_64G_BASE_MIN_MEMORY=10485760
UNS_64G_GPU_UTILIZATION=0
UNS_64G_MAX_CONCURRENT=0
UNS_64G_MAX_MEMORY=137438953472
UNS_64G_MAX_PARALLELISM=1
UNS_64G_MAX_PARALLELISM_GPU=0
UNS_64G_MIN_MEMORY=120259084288
WDP_64G_BASE_MIN_MEMORY=68719476736
WDP_64G_GPU_UTILIZATION=1
WDP_64G_MAX_CONCURRENT=0
WDP_64G_MAX_MEMORY=128849018880
WDP_64G_MAX_PARALLELISM=-1
WDP_64G_MAX_PARALLELISM_GPU=6
WDP_64G_MIN_MEMORY=64424509440
WNP_64G_BASE_MIN_MEMORY=68719476736
WNP_64G_GPU_UTILIZATION=1
WNP_64G_MAX_CONCURRENT=0
WNP_64G_MAX_MEMORY=1073741824
WNP_64G_MAX_PARALLELISM=-1
WNP_64G_MAX_PARALLELISM_GPU=6
WNP_64G_MIN_MEMORY=1073741824
```
{{< /details >}}

{{< details "All Environment variables" >}}
```plaintext
AP_2K_BASE_MIN_MEMORY=2048
AP_2K_GPU_UTILIZATION=0
AP_2K_MAX_CONCURRENT=0
AP_2K_MAX_MEMORY=2048
AP_2K_MAX_PARALLELISM=1
AP_2K_MAX_PARALLELISM_GPU=0
AP_2K_MIN_MEMORY=2048
AP_32G_BASE_MIN_MEMORY=1073741824
AP_32G_GPU_UTILIZATION=0
AP_32G_MAX_CONCURRENT=0
AP_32G_MAX_MEMORY=4294967296
AP_32G_MAX_PARALLELISM=1
AP_32G_MAX_PARALLELISM_GPU=0
AP_32G_MIN_MEMORY=4294967296
AP_512M_BASE_MIN_MEMORY=1073741824
AP_512M_GPU_UTILIZATION=0
AP_512M_MAX_CONCURRENT=0
AP_512M_MAX_MEMORY=1073741824
AP_512M_MAX_PARALLELISM=1
AP_512M_MAX_PARALLELISM_GPU=0
AP_512M_MIN_MEMORY=1073741824
AP_64G_BASE_MIN_MEMORY=1073741824
AP_64G_GPU_UTILIZATION=0
AP_64G_MAX_CONCURRENT=0
AP_64G_MAX_MEMORY=8589934592
AP_64G_MAX_PARALLELISM=1
AP_64G_MAX_PARALLELISM_GPU=0
AP_64G_MIN_MEMORY=8589934592
AP_8M_BASE_MIN_MEMORY=8388608
AP_8M_GPU_UTILIZATION=0
AP_8M_MAX_CONCURRENT=0
AP_8M_MAX_MEMORY=8388608
AP_8M_MAX_PARALLELISM=1
AP_8M_MAX_PARALLELISM_GPU=0
AP_8M_MIN_MEMORY=8388608
C1_2K_BASE_MIN_MEMORY=2048
C1_2K_GPU_UTILIZATION=0
C1_2K_MAX_CONCURRENT=0
C1_2K_MAX_MEMORY=2048
C1_2K_MAX_PARALLELISM=0
C1_2K_MAX_PARALLELISM_GPU=0
C1_2K_MIN_MEMORY=2048
C1_32G_BASE_MIN_MEMORY=1073741824
C1_32G_GPU_UTILIZATION=0
C1_32G_MAX_CONCURRENT=0
C1_32G_MAX_MEMORY=1073741824
C1_32G_MAX_PARALLELISM=0
C1_32G_MAX_PARALLELISM_GPU=0
C1_32G_MIN_MEMORY=1073741824
C1_512M_BASE_MIN_MEMORY=1073741824
C1_512M_GPU_UTILIZATION=0
C1_512M_MAX_CONCURRENT=0
C1_512M_MAX_MEMORY=1073741824
C1_512M_MAX_PARALLELISM=0
C1_512M_MAX_PARALLELISM_GPU=0
C1_512M_MIN_MEMORY=1073741824
C1_64G_BASE_MIN_MEMORY=1073741824
C1_64G_GPU_UTILIZATION=0
C1_64G_MAX_CONCURRENT=0
C1_64G_MAX_MEMORY=1073741824
C1_64G_MAX_PARALLELISM=0
C1_64G_MAX_PARALLELISM_GPU=0
C1_64G_MIN_MEMORY=1073741824
C1_8M_BASE_MIN_MEMORY=8388608
C1_8M_GPU_UTILIZATION=0
C1_8M_MAX_CONCURRENT=0
C1_8M_MAX_MEMORY=8388608
C1_8M_MAX_PARALLELISM=0
C1_8M_MAX_PARALLELISM_GPU=0
C1_8M_MIN_MEMORY=8388608
C2_2K_BASE_MIN_MEMORY=2048
C2_2K_GPU_UTILIZATION=1
C2_2K_MAX_CONCURRENT=0
C2_2K_MAX_MEMORY=2048
C2_2K_MAX_PARALLELISM=1
C2_2K_MAX_PARALLELISM_GPU=0
C2_2K_MIN_MEMORY=2048
C2_32G_BASE_MIN_MEMORY=34359738368
C2_32G_GPU_UTILIZATION=1
C2_32G_MAX_CONCURRENT=0
C2_32G_MAX_MEMORY=161061273600
C2_32G_MAX_PARALLELISM=-1
C2_32G_MAX_PARALLELISM_GPU=6
C2_32G_MIN_MEMORY=32212254720
C2_512M_BASE_MIN_MEMORY=10737418240
C2_512M_GPU_UTILIZATION=1
C2_512M_MAX_CONCURRENT=0
C2_512M_MAX_MEMORY=1610612736
C2_512M_MAX_PARALLELISM=1
C2_512M_MAX_PARALLELISM_GPU=0
C2_512M_MIN_MEMORY=1073741824
C2_64G_BASE_MIN_MEMORY=68719476736
C2_64G_GPU_UTILIZATION=1
C2_64G_MAX_CONCURRENT=0
C2_64G_MAX_MEMORY=204010946560
C2_64G_MAX_PARALLELISM=-1
C2_64G_MAX_PARALLELISM_GPU=6
C2_64G_MIN_MEMORY=64424509440
C2_8M_BASE_MIN_MEMORY=8388608
C2_8M_GPU_UTILIZATION=1
C2_8M_MAX_CONCURRENT=0
C2_8M_MAX_MEMORY=8388608
C2_8M_MAX_PARALLELISM=1
C2_8M_MAX_PARALLELISM_GPU=0
C2_8M_MIN_MEMORY=8388608
DC_2K_BASE_MIN_MEMORY=2048
DC_2K_GPU_UTILIZATION=0
DC_2K_MAX_CONCURRENT=0
DC_2K_MAX_MEMORY=2048
DC_2K_MAX_PARALLELISM=1
DC_2K_MAX_PARALLELISM_GPU=0
DC_2K_MIN_MEMORY=2048
DC_32G_BASE_MIN_MEMORY=1073741824
DC_32G_GPU_UTILIZATION=0
DC_32G_MAX_CONCURRENT=0
DC_32G_MAX_MEMORY=4294967296
DC_32G_MAX_PARALLELISM=1
DC_32G_MAX_PARALLELISM_GPU=0
DC_32G_MIN_MEMORY=4294967296
DC_512M_BASE_MIN_MEMORY=1073741824
DC_512M_GPU_UTILIZATION=0
DC_512M_MAX_CONCURRENT=0
DC_512M_MAX_MEMORY=1073741824
DC_512M_MAX_PARALLELISM=1
DC_512M_MAX_PARALLELISM_GPU=0
DC_512M_MIN_MEMORY=1073741824
DC_64G_BASE_MIN_MEMORY=1073741824
DC_64G_GPU_UTILIZATION=0
DC_64G_MAX_CONCURRENT=0
DC_64G_MAX_MEMORY=8589934592
DC_64G_MAX_PARALLELISM=1
DC_64G_MAX_PARALLELISM_GPU=0
DC_64G_MIN_MEMORY=8589934592
DC_8M_BASE_MIN_MEMORY=8388608
DC_8M_GPU_UTILIZATION=0
DC_8M_MAX_CONCURRENT=0
DC_8M_MAX_MEMORY=8388608
DC_8M_MAX_PARALLELISM=1
DC_8M_MAX_PARALLELISM_GPU=0
DC_8M_MIN_MEMORY=8388608
GET_2K_BASE_MIN_MEMORY=0
GET_2K_GPU_UTILIZATION=0
GET_2K_MAX_CONCURRENT=0
GET_2K_MAX_MEMORY=1048576
GET_2K_MAX_PARALLELISM=0
GET_2K_MAX_PARALLELISM_GPU=0
GET_2K_MIN_MEMORY=1048576
GET_32G_BASE_MIN_MEMORY=0
GET_32G_GPU_UTILIZATION=0
GET_32G_MAX_CONCURRENT=0
GET_32G_MAX_MEMORY=1048576
GET_32G_MAX_PARALLELISM=0
GET_32G_MAX_PARALLELISM_GPU=0
GET_32G_MIN_MEMORY=1048576
GET_512M_BASE_MIN_MEMORY=0
GET_512M_GPU_UTILIZATION=0
GET_512M_MAX_CONCURRENT=0
GET_512M_MAX_MEMORY=1048576
GET_512M_MAX_PARALLELISM=0
GET_512M_MAX_PARALLELISM_GPU=0
GET_512M_MIN_MEMORY=1048576
GET_64G_BASE_MIN_MEMORY=0
GET_64G_GPU_UTILIZATION=0
GET_64G_MAX_CONCURRENT=0
GET_64G_MAX_MEMORY=1048576
GET_64G_MAX_PARALLELISM=0
GET_64G_MAX_PARALLELISM_GPU=0
GET_64G_MIN_MEMORY=1048576
GET_8M_BASE_MIN_MEMORY=0
GET_8M_GPU_UTILIZATION=0
GET_8M_MAX_CONCURRENT=0
GET_8M_MAX_MEMORY=1048576
GET_8M_MAX_PARALLELISM=0
GET_8M_MAX_PARALLELISM_GPU=0
GET_8M_MIN_MEMORY=1048576
GSK_2K_BASE_MIN_MEMORY=2048
GSK_2K_GPU_UTILIZATION=0
GSK_2K_MAX_CONCURRENT=0
GSK_2K_MAX_MEMORY=2048
GSK_2K_MAX_PARALLELISM=1
GSK_2K_MAX_PARALLELISM_GPU=0
GSK_2K_MIN_MEMORY=2048
GSK_32G_BASE_MIN_MEMORY=1073741824
GSK_32G_GPU_UTILIZATION=0
GSK_32G_MAX_CONCURRENT=0
GSK_32G_MAX_MEMORY=4294967296
GSK_32G_MAX_PARALLELISM=1
GSK_32G_MAX_PARALLELISM_GPU=0
GSK_32G_MIN_MEMORY=4294967296
GSK_512M_BASE_MIN_MEMORY=1073741824
GSK_512M_GPU_UTILIZATION=0
GSK_512M_MAX_CONCURRENT=0
GSK_512M_MAX_MEMORY=1073741824
GSK_512M_MAX_PARALLELISM=1
GSK_512M_MAX_PARALLELISM_GPU=0
GSK_512M_MIN_MEMORY=1073741824
GSK_64G_BASE_MIN_MEMORY=1073741824
GSK_64G_GPU_UTILIZATION=0
GSK_64G_MAX_CONCURRENT=0
GSK_64G_MAX_MEMORY=8589934592
GSK_64G_MAX_PARALLELISM=1
GSK_64G_MAX_PARALLELISM_GPU=0
GSK_64G_MIN_MEMORY=8589934592
GSK_8M_BASE_MIN_MEMORY=8388608
GSK_8M_GPU_UTILIZATION=0
GSK_8M_MAX_CONCURRENT=0
GSK_8M_MAX_MEMORY=8388608
GSK_8M_MAX_PARALLELISM=1
GSK_8M_MAX_PARALLELISM_GPU=0
GSK_8M_MIN_MEMORY=8388608
PC1_2K_BASE_MIN_MEMORY=2048
PC1_2K_GPU_UTILIZATION=0
PC1_2K_MAX_CONCURRENT=0
PC1_2K_MAX_MEMORY=2048
PC1_2K_MAX_PARALLELISM=1
PC1_2K_MAX_PARALLELISM_GPU=0
PC1_2K_MIN_MEMORY=2048
PC1_32G_BASE_MIN_MEMORY=10485760
PC1_32G_GPU_UTILIZATION=0
PC1_32G_MAX_CONCURRENT=0
PC1_32G_MAX_MEMORY=68719476736
PC1_32G_MAX_PARALLELISM=1
PC1_32G_MAX_PARALLELISM_GPU=0
PC1_32G_MIN_MEMORY=60129542144
PC1_512M_BASE_MIN_MEMORY=1048576
PC1_512M_GPU_UTILIZATION=0
PC1_512M_MAX_CONCURRENT=0
PC1_512M_MAX_MEMORY=1073741824
PC1_512M_MAX_PARALLELISM=1
PC1_512M_MAX_PARALLELISM_GPU=0
PC1_512M_MIN_MEMORY=805306368
PC1_64G_BASE_MIN_MEMORY=10485760
PC1_64G_GPU_UTILIZATION=0
PC1_64G_MAX_CONCURRENT=0
PC1_64G_MAX_MEMORY=137438953472
PC1_64G_MAX_PARALLELISM=1
PC1_64G_MAX_PARALLELISM_GPU=0
PC1_64G_MIN_MEMORY=120259084288
PC1_8M_BASE_MIN_MEMORY=8388608
PC1_8M_GPU_UTILIZATION=0
PC1_8M_MAX_CONCURRENT=0
PC1_8M_MAX_MEMORY=8388608
PC1_8M_MAX_PARALLELISM=1
PC1_8M_MAX_PARALLELISM_GPU=0
PC1_8M_MIN_MEMORY=8388608
PC2_2K_BASE_MIN_MEMORY=2048
PC2_2K_GPU_UTILIZATION=0
PC2_2K_MAX_CONCURRENT=0
PC2_2K_MAX_MEMORY=2048
PC2_2K_MAX_PARALLELISM=-1
PC2_2K_MAX_PARALLELISM_GPU=0
PC2_2K_MIN_MEMORY=2048
PC2_32G_BASE_MIN_MEMORY=1073741824
PC2_32G_GPU_UTILIZATION=1
PC2_32G_MAX_CONCURRENT=0
PC2_32G_MAX_MEMORY=16106127360
PC2_32G_MAX_PARALLELISM=-1
PC2_32G_MAX_PARALLELISM_GPU=6
PC2_32G_MIN_MEMORY=16106127360
PC2_512M_BASE_MIN_MEMORY=1073741824
PC2_512M_GPU_UTILIZATION=0
PC2_512M_MAX_CONCURRENT=0
PC2_512M_MAX_MEMORY=1610612736
PC2_512M_MAX_PARALLELISM=-1
PC2_512M_MAX_PARALLELISM_GPU=0
PC2_512M_MIN_MEMORY=1073741824
PC2_64G_BASE_MIN_MEMORY=1073741824
PC2_64G_GPU_UTILIZATION=1
PC2_64G_MAX_CONCURRENT=0
PC2_64G_MAX_MEMORY=32212254720
PC2_64G_MAX_PARALLELISM=-1
PC2_64G_MAX_PARALLELISM_GPU=6
PC2_64G_MIN_MEMORY=32212254720
PC2_8M_BASE_MIN_MEMORY=8388608
PC2_8M_GPU_UTILIZATION=0
PC2_8M_MAX_CONCURRENT=0
PC2_8M_MAX_MEMORY=8388608
PC2_8M_MAX_PARALLELISM=-1
PC2_8M_MAX_PARALLELISM_GPU=0
PC2_8M_MIN_MEMORY=8388608
PR1_2K_BASE_MIN_MEMORY=2048
PR1_2K_GPU_UTILIZATION=0
PR1_2K_MAX_CONCURRENT=0
PR1_2K_MAX_MEMORY=2048
PR1_2K_MAX_PARALLELISM=0
PR1_2K_MAX_PARALLELISM_GPU=0
PR1_2K_MIN_MEMORY=2048
PR1_32G_BASE_MIN_MEMORY=1073741824
PR1_32G_GPU_UTILIZATION=0
PR1_32G_MAX_CONCURRENT=0
PR1_32G_MAX_MEMORY=1073741824
PR1_32G_MAX_PARALLELISM=0
PR1_32G_MAX_PARALLELISM_GPU=0
PR1_32G_MIN_MEMORY=1073741824
PR1_512M_BASE_MIN_MEMORY=1073741824
PR1_512M_GPU_UTILIZATION=0
PR1_512M_MAX_CONCURRENT=0
PR1_512M_MAX_MEMORY=1073741824
PR1_512M_MAX_PARALLELISM=0
PR1_512M_MAX_PARALLELISM_GPU=0
PR1_512M_MIN_MEMORY=1073741824
PR1_64G_BASE_MIN_MEMORY=1073741824
PR1_64G_GPU_UTILIZATION=0
PR1_64G_MAX_CONCURRENT=0
PR1_64G_MAX_MEMORY=1073741824
PR1_64G_MAX_PARALLELISM=0
PR1_64G_MAX_PARALLELISM_GPU=0
PR1_64G_MIN_MEMORY=1073741824
PR1_8M_BASE_MIN_MEMORY=8388608
PR1_8M_GPU_UTILIZATION=0
PR1_8M_MAX_CONCURRENT=0
PR1_8M_MAX_MEMORY=8388608
PR1_8M_MAX_PARALLELISM=0
PR1_8M_MAX_PARALLELISM_GPU=0
PR1_8M_MIN_MEMORY=8388608
PR2_2K_BASE_MIN_MEMORY=2048
PR2_2K_GPU_UTILIZATION=1
PR2_2K_MAX_CONCURRENT=0
PR2_2K_MAX_MEMORY=2048
PR2_2K_MAX_PARALLELISM=1
PR2_2K_MAX_PARALLELISM_GPU=0
PR2_2K_MIN_MEMORY=2048
PR2_32G_BASE_MIN_MEMORY=34359738368
PR2_32G_GPU_UTILIZATION=1
PR2_32G_MAX_CONCURRENT=0
PR2_32G_MAX_MEMORY=161061273600
PR2_32G_MAX_PARALLELISM=-1
PR2_32G_MAX_PARALLELISM_GPU=6
PR2_32G_MIN_MEMORY=32212254720
PR2_512M_BASE_MIN_MEMORY=10737418240
PR2_512M_GPU_UTILIZATION=1
PR2_512M_MAX_CONCURRENT=0
PR2_512M_MAX_MEMORY=1610612736
PR2_512M_MAX_PARALLELISM=1
PR2_512M_MAX_PARALLELISM_GPU=0
PR2_512M_MIN_MEMORY=1073741824
PR2_64G_BASE_MIN_MEMORY=68719476736
PR2_64G_GPU_UTILIZATION=1
PR2_64G_MAX_CONCURRENT=0
PR2_64G_MAX_MEMORY=204010946560
PR2_64G_MAX_PARALLELISM=-1
PR2_64G_MAX_PARALLELISM_GPU=6
PR2_64G_MIN_MEMORY=64424509440
PR2_8M_BASE_MIN_MEMORY=8388608
PR2_8M_GPU_UTILIZATION=1
PR2_8M_MAX_CONCURRENT=0
PR2_8M_MAX_MEMORY=8388608
PR2_8M_MAX_PARALLELISM=1
PR2_8M_MAX_PARALLELISM_GPU=0
PR2_8M_MIN_MEMORY=8388608
RU_2K_BASE_MIN_MEMORY=2048
RU_2K_GPU_UTILIZATION=0
RU_2K_MAX_CONCURRENT=0
RU_2K_MAX_MEMORY=2048
RU_2K_MAX_PARALLELISM=1
RU_2K_MAX_PARALLELISM_GPU=0
RU_2K_MIN_MEMORY=2048
RU_32G_BASE_MIN_MEMORY=1073741824
RU_32G_GPU_UTILIZATION=0
RU_32G_MAX_CONCURRENT=0
RU_32G_MAX_MEMORY=4294967296
RU_32G_MAX_PARALLELISM=1
RU_32G_MAX_PARALLELISM_GPU=0
RU_32G_MIN_MEMORY=4294967296
RU_512M_BASE_MIN_MEMORY=1073741824
RU_512M_GPU_UTILIZATION=0
RU_512M_MAX_CONCURRENT=0
RU_512M_MAX_MEMORY=1073741824
RU_512M_MAX_PARALLELISM=1
RU_512M_MAX_PARALLELISM_GPU=0
RU_512M_MIN_MEMORY=1073741824
RU_64G_BASE_MIN_MEMORY=1073741824
RU_64G_GPU_UTILIZATION=0
RU_64G_MAX_CONCURRENT=0
RU_64G_MAX_MEMORY=8589934592
RU_64G_MAX_PARALLELISM=1
RU_64G_MAX_PARALLELISM_GPU=0
RU_64G_MIN_MEMORY=8589934592
RU_8M_BASE_MIN_MEMORY=8388608
RU_8M_GPU_UTILIZATION=0
RU_8M_MAX_CONCURRENT=0
RU_8M_MAX_MEMORY=8388608
RU_8M_MAX_PARALLELISM=1
RU_8M_MAX_PARALLELISM_GPU=0
RU_8M_MIN_MEMORY=8388608
UNS_2K_BASE_MIN_MEMORY=2048
UNS_2K_GPU_UTILIZATION=0
UNS_2K_MAX_CONCURRENT=0
UNS_2K_MAX_MEMORY=2048
UNS_2K_MAX_PARALLELISM=1
UNS_2K_MAX_PARALLELISM_GPU=0
UNS_2K_MIN_MEMORY=2048
UNS_32G_BASE_MIN_MEMORY=10485760
UNS_32G_GPU_UTILIZATION=0
UNS_32G_MAX_CONCURRENT=0
UNS_32G_MAX_MEMORY=68719476736
UNS_32G_MAX_PARALLELISM=1
UNS_32G_MAX_PARALLELISM_GPU=0
UNS_32G_MIN_MEMORY=60129542144
UNS_512M_BASE_MIN_MEMORY=1048576
UNS_512M_GPU_UTILIZATION=0
UNS_512M_MAX_CONCURRENT=0
UNS_512M_MAX_MEMORY=1073741824
UNS_512M_MAX_PARALLELISM=1
UNS_512M_MAX_PARALLELISM_GPU=0
UNS_512M_MIN_MEMORY=805306368
UNS_64G_BASE_MIN_MEMORY=10485760
UNS_64G_GPU_UTILIZATION=0
UNS_64G_MAX_CONCURRENT=0
UNS_64G_MAX_MEMORY=137438953472
UNS_64G_MAX_PARALLELISM=1
UNS_64G_MAX_PARALLELISM_GPU=0
UNS_64G_MIN_MEMORY=120259084288
UNS_8M_BASE_MIN_MEMORY=8388608
UNS_8M_GPU_UTILIZATION=0
UNS_8M_MAX_CONCURRENT=0
UNS_8M_MAX_MEMORY=8388608
UNS_8M_MAX_PARALLELISM=1
UNS_8M_MAX_PARALLELISM_GPU=0
UNS_8M_MIN_MEMORY=8388608
WDP_2K_BASE_MIN_MEMORY=2048
WDP_2K_GPU_UTILIZATION=1
WDP_2K_MAX_CONCURRENT=0
WDP_2K_MAX_MEMORY=2048
WDP_2K_MAX_PARALLELISM=1
WDP_2K_MAX_PARALLELISM_GPU=0
WDP_2K_MIN_MEMORY=2048
WDP_32G_BASE_MIN_MEMORY=34359738368
WDP_32G_GPU_UTILIZATION=1
WDP_32G_MAX_CONCURRENT=0
WDP_32G_MAX_MEMORY=103079215104
WDP_32G_MAX_PARALLELISM=-1
WDP_32G_MAX_PARALLELISM_GPU=6
WDP_32G_MIN_MEMORY=32212254720
WDP_512M_BASE_MIN_MEMORY=10737418240
WDP_512M_GPU_UTILIZATION=1
WDP_512M_MAX_CONCURRENT=0
WDP_512M_MAX_MEMORY=1610612736
WDP_512M_MAX_PARALLELISM=1
WDP_512M_MAX_PARALLELISM_GPU=0
WDP_512M_MIN_MEMORY=1073741824
WDP_64G_BASE_MIN_MEMORY=68719476736
WDP_64G_GPU_UTILIZATION=1
WDP_64G_MAX_CONCURRENT=0
WDP_64G_MAX_MEMORY=128849018880
WDP_64G_MAX_PARALLELISM=-1
WDP_64G_MAX_PARALLELISM_GPU=6
WDP_64G_MIN_MEMORY=64424509440
WDP_8M_BASE_MIN_MEMORY=8388608
WDP_8M_GPU_UTILIZATION=1
WDP_8M_MAX_CONCURRENT=0
WDP_8M_MAX_MEMORY=8388608
WDP_8M_MAX_PARALLELISM=1
WDP_8M_MAX_PARALLELISM_GPU=0
WDP_8M_MIN_MEMORY=8388608
WNP_2K_BASE_MIN_MEMORY=2048
WNP_2K_GPU_UTILIZATION=1
WNP_2K_MAX_CONCURRENT=0
WNP_2K_MAX_MEMORY=2048
WNP_2K_MAX_PARALLELISM=1
WNP_2K_MAX_PARALLELISM_GPU=0
WNP_2K_MIN_MEMORY=2048
WNP_32G_BASE_MIN_MEMORY=34359738368
WNP_32G_GPU_UTILIZATION=1
WNP_32G_MAX_CONCURRENT=0
WNP_32G_MAX_MEMORY=1073741824
WNP_32G_MAX_PARALLELISM=-1
WNP_32G_MAX_PARALLELISM_GPU=6
WNP_32G_MIN_MEMORY=1073741824
WNP_512M_BASE_MIN_MEMORY=10737418240
WNP_512M_GPU_UTILIZATION=1
WNP_512M_MAX_CONCURRENT=0
WNP_512M_MAX_MEMORY=2048
WNP_512M_MAX_PARALLELISM=1
WNP_512M_MAX_PARALLELISM_GPU=0
WNP_512M_MIN_MEMORY=2048
WNP_64G_BASE_MIN_MEMORY=68719476736
WNP_64G_GPU_UTILIZATION=1
WNP_64G_MAX_CONCURRENT=0
WNP_64G_MAX_MEMORY=1073741824
WNP_64G_MAX_PARALLELISM=-1
WNP_64G_MAX_PARALLELISM_GPU=6
WNP_64G_MIN_MEMORY=1073741824
WNP_8M_BASE_MIN_MEMORY=8388608
WNP_8M_GPU_UTILIZATION=1
WNP_8M_MAX_CONCURRENT=0
WNP_8M_MAX_MEMORY=8388608
WNP_8M_MAX_PARALLELISM=1
WNP_8M_MAX_PARALLELISM_GPU=0
WNP_8M_MIN_MEMORY=8388608
```
{{< /details >}}

## Advanced PoSt worker configurations

Although the default settings for the PoSt workers are reasonable, you can configure some advanced settings when running the PoSt workers. These settings should be tested for local optimizations of your hardware. Most of the advanced PoSt worker configurations can be found in the helptext.

### Proving challenges

The `--post-parallel-reads` option lets you set an upper boundary of how many challenges it reads from your storage simultaneously. This option is set to 128 by default.

```plaintext
--post-parallel-reads value   maximum number of parallel challenge reads (0 = no limit) (default: 128)
```

The `--post-read-timeout` option lets you set a cut off time for reading challenges from storage, after which it will abort the job. This option has no default limit.

```plaintext
--post-read-timeout value     time limit for reading PoSt challenges (0 = no limit) (default: 0s)
```

Both these settings can be set at runtime of the PoSt workers.

{{< alert icon="tip" >}}
Use with caution. Changing these values to extremes might cause you to miss windowPoSt.
 {{< /alert >}}
 
### PoSt computations

Window PoSt computations on the lotus-miner process can be fully disabled by setting `DisableBuiltinWindowPoSt = true` in your lotus-miner config file.

{{< alert icon="tip" >}}
This setting will be respected even if there are no connected window PoSt workers. Before enabling this option please ensure that your PoSt workers are correctly configured and available.
{{< /alert >}}
 
Winnning PoSt computations on the lotus-miner process can be also be fully disabled by setting `DisableBuiltinWinningPoSt = true` in your lotus-miner config file.

{{< alert icon="tip" >}}
This setting will also be respected even if there are no connected winning PoSt workers. Before enabling this option please ensure that your PoSt workers are correctly configured and available.
{{< /alert >}}

### Proving pre-checks

In normal operation, when preparing to compute WindowPoSt, lotus-miner will perform a round of reading challenges from all sectors to confirm that those sectors can be proven.

Disabling sector pre-checks will slightly reduce IO load when proving sectors, possibly resulting in shorter time to produce window PoSt. In setups with good IO capabilities the effect of this option on proving time should be negligible.

Sector pre-checks can be disabled by setting `DisableWDPoStPreChecks = true` in your lotus-miner config file.

{{< alert icon="tip" >}}
Disabling sector pre-checks on systems with no PoSt workers is strongly discouraged.
{{< /alert >}}

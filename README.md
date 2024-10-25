# Wall-time Profiler Demo

## Introduction & Problem Motivation 

The profiler currently available in Julia v1.10 is a sampling CPU profiler. At a high level, the profiler periodically stops all Julia compute threads to collect their backtraces and estimates the time spent in each function based on the number of backtrace samples that include a frame from that function. However, note that only tasks currently running on system threads just before the profiler stops them will have their backtraces collected.

While this profiler is typically well-suited for workloads where the majority of tasks are compute-bound, it is less helpful for systems where most tasks are IO-heavy or for diagnosing contention on synchronization primitives in your code.

Let's consider this simple workload:

```Julia
using Base.Threads
using Profile
using PProf

ch = Channel(1)

const N_SPAWNED_TASKS = (1 << 10)
const WAIT_TIME_NS = 10_000_000

function spawn_a_bunch_of_tasks_waiting_on_channel()
    for i in 1:N_SPAWNED_TASKS
        Threads.@spawn begin
            take!(ch)
        end
    end
end

function busywait()
    t0 = time_ns()
    while true
        if time_ns() - t0 > WAIT_TIME_NS
            break
        end
    end
end

function main()
    spawn_a_bunch_of_tasks_waiting_on_channel()
    for i in 1:N_SPAWNED_TASKS
        put!(ch, i)
        busywait()
    end
end

Profile.@profile main()
```

Our goal is to detect whether there is contention on the `ch` channel—i.e., whether the number of waiters is excessive given the rate at which work items are being produced in the channel.

If we run this, we obtain the following [PProf](https://github.com/JuliaPerf/PProf.jl) flame graph:

<img width="1496" alt="Screenshot 2024-10-24 at 16 44 13" src="https://github.com/user-attachments/assets/81a076e1-bbca-4baa-a0c8-d4893ff13a72">

This profile provides no information to help determine where contention occurs in the system’s synchronization primitives. Waiters on a channel will be blocked and descheduled, meaning no system thread will be running the tasks assigned to those waiters, and as a result, they won't be sampled by the profiler.

## Wall-time Profiler

Instead of sampling threads—and thus only sampling tasks that are running—a wall-time task profiler samples tasks independently of their scheduling state. For example, tasks that are sleeping on a synchronization primitive at the time the profiler is running will be sampled with the same probability as tasks that were actively running when the profiler attempted to capture backtraces.

This approach allows us to construct a profile where backtraces from tasks blocked on the `ch` channel, as in the example above, are actually represented.

Let's run the same example, but now with a wall-time profiler:


```Julia
using Base.Threads
using Profile
using PProf

ch = Channel(1)

const N_SPAWNED_TASKS = (1 << 10)
const WAIT_TIME_NS = 10_000_000

function spawn_a_bunch_of_tasks_waiting_on_channel()
    for i in 1:N_SPAWNED_TASKS
        Threads.@spawn begin
            take!(ch)
        end
    end
end

function busywait()
    t0 = time_ns()
    while true
        if time_ns() - t0 > WAIT_TIME_NS
            break
        end
    end
end

function main()
    spawn_a_bunch_of_tasks_waiting_on_channel()
    for i in 1:N_SPAWNED_TASKS
        put!(ch, i)
        busywait()
    end
end

Profile.@profile_walltime main()
```

We obtain the following flame graph:

<img width="1506" alt="Screenshot 2024-10-25 at 10 42 23" src="https://github.com/user-attachments/assets/fa0d4ecb-a61c-432b-86d2-ea6d1afecff1">

We see that a large number of samples come from channel-related `take!` functions, which allows us to determine that there is indeed an excessive number of waiters in `ch`.

## A Compute-Bound Workload

Despite the wall-time profiler sampling all live tasks in the system and not just the currently running ones, it can still be helpful for identifying performance hotspots, even if your code is compute-bound. Let’s consider a simple example:

```Julia
using Base.Threads
using Profile
using PProf

ch = Channel(1)

const MAX_ITERS = (1 << 22)
const N_TASKS = (1 << 12)

function spawn_a_task_waiting_on_channel()
    Threads.@spawn begin
        take!(ch)
    end
end

function sum_of_sqrt()
    sum_of_sqrt = 0.0
    for i in 1:MAX_ITERS
        sum_of_sqrt += sqrt(i)
    end
    return sum_of_sqrt
end

function spawn_a_bunch_of_compute_heavy_tasks()
    Threads.@sync begin
        for i in 1:N_TASKS
            Threads.@spawn begin
                sum_of_sqrt()
            end
        end
    end
end

function main()
    spawn_a_task_waiting_on_channel()
    spawn_a_bunch_of_compute_heavy_tasks()
end

Profile.@profile_walltime main()
```

After collecting a wall-time profile, we get the following flame graph:

<img width="1512" alt="Screenshot 2024-10-25 at 10 45 28" src="https://github.com/user-attachments/assets/e7045dd7-c9d1-4547-b8d2-720bf44aac8d">

Notice how many of the samples contain `sum_of_sqrt`, which is the expensive compute function in our example.

## Identifying Task Sampling Failures in your Profile

In the current implementation, the wall-time profiler attempts to sample from tasks that have been alive since the last garbage collection, along with those created afterward. However, if most tasks are extremely short-lived, you may end up sampling tasks that have already completed, resulting in missed backtrace captures.

If you encounter samples containing `failed_to_sample_task_fun` or `failed_to_stop_thread_fun`, this likely indicates a high volume of short-lived tasks, which prevented their backtraces from being collected.

Let's consider this simple example:

```Julia
using Base.Threads
using Profile
using PProf

const N_SPAWNED_TASKS = (1 << 16)
const WAIT_TIME_NS = 100_000

function spawn_a_bunch_of_short_lived_tasks()
    for i in 1:N_SPAWNED_TASKS
        Threads.@spawn begin
            # Do nothing
        end
    end
end

function busywait()
    t0 = time_ns()
    while true
        if time_ns() - t0 > WAIT_TIME_NS
            break
        end
    end
end

function main()
    GC.enable(false)
    spawn_a_bunch_of_short_lived_tasks()
    for i in 1:N_SPAWNED_TASKS
        busywait()
    end
    GC.enable(true)
end

Profile.@profile_walltime main()
```

Notice that the tasks spawned in `spawn_a_bunch_of_short_lived_tasks` are extremely short-lived. Since these tasks constitute the majority in the system, we will likely miss capturing a backtrace for most sampled tasks.

After collecting a wall-time profile, we obtain the following flame graph:

<img width="1508" alt="Screenshot 2024-10-25 at 11 10 17" src="https://github.com/user-attachments/assets/54c13909-20f6-4476-ae83-c4a401fda05a">

The large number of samples from `failed_to_stop_thread_fun` confirms that we have a significant number of short-lived tasks in the system.

---
description: The runtime context allows global access to information about the current run.
tags:
    - flows
    - subflows
    - tasks
    - deployments
---


# Runtime context

Prefect tracks information about the current flow or task run with a run context. The run context can be thought of as a global variable that allows the Prefect engine to determine relationships between your runs, e.g. which flow your task is called from.

The run context itself contains many internal objects used by Prefect to manage execution of your run and is only available in specific situtations. For this reason, we expose a simple interface that only includes the items you care about and dynamically retrieves additional information when necessary. We call this the "runtime context" as it contains information that can only be accessed when a run is happening.

!!! tip "Mock values via environment variable"
    Oftentimes, you may want to mock certain values for testing purposes.  For example, by manually setting an ID or a scheduled start time to ensure your code is functioning properly.  Starting in version `2.10.3`, you can mock values in runtime via environment variable using the schema `PREFECT__RUNTIME__{SUBMODULE}__{KEY_NAME}=value`:
    <div class="terminal">
    ```bash
    $ export PREFECT__RUNTIME__TASK_RUN__FAKE_KEY='foo'
    $ python -c 'from prefect.runtime import task_run; print(task_run.fake_key)' # "foo"
    ```
    </div>


## Accessing runtime information

The `prefect.runtime` module is the home for all runtime context access. Each major runtime concept has its own submodule:

- `deployment`: Access information about the deployment for the current run
- `flow_run`: Access information about the current flow run
- `task_run`: Access information about the current task run


For example:

```python hl_lines="2 6 7 12 13"
from prefect import flow, task
from prefect import runtime

@flow(log_prints=True)
def my_flow(x):
    print("My name is", runtime.flow_run.name)
    print("I belong to deployment", runtime.deployment.name)
    my_task(2)

@task
def my_task(y):
    print("My name is", runtime.task_run.name)
    print("Flow run parameters:", runtime.flow_run.parameters)

my_flow(1)
```

Outputs:

<div class="terminal">
```bash
$ python my_runtime_info.py
10:08:02.948 | INFO    | prefect.engine - Created flow run 'solid-gibbon' for flow 'my-flow'
10:08:03.555 | INFO    | Flow run 'solid-gibbon' - My name is solid-gibbon
10:08:03.558 | INFO    | Flow run 'solid-gibbon' - I belong to deployment None
10:08:03.703 | INFO    | Flow run 'solid-gibbon' - Created task run 'my_task-0' for task 'my_task'
10:08:03.704 | INFO    | Flow run 'solid-gibbon' - Executing 'my_task-0' immediately...
10:08:04.006 | INFO    | Task run 'my_task-0' - My name is my_task-0
10:08:04.007 | INFO    | Task run 'my_task-0' - Flow run parameters: {'x': 1}
10:08:04.105 | INFO    | Task run 'my_task-0' - Finished in state Completed()
10:08:04.968 | INFO    | Flow run 'solid-gibbon' - Finished in state Completed('All states completed.')
```
</div>
Above, we demonstrate access to information about the current flow run, task run, and deployment. When run without a deployment (via `python my_runtime_info.py`), you should see `"I belong to deployment None"` logged. When information is not available, the runtime will always return an empty value. Because this flow was run without a deployment, there is no deployment data. If this flow were deployed and executed by an agent or worker, we'd see the name of the deployment instead.

See the [runtime API reference](/api-ref/prefect/runtime/flow_run/) for a full list of available attributes.

## Accessing the run context directly

The current run context can be accessed with `prefect.context.get_run_context()`. This will raise an exception if no run context is available, meaning you are not in a flow or task run. If a task run context is available, it will be returned even if a flow run context is available.

Alternatively, you can access the flow run or task run context explicitly. This will, for example, allow you to access the flow run context from a task run. Note that we do not send the flow run context to distributed task workers because the context is costly to serialize and deserialize.

```python
from prefect.context import FlowRunContext, TaskRunContext

flow_run_ctx = FlowRunContext.get()
task_run_ctx = TaskRunContext.get()
```

Unlike `get_run_context`, this will not raise an error if the context is not available. Instead, it will return `None`.

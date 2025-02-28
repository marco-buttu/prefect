---
description: Execute code in response to a flow or task entering a given state, without involvement of the Prefect API.
tags:
    - state change
    - hooks
    - triggers
---

# State Change Hooks

You've read about how [state change hooks execute code in response to changes in flow or task run states](/concepts/states/#state-change-hooks), enabling you to define actions for specific state transitions in a workflow. Now let's see some real-world use cases!

## Example use cases

### Send a notification when a flow run fails
State change hooks enable you to customize messages sent when tasks transition between states, such as sending notifications containing sensitive information when tasks enter a `Failed` state. Let's run a client-side hook upon a flow run entering a `Failed` state.

```python
from prefect import flow
from prefect.blocks.core import Block
from prefect.settings import PREFECT_API_URL

def notify_slack(flow, flow_run, state):
    slack_webhook_block = Block.load(
        "slack-webhook/my-slack-webhook"
    )
            
    slack_webhook_block.notify(
        (
            f"Your job {flow_run.name} entered {state.name} "
            f"with message:\n\n"
            f"See <https://{PREFECT_API_URL.value()}/flow-runs/"
            f"flow-run/{flow_run.id}|the flow run in the UI>\n\n"
            f"Tags: {flow_run.tags}\n\n"
            f"Scheduled start: {flow_run.expected_start_time}"
        )
    )

@flow(on_failure=[notify_slack], retries=1)
def failing_flow():
    raise ValueError("oops!")

if __name__ == "__main__":
    failing_flow()
```

Note that because we've configured retries here, the `on_failure` hook will not run until all `retries` have completed, when the flow run finally enters a `Failed` state.


### Delete a Cloud Run job when a flow crashes
State change hooks can aid in managing infrastructure cleanup in scenarios where tasks spin up individual infrastructure resources independently of Prefect. When a flow run crashes, tasks may exit abruptly, resulting in the potential omission of cleanup logic within the tasks. State change hooks can be used to ensure infrastructure is properly cleaned up even when a flow run enters a `Crashed` state.

Let's create a hook that deletes a Cloud Run job if the flow run crashes.

```python
import os
from prefect import flow, task
from prefect.blocks.system import String
from prefect.client import get_client
import prefect.runtime

async def delete_cloud_run_job(flow, flow_run, state):
    """Flow run state change hook that deletes a Cloud Run Job if
    the flow run crashes."""

    # retrieve Cloud Run job name
    cloud_run_job_name = await String.load(
        name="crashing-flow-cloud-run-job"
    )

    # delete Cloud Run job
    delete_job_command = f"yes | gcloud beta run jobs delete 
    {cloud_run_job_name.value} --region us-central1"
    os.system(delete_job_command)

    # clean up the Cloud Run job string block as well
    async with get_client() as client:
        block_document = await client.read_block_document_by_name(
            "crashing-flow-cloud-run-job", block_type_slug="string"
        )
        await client.delete_block_document(block_document.id)

@task
def my_task_that_crashes():
    raise SystemExit("Crashing on purpose!")

@flow(on_crashed=[delete_cloud_run_job])
def crashing_flow():
    """Save the flow run name (i.e. Cloud Run job name) as a 
    String block. It then executes a task that ends up crashing."""
    flow_run_name = prefect.runtime.flow_run.name
    cloud_run_job_name = String(value=flow_run_name)
    cloud_run_job_name.save(
        name="crashing-flow-cloud-run-job", overwrite=True
    )

    my_task_that_crashes()

if __name__ == "__main__":
    crashing_flow()
```

---
description: Documentation on A/B testing of model deployments, how to run a candidate configuration next to the live one with a share of the traffic, and roll it out or discard it.
---

# How To A/B Test a Deployment { #ab-testing }

## Introduction

In this guide, you will learn how to test a new configuration of a deployment against live traffic before you commit to it.

A ==candidate== is a second configuration of a deployment that runs next to the live one and receives a chosen share of the traffic.
When you are satisfied, you **roll it out**, and it becomes the active version of the deployment, or you **discard** it.
The live version never changes during a test.

## Requirements

- The deployment is served by KServe in **Standard** mode, that is `knative_mode=False` in Python and the `Knative` checkbox cleared in the UI, see [Deployment mode](autoscaling.md#deployment-mode).
- The deployment is running.

The mode is set when the deployment is created, and it can only be changed while the deployment is stopped.
A deployment in Knative mode cannot have a candidate.

## Candidates and versions

A candidate relates to the [deployment versions][deployment-versions] of the deployment as follows:

- A candidate takes the next version number, but it is not activated.
- Rolling it out activates it, with the activation reason `Rollout` in the `Versions` card, and the deployment moves to that number.
- Discarding it keeps the number as a version that was never activated, so numbers are never reused and logged predictions stay attributable.
- While a candidate exists, `Save`, `Save as new version`, `Roll back` and `Stop` are refused until you roll out or discard it.
- A candidate cannot be edited: discard it and start a new one.

## Traffic

A candidate starts at 0% of the traffic, and its share, from 0 to 99%, can be set once it is running.
The rest of the traffic goes to the live version.

Both are reached through the normal endpoint of the deployment, so nothing changes for clients, see the [REST API Guide](rest-api.md).
When the deployment has a transformer, each of the two has its own transformer and predictor.

## Web UI

### Step 1: Start a candidate

Open the deployment and go to its edit page.
After making your changes, open the menu next to the `Save` button and click on `Start as candidate…`.
In the dialog, choose the share of the traffic for the candidate and confirm.

### Step 2: Follow the candidate

The `Candidate` card on the deployment overview page shows the status and the instances of the candidate and its share of the traffic.
Move the slider and click on `Apply` to change the share.
Click on `Details` to see the configuration of the candidate.

The `Logs` dialog has a `Primary` and a `Candidate` tab, to read the logs of each.

### Step 3: Roll out or discard

Click on `Roll out` to make the candidate the active version, or on `Discard` to remove it and send all the traffic to the live version again.
During a rollout, the card shows the restart of the deployment, and it finishes automatically.

## Code

### Step 1: Connect to Hopsworks

=== "Python"

    ```python
    import hopsworks


    project = hopsworks.login()

    # get Hopsworks Model Serving handle
    ms = project.get_model_serving()

    # a running deployment in Standard mode
    deployment = ms.get_deployment("fraud")
    ```

### Step 2: Start a candidate

Edit the deployment as you would before calling `.save()`, then create the candidate.

=== "Python"

    ```python
    deployment.script_file = "predictor_v2.py"
    candidate = deployment.create_candidate(traffic_percentage=20)

    print(candidate.version, candidate.state, candidate.status)
    ```

Without the `predictor` argument, the current edits of the deployment object become the candidate, and the local object is reset to the live configuration.
You can pass a `Predictor` built with `ms.create_predictor(...)` instead.
The call waits for the candidate to run and then sets its traffic.

### Step 3: Change the traffic

=== "Python"

    ```python
    deployment.update_candidate_traffic(50)
    ```

### Step 4: Roll out or discard

=== "Python"

    ```python
    # the candidate becomes the active version
    deployment.rollout_candidate()

    # or discard it, the version number is kept
    deployment.delete_candidate()
    ```

A rollout restarts the deployment onto the configuration of the candidate.
The running instances are replaced, the candidate keeps serving until the new ones are ready, and then the candidate instance is removed.
If it times out, call `rollout_candidate()` again to finish.

### Step 5: Inspect the candidate

=== "Python"

    ```python
    print(deployment.candidate)  # None when there is no candidate

    version = deployment.get_version(2)
    print(version.candidate, version.activated)

    deployment.get_logs(component="predictor", variant="candidate")
    ```

!!! api "API reference"

    - <code class="doc-symbol doc-symbol-method"></code> [`ModelServing.get_deployment`][hsml.model_serving.ModelServing.get_deployment]
    - <code class="doc-symbol doc-symbol-class"></code> [`Deployment`][hsml.deployment.Deployment]
        - <code class="doc-symbol doc-symbol-method"></code> [`create_candidate`][hsml.deployment.Deployment.create_candidate]
        - <code class="doc-symbol doc-symbol-method"></code> [`update_candidate_traffic`][hsml.deployment.Deployment.update_candidate_traffic]
        - <code class="doc-symbol doc-symbol-method"></code> [`rollout_candidate`][hsml.deployment.Deployment.rollout_candidate]
        - <code class="doc-symbol doc-symbol-method"></code> [`delete_candidate`][hsml.deployment.Deployment.delete_candidate]
        - <code class="doc-symbol doc-symbol-method"></code> [`get_version`][hsml.deployment.Deployment.get_version]
    - <code class="doc-symbol doc-symbol-class"></code> [`DeploymentCandidate`][hsml.deployment_candidate.DeploymentCandidate]

    <a class="hops-api-cta" href="../../../../python-api/hopsworks/">Browse the full Python API :material-arrow-right:</a>

## Telling the arms apart in logged predictions

The feature logging of a deployment can carry the reserved `deployment_version` column, declared in the `extra_logging_columns` of the feature view, see [Feature logging][deployment-schema-feature-logging].
The pods of the candidate log their own version number, so the logged rows can be grouped per arm.
A candidate that serves a different model version can also be told apart by the `model_version` column.

## In the pod

Inside a candidate pod, the `DEPLOYMENT_VERSION` environment variable holds the version number of the candidate, see the [environment variables](predictor.md#environment-variables).

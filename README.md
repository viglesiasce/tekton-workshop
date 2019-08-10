
Based on https://github.com/tektoncd/pipeline/blob/master/docs/tutorial.md

# Tekton Workshop

## Create a Service Account for your pipelines to use

1. Create the GCP service account

    ```
    gcloud iam service-accounts create tekton --display-name tekton
    ```

1. Add permissions to the service account to be able to push images and deploy to your Kubernetes clusters.

    ```
    export GCP_PROJECT=$(gcloud config get-value project)
    gcloud projects add-iam-policy-binding ${GCP_PROJECT} \
           --member serviceAccount:tekton@${GCP_PROJECT}.iam.gserviceaccount.com \
           --role roles/storage.admin
    gcloud projects add-iam-policy-binding ${GCP_PROJECT} \
           --member serviceAccount:tekton@${GCP_PROJECT}.iam.gserviceaccount.com \
           --role roles/container.developer
    ```


## Create a Kubernetes Cluster

1. Enable the Kubernetes Engine API.

    ```
    gcloud services enable container.googleapis.com
    ```

1. Create the Google Kubernetes Engine Cluster that you'll use to deploy Tekton and its pipelines.

    ```shell
    gcloud config set compute/zone us-east1-d
    export GCP_PROJECT=$(gcloud config get-value project)
    gcloud container clusters create tekton-workshop --service-account tekton@${GCP_PROJECT}.iam.gserviceaccount.com \
                                    --scopes cloud-platform
    ```

## Installation

### Tekton

1. Install Tekton into your cluster.

```
kubectl apply --filename https://storage.googleapis.com/tekton-releases/latest/release.yaml
```

### `tkn` CLI

1. Install the `tkn` CLI into your Cloud Shell.

```
curl -LO https://github.com/tektoncd/cli/releases/download/v0.2.0/tkn_0.2.0_Linux_x86_64.tar.gz
sudo tar xvzf tkn_0.2.0_Linux_x86_64.tar.gz -C /usr/local/bin/ tkn
```

### Installing Tekton Dashboard

1. Install the Tekton Dashboard to visualize your Tekton pipelines.

```
kubectl apply --filename https://github.com/tektoncd/dashboard/releases/download/v0.1.0/release.yaml
```

## Hello World!


### Creating and running a task

1. First you'll create a task definition. In this case the task will echo "Hello Tekton Workshop!". Note that the task step spec is a normal Kubernetes [Pod Spec definition](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.15/#podspec-v1-core).

    ```
    kubectl apply -f tasks/hello-world.yaml
    ```

1. You'll notice nothing is running in your cluster yet. This is because we haven't yet created the resource needed to invoke our task, the TaskRun.

    ```
    kubectl get pods
    ```

1. Apply the TaskRun in your cluster

    ```
    kubectl apply -f taskruns/hello-world.yaml
    ```

1. Now you should see the task's pod running.

    ```
    kubectl get pods
    ```

1. When it completes you should be able to see the status of complete

    ```
    $  kubectl get pods
    NAME                                   READY   STATUS      RESTARTS   AGE
    echo-hello-world-task-run-pod-3ec0b2   0/1     Completed   0          28s
    ```

1. Now you can check the logs. We'll use the automatically added label `tekton.dev/task` to find a pod from our task.

    ```
    kubectl logs -l tekton.dev/task=echo-hello-world
    ```

### Creating a task with inputs and outputs

That was awesome! Kind of... We didn't really do much more than we could accomplish with a Kubernetes Job or Pod.

In this next section you'll create a reusable task that takes in inputs and creates outputs so that we can leverage the business logic of our task across different pipelines.

The inputs and outputs of `Tasks` are `PipelineResources`. There are a few types of pipeline resources that are baked into tekton, for example the `git` and `image`. The `git` resource is (as you'd expect) how we can get source code into our pipelines. The `image` resource allows us to either use or create a Docker image.

1. Create a `git` resource to clone the Skaffold project. We'll use a sample application from there to demonstrate how a pipeline is stitched together.

    ```
    kubectl apply -f resources/git-skaffold.yaml
    ```

1. Next, we'll create an `image` resource for the image we'll build from the sample application.

    ```
    export GCP_PROJECT=$(gcloud config get-value project)
    sed -i s/GCP_PROJECT/$GCP_PROJECT/ resources/image-leeroy-web.yaml
    kubectl apply -f resources/image-leeroy-web.yaml
    ```

1. Now that our resources have been created, lets create a `Task` that clones a git repo as an input
   and builds an image from it as an output.

   ```
   kubectl apply -f tasks/build-docker-image-from-git.yaml
   ```

//TODO diagram here that shows the mapping of these resources

1. Next we'll create the `TaskRun` that will map our resources to the task we just created.

    ```
    kubectl apply -f taskruns/leeroy-web-image-build.yaml
    ```
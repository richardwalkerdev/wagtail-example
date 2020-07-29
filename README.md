## Basic Wagtail Example for OpenShift Pipeline

This repo is for testing OpenShift Pipelines. It uses a Django based Wagtail CMS app, nothing more that a `wagtail start example .`.

First attempts at deploying this failed due to `django.core.exceptions.ImproperlyConfigured` SQLite versions found in the standard s2i-python36 image. 
The solution to this was to build a s2s-python38 image from https://github.com/sclorg/s2i-python-container.git and add a tweaked cluster task to use my newer image.

The OpenShift pipeline process was taken and adapted from https://docs.openshift.com/container-platform/4.5/pipelines/creating-applications-with-cicd-pipelines.html 

However, it took quite some effort to get to understanding and having a working build pipeline using something other that the typical Java or Node.js examples. This exercise has served me well as a start-for-ten, using S2I and OpenShift Pipeline for current Python projects. 

### Building Python from sclorg

SImply clone and build a custom `s2i-python38-container-img` and push it to your own public repository:

```text
git clone https://github.com/sclorg/s2i-python-container.git
```

```text
cd s2i-python-container/3.8/
```

```text
cp Dockerfile.rhel8 Dockerfile
```

```text
buildah bud -t richardwalker.dev/s2i-python38-container-img .
```

```text
skopeo copy containers-storage:richardwalker.dev/s2i-python38-container-img docker://quay.io/richardwalkerdev/s2i-python38-container
```

### OpenShift Pipelines

Time of writing was with OCP 4.5, OpenShift pipeline can be install via the web consoles Operator hub or:

```text 
vi sub.yaml
```
```yaml
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: openshift-pipelines-operator
  namespace: openshift-operators
spec:
  channel: ocp-4.4
  name: openshift-pipelines-operator-rh 
  source: redhat-operators 
  sourceNamespace: openshift-marketplace
```

```text
oc apply -f sub.yaml
```

### k8s

The directory `k8s` in this repository includes three yaml files for an OpenShift deployment, service and route.

Note that the custom s2i image by sclorg uses port 8080 (not the usual 8000 in Django world).

Also note that the deployment name (found in deployment.yaml) is important for the pipeline run!

Also note the spec/containers/image (found in deployment.yaml) name is deliberately filled with place holders, this is because it cost me two days understand that that image name needs to be over written with a patching step in the pipeline to enable the pipeline to remain generic.

### Tekton/OpenSHift Pipeline

YAML files found under openshift-pipeline are all the pipeline "kinds" needed to build a pipeline.

Create a new project to work in:

```text
oc new-project wagtail-pipeline
```

### Resources

Create a pipeline resource that holds the reference/parameter (in this case) to _this_ git repository:

```text
oc create -f tkn_git_resource.yaml
```

Create a pipeline resource that holds a URL to the OpenShift internal image registry, this URL is the image that the pipeline build step will build. 
**IT IS ABSOLUTLY IMPORTANT** that the path includes the OpenShift project name you're currently inside and adding the pipeline within!!! In this example I'm using `wagtail-pipeline` the project name created with `oc new-project wagtail-pipeline`, when creating other projects this needs to be changes accordingly. 

```
   value: 'image-registry.openshift-image-registry.svc:5000/<OPENSHIFT_PROJECT_NAME>/wagtail-img:latest'
```

```text
oc create -f tkn_image_resource.yaml
```

```
tkn resource list
```

##### Install `tkn` command

```text
sudo dnf copr enable chmouel/tektoncd-cli
sudo dnf install tektoncd-cli -y
```

### Custom Cluster Task

As I said, the standard s2i image available is out-of-date, so I copied and modified the existing `ClusterTask` to use my `quay.io/richardwalkerdev/s2i-python38-container` image. 

Only two line altered:

```yaml
spec:
  params:
    - default: '8'
```

```yaml
steps:
    - command:
        - s2i
        - build
        - $(params.PATH_CONTEXT)
        - quay.io/richardwalkerdev/s2i-python3$(params.MINOR_VERSION)-container
```


```text
oc create -f tkn_cluster_task.yaml
```

### Tasks

These are carbon copies of the tasks from https://docs.openshift.com/container-platform/4.5/pipelines/creating-applications-with-cicd-pipelines.html 

The `apply_manifest.yaml` uses the `oc` command to apply the YAML files under `k8s` in this repo.

```text
oc create -f tkn_apply_manifests_task.yaml
```

The `tkn_update_deplyment_image_task.yaml` uses the `oc` command to patch the deployment to use the container built in the `s2i-build-image` task, which should equal the URL as defined in the image resource defined earlier e.g. `image-registry.openshift-image-registry.svc:5000/wagtail-pipeline/wagtail-img:latest`

```text
oc create -f tkn_update_deplyment_image_task.yaml
```

### Pipeline

Finally, add the pipeline its self which includes three steps, the `s2i-build-image` that uses the custom Cluster Task added before which takes the source code from the git repository resource and builds an image, the `apply-manifests` which applies the deployment, service and route under `k8s` and the `update-deployment` which patches the deployment to use the the image built in step 1.

```text
oc create -f tkn_pipeline.yaml
```

### Running it

IMPORTANT: The `deployment-name` parameter **MUST** match the name of the deployment as defined in `k8s/deployment.yaml`.

The following command kicks of the pipeline using the resources defined earlier and specifying the correct deployment name:

```text
tkn pipeline start build-and-deploy -r git-repo-resource=wagtail-example-git-resource -r target-image-resource=wagtail-example-image-resource -p deployment-name=wagtail-example
```

It will tell you how to tail the logs from the cli using the `tkn` command, it's self-intuitive to use the web console.


### Screen shots


![Image of Pipeline](screenshots/pipeline.png?raw=true)
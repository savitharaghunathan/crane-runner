Stateful Application Migration
==============================

In this tutorial you will migrate a simple stateful application, 
[PHP Guestbook application](https://kubernetes.io/docs/tutorials/stateless-application/guestbook/)
modified so that the redis-leader and redis-follower use persistent storage via
a pair of PersistentVolumeClaims.

# Roadmap

* Deploy Guestbook application in "source" cluster.
* Prepare for application migration.
* Add data to the application.
* Migrate application data with `transfer-pvc`.
* Migrate application to "destination" cluster via
    [Tekton PipelineRun](https://tekton.dev/docs/pipelines/pipelineruns/).
* Verify application data was migrated.

# Before you begin

You will need a "source" and "destination" Kubernetes cluster with Tekton and
the Crane Runner ClusterTasks installed. Below are the steps required for easy
copy/paste:

```bash
# Start up "source" and "destination" clusters in minikube
curl -s "https://raw.githubusercontent.com/konveyor/crane/main/hack/minikube-clusters-start.sh" | bash

# Install Tekton
# See https://tekton.dev/docs/getting-started/ for help with installing Tekton
kubectl --context dest apply -f "https://storage.googleapis.com/tekton-releases/pipeline/latest/release.yaml"
kubectl --context dest --namespace tekton-pipelines wait --for=condition=ready pod --selector=app.kubernetes.io/component=controller --timeout=180s

# Install Crane Runner manifests
kustomize build github.com/konveyor/crane-runner/manifests | kubectl --context dest apply -f -
```

# Deploy Guestbook application in "source" cluster

You will be deploying
[Kubernetes' stateless guestbook application](https://kubernetes.io/docs/tutorials/stateless-application/guestbook/)
modified for this scenario to mount persistent storage to both the redis leader
and follower. The guestbook application consisists of:

* redis leader deployment + PVC and service
* redis follower deployment + PVC and service
* guestbook front-end deployment and service


```bash
kubectl --context src create namespace guestbook
kustomize build github.com/konveyor/crane-runner/examples/resources/guestbook-persistent | kubectl --context src --namespace guestbook apply -f -
kubectl --context src --namespace guestbook wait --for=condition=ready pod --selector=app=guestbook --timeout=180s
```

**NOTE** If you previously deployed the guestbook application, it may be missing
PVCs. At a minimum, you should run:

```bash
kustomize build github.com/konveyor/crane-runner/examples/resources/guestbook-persistent | kubectl --context src --namespace guestbook apply -f -
```

This will create two PVCs bound to the redis leader and follower deployments.

# Prepare for Application Migration

First, create the `guestbook` namespace in the "destination" cluster
where you will migrate the guestbook application from the "source" cluster.

```bash
kubectl --context dest create namespace guestbook
```

You must upload your kubeconfig as a secret. This will be used by the
ClusterTasks to migrate the application.
```bash
kubectl config view --flatten | kubectl --context dest --namespace guestbook create secret generic kubeconfig --from-file=config=/dev/stdin
```

# Add Data

Expose the application's frontend.

```bash
kubectl --context src --namespace guestbook port-forward svc/frontend 8080:80
```

**NOTE** This process runs in the foreground.

Then navigate to localhost:8080 from your browser and add messages to the
guestbook.


# Migrate Application Data

```bash
cat <<EOF | kubectl --context dest --namespace guestbook create -f -
apiVersion: tekton.dev/v1beta1
kind: PipelineRun
metadata:
  generateName: stateful-app-migrate-
spec:
  pipelineSpec:
    workspaces:
    - name: kubeconfig
    tasks:
    - name: redis-data01
      taskRef:
        name: crane-transfer-pvc
        kind: ClusterTask
      params:
      - name: src-context
        value: src
      - name: dest-context
        value: dest
      - name: dest-namespace
        value: guestbook
      - name: pvc-name
        value: redis-data01
      - name: endpoint-type
        value: nginx-ingress
      workspaces:
      - name: kubeconfig
        workspace: kubeconfig
    - name: redis-data02
      runAfter:
      - redis-data01
      taskRef:
        name: crane-transfer-pvc
        kind: ClusterTask
      params:
      - name: src-context
        value: src
      - name: dest-context
        value: dest
      - name: dest-namespace
        value: guestbook
      - name: pvc-name
        value: redis-data02
      - name: endpoint-type
        value: nginx-ingress
      workspaces:
      - name: kubeconfig
        workspace: kubeconfig
  workspaces:
  - name: kubeconfig
    secret:
      secretName: kubeconfig
EOF
```

# Migrate Application Workloads

```bash
kubectl --context dest --namespace guestbook create -f "https://raw.githubusercontent.com/konveyor/crane-runner/main/examples/005_stateful-app-migrate/pipelinerun.yaml"
```

# Did it Work?

Expose the application's frontend.

```bash
kubectl --context dest --namespace guestbook port-forward svc/frontend 8080:80
```

**NOTE** This process runs in the foreground.

Now you can navigate to localhost:8080 in your browser and verify that the
messages were maintained during migration.
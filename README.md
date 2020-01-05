# Presentation

A tekton tutorial project to build and deploy application based on OpenShift/Kubernetes manifests 

# PRE-REQUISITES
- Use a OCP 4 cluster
- Connect as cluster admin user

# Install Tekton pipelines
<pre>
oc new-project tekton-pipelines
oc adm policy add-scc-to-user anyuid -z tekton-pipelines-controller
oc apply --filename https://storage.googleapis.com/tekton-releases/latest/release.yaml
# Wait until all pods are running
oc get pods --namespace tekton-pipelines
</pre>

# Install Tekton dashboard
<pre>
oc apply -f https://github.com/tektoncd/dashboard/releases/download/v0.3.0/dashboard-latest-release.yaml
oc expose svc tekton-dashboard
</pre>

# Configure Tekton pipelines
## Create PVC to share tekton resources : DOESNT WORK YET - skip
<pre>
cat << EOF | oc apply -f -  
apiVersion: v1
kind: ConfigMap
metadata:
  name: config-artifact-pvc
data:
	storageClassName: gp2
EOF
</pre>

## Override default service account for Task and pipeline Run : DOESNT WORK YET - skip
<pre>
cat << EOF |  oc apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: config-defaults
data:
  default-service-account: "tekton"
EOF
</pre>

# Create openshift manifests
## Create namespaces
<pre>
oc new-project saberkan-dev
oc new-project saberkan-prod
</pre>

## Create deployment configs
<pre>
oc create -f resources/manifests/dc-sb-dev.yaml -n saberkan-dev
oc create -f resources/manifests/dc-sb-prod.yaml -n saberkan-prod
</pre>

## Create svc
<pre>
oc create -f resources/manifests/svc-sb.yaml -n saberkan-dev
oc create -f resources/manifests/svc-sb.yaml -n saberkan-prod
</pre>

## Create route
<pre>
oc expose svc/sb -n saberkan-dev
oc expose svc/sb -n saberkan-prod
</pre>

# Create build & release pipeline
## Create build & push task
### Git input resource
<pre>
cat << EOF | oc create -f -
apiVersion: tekton.dev/v1alpha1
kind: PipelineResource
metadata:
  name: sb-git
spec:
  type: git
  params:
  - name: url
    value: https://github.com/saberkan/sb-project.git
EOF
 </pre>
### Image output resource
<pre>
cat << EOF | oc create -f -
apiVersion: tekton.dev/v1alpha1
kind: PipelineResource
metadata:
  name: sb-image
spec:
  type: image
  params:
  - name: url
    value: image-registry.openshift-image-registry.svc:5000/openshift/sb
EOF
</pre>

### Create build & push task
<pre>
oc create -f resources/pipeline/task.yaml
</pre>

### Allow default user to push into registry (could be tekton sa, but config map doesnt work)
<pre>
# Role need to be adapted, using cluster-admin for demo purpose
oc adm policy add-cluster-role-to-user cluster-admin -z default
</pre>

### Create  build & push task run to put resources and task together and run the task: Use only to test task, later task will run with pipelinerun
<pre>
oc create -f resources/pipeline/taskrun_build_push.yaml
</pre>

## Create release task
<pre>
oc create -f resources/pipeline/task_release.yaml
</pre>

## Create release task run : Use only to test task, later task will run with pipelinerun
<pre>
oc create -f resources/pipeline/taskrun_release.yaml
</pre>

## Create pipeline
<pre>
oc create -f resources/pipeline/pipeline_build_release.yaml
</pre>

## Create pipelinerun
<pre>
oc create -f resources/pipeline/pipelinerun_build_release.yaml
</pre>

# Creating task for promoting image
PS: This step could be included on the first pipeline, but currently asking for approval is not implemented yet within tekton.
https://github.com/tektoncd/pipeline/issues/233
## Create promote task
<pre>
oc create -f resources/pipeline/task_promote.yaml
</pre>

## Create promote task run : Use only to test task, later task will run with pipelinerun
<pre>
oc create -f resources/pipeline/taskrun_promote.yaml
</pre>


# TO GO FURTHER: 
## Create pipelinewebhooks using listeners and triggers
https://github.com/tektoncd/triggers/tree/master/examples

## Use cli tkt
https://github.com/tektoncd/cli





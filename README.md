# Presentation

A tekton tutorial project to build and deploy application into production based on OpenShift/Kubernetes manifests 

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

# Configure Tekton pipelines
## Create PVC to share tekton resources
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

## Override default service account for Task and pipeline Run
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

# Create pipeline
## Create build & push pipeline resources
### Git input resource
<pre>
cat << EOF | oc create -f -
apiVersion: tekton.dev/v1alpha1
kind: PipelineResource
metadata:
  name: sb-git-build-push-kaniko
spec:
  type: git
  params:
  - name: revision
    value: master
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
  name: sb-build-push-kaniko
spec:
  type: image
  params:
  - name: url
    value: image-registry.openshift-image-registry.svc:5000/openshift/sb
EOF
</pre>

## Create build & push task
<pre>
copy content in file: tmp.yaml # otherwise bad substitution when using cat << EOF
apiVersion: tekton.dev/v1alpha1
kind: Task
metadata:
  name: sb-build-push-kaniko
spec:
  inputs:
    resources:
    - name: git-resource
      type: git
    params:
    - name: pathToDockerFile
      description: The path to the dockerfile to build
      default: /workspace/workspace/Dockerfile
    - name: pathToContext
      description: The build context used by Kaniko (https://github.com/GoogleContainerTools/kaniko#kaniko-build-contexts)
      default: /workspace/workspace
  outputs:
    resources:
    - name: image-resource
      type: image
  steps:
  - name: build-and-push
    image: gcr.io/kaniko-project/executor:v0.13.0
    # specifying DOCKER_CONFIG is required to allow kaniko to detect docker credential
    env:
    - name: "DOCKER_CONFIG"
      value: "/tekton/home/.docker/"
    args:
    - --dockerfile=$(inputs.params.pathToDockerFile)
    - --destination=$(outputs.resources.image-resource.url)
    - --context=$(inputs.params.pathToContext)
    - --oci-layout-path=$(inputs.resources.image-resource.path)
    securityContext:
      runAsUser: 0
  sidecars:
    - image: registry
      name: registry

oc create -f tmp.yaml && rm tmp.yaml
</pre>

## Create task build & push task run to put resources and task together and run the task
<pre>
cat << EOF | oc create -f -
apiVersion: tekton.dev/v1alpha1
kind: TaskRun
metadata:
  name: sb-build-push-kaniko
spec:
  taskRef:
    name: build-push-kaniko
  inputs:
    resources:
    - name: git-resource
      resourceRef:
        name: sb-git-build-push-kaniko
    params:
    - name: pathToDockerFile
      value: Dockerfile
    - name: pathToContext
      value: /workspace/workspace/sb-project
  outputs:
    resources:
    - name: image-resource
      resourceRef:
        name: sb-build-push-kaniko
EOF
</pre>





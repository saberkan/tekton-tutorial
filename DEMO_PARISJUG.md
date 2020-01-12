Demo - Paris JUG

Tekton route : 
tekton-dashboard-tekton-pipelines.apps.cluster-95e5.95e5.sandbox1356.opentlc.com

1 - Create OpenShift rousources
<pre>
# Create namespace
oc new-project demo

# Create deployment config
oc create -f resources/manifests/dc-sb-dev.yaml -n demo

# Create route & service
oc create -f resources/manifests/svc-sb.yaml -n demo
oc expose svc/sb -n demo
</pre>

2 - Create tekton resources
<pre>
oc project tekton-pipelines 

# Create tasks
oc create -f resources/pipeline/task_build_push.yaml
oc create -f resources/pipeline/task_release.yaml

# Create pipeline
oc create -f resources/pipeline/pipeline_build_release.yaml

# Create trigger template
oc create -f resources/triggers/sb-template.yaml

# Create trigger binding
oc create -f resources/triggers/sb-binding.yaml

# Create event listener
oc create -f resources/triggers/sb-eventlistener.yaml

# Expose listener
svc=$(oc get svc -l eventlistener=sb-eventlistener --no-headers | awk '{print $1}')
oc expose svc/$svc
route=$(oc get route $svc --no-headers | awk '{print $2}')

</pre>

3 - Run pipline
version=0.0.1-trigger
curl -X POST \
  http://$route \
  -H 'Content-Type: application/json' \
  -d '{
		"git-repo": "https://github.com/saberkan/sb-project.git",
		"image-url": "image-registry.openshift-image-registry.svc:5000/openshift/sb"
		"tag": "'$version'"
     }'
</pre>


4 - Check application on browser
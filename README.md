# Pod Autoscaling

## Overview:

Kubernetes enables autoscaling at the cluster/node level as well as at the pod level, two different but fundamentally connected layers of Kubernetes architecture. It has three scalability tools. Two of these, the Horizontal pod autoscaler (HPA) and the Vertical pod autoscaler (VPA), function on the application abstraction layer while the cluster autoscaler works on the infrastructure layer.

## Horizontal Pod Autoscaler (HPA)
HPA scales the number of pods in a replication controller, deployment, replica set or stateful set based on the observed metrics such as average CPU utilization, average memory utilization or any other custom metric to the target specified by the user.

More details about the algorithm and architecture https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/#algorithm-details

### Points to consider while deploying HPA

- Install metrics-server - Tanzu Kubernetes Cluster already has a metrics server installed
- Ensure all pods have resource requests specified. If some of the pod's containers do not have the relevant resource request set, CPU utilization for the pod will not be defined and the autoscaler will not take any action for that metric
Memory requests- 350Mi
CPU requests - 200m
- This document takes Guestbook app as an example.  Please make sure you have guestbook app up and running in a guestbook namespace.

### Deploying HPA for demo guestbook app
- Login to Tanzu Kubernetes workload cluster and set the cluster context. Make sure guestbook app is installed and running
<pre>
kubectl get pods -n guestbook
</pre>
- Run following command to deploy HPA
<pre>
kubectl autoscale deployment guestbook-frontend-deployment --min=1 --max=10 --cpu-percent=50 -n guestbook
</pre>
- Watch HPA object created
<pre>
Kubectl get hpa -n guestbook —watch
</pre>
- Open another tab for command window, login to tanzu kubernetes workload cluster and set the cluster context.  Run the load generator for guestbook app
<pre>
kubectl run -it --rm load-generator --image=busybox /bin/sh

while true; do wget -q -O- http://<your-ip>; done
</pre>
- Switch to previous tab and see HPA o/p, after few mins you should see load incresing and deployment scaled
<pre>
NAME                            REFERENCE                                        TARGET      MINPODS   MAXPODS   REPLICAS   AGE
guestbook-frontend-deployment   Deployment/guestbook-frontend-deployment/scale   305% / 50%  1         8   
</pre>
- After few mins stop the load generator with CTRl+C and exit.  Make sure load generator is exited.  You should see HPA scaling down the deployment.  This will take few mins.
<pre>
NAME                            REFERENCE                                        TARGET      MINPODS   MAXPODS   REPLICAS   AGE
guestbook-frontend-deployment   Deployment/guestbook-frontend-deployment/scale   0% / 50%  1         1   
</pre>

## Verital Pod Autoscaler (VPA)
### Overview
Vertical Pod autoscaling is an autoscaling tool to help size Pods for the optimal CPU and memory resources required by the Pods. The VPA automatically computes historic and current CPU and memory usage for the containers in those pods and uses this data to determine optimized resource limits and requests to ensure that these pods are operating efficiently at all times. The VPA automatically deletes any pods that are out of alignment with its recommendations one at a time, so that your applications can continue to serve requests with no downtime. The workload objects then re-deploy the pods with the original resource limits and requests. The VPA uses a mutating admission web-hook to update the pods with optimized resource limits and requests before the pods are admitted to a node. If you do not want the VPA to delete pods, you can view the VPA resource limits and requests and manually update the pods as needed.The target reference allows us to specify which workload is subject to the actions of the VPA, and for our use-case we have a deployment eg.shopping-blue . It could be any of Deployment, DaemonSet, ReplicaSet, StatefulSet, ReplicationController, Job, or CronJob.

### Components of vpa:

- Recommender:
The recommender as its name suggests, assesses the current and historical workload metrics, and based on what it observes makes recommendations for a workload's resource requests. It does this in conjunction with any limit ranges set, and the resource policy set in the relevant VPA API object.
- Updater:
The job of the updater is to respond to the recommendations based on the update policy defined in the VPA API object. If a workload's pods need updating according to the recommendations, the updater will evict the pods whilst accounting for any governing pod disruption budget.
- Admission controller:
It mutates the resource requests values for the containers that comprise the workload. This it does in line with the recommendations it finds in the corresponding VPA object, and it annotates the pods accordingly.

### Install VPA Custom Resource
- Tanzu does not ship with VPA but you can install VPA custom resource.
<pre>
git clone https://github.com/kubernetes/autoscaler.git
git checkout bb860357f691313fca499e973a5241747c2e38b2
cd autoscaler/vertical-pod-autoscaler
./hack/vpa-process-yamls.sh print
./hack/vpa-up.sh
</pre>
- make sure VPA is installed and running
<pre>
kubectl api-resources | grep autoscaling
</pre>

### Deploy VPA for guestbook app
- Create VPA resource for guestbook app
<pre>
  apiVersion: autoscaling.k8s.io/v1
  kind: VerticalPodAutoscaler
  metadata:
    name: guestbook-frontend-deployment
  spec:
    targetRef:
      apiVersion: "apps/v1"
      kind:       Deployment
      name:       guestbook-frontend-deployment
    updatePolicy:
      updateMode: "Off
</pre>
<pre>
Kubectl apply -f vpa.yaml -n guestbook
</pre>
- Get the VPA and describe it
<pre>
kubectl describe vpa guestbook-frontend-deployment -n guestbook
</pre>
- VPA updateMode is set to off, it will provide recommendations for pod requests and limits and won't apply those.
  - The lower bound is the minimum estimation for the container.
  - The upper bound is the maximum recommended resource estimation for the container.
  - Target estimation is the one we will use for setting resource requests.
  - All of these estimations are capped based on min allowed and max allowed container policies.
  - The uncapped target estimation is a target estimation produced if there were    no minAllowed and maxAllowed restrictions.
- Be very careful when turning on VPA updateMode, it will restart pods to match target resource limits.
- VPA is good for getting recommendations for pod resource limits

## References
https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/




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






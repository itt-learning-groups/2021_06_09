# Kubernetes "Lab" Exercise: Explore the EKS cluster + service we created

## June 9, 2021

## Prerequisites

* Set up an EKS cluster on AWS with a simple deployed `deployment` and `service` as described [here](https://github.com/us-learn-and-devops/2021_05_26/blob/main/README.md) in last week's group exercise.

## Lab

### Explore the cluster's fault tolerance

* Use the AWS console to find your cluster's worker nodes. There should be 3 of them.
* Use kubectl to list the nginx-svc pods (`kubectl get pods`). Then "describe" one of them, e.g. `kubectl describe pod nginx-deployment-75ffc6d4d-7b8xg` to find its node assignment from the k8s Scheduler. Find the EC2 worker node this corresponds to.
* Also use kubectl to set a "watch" on the nginx-svc pods: `kubectl get pods --watch`. (BTW, note the "watch" mechanism in use here is the same one we've seen as so important for various internal components of k8s that "watch" the API server for changes.)
* Terminate that node you identified. Watch what happens to...
  * The worker-node list in the AWS console.
  * The list of pods.
* It will take a few minutes to see what happens. Be patient. What does k8s do? For which parts of what you observe is the k8s "control plane" responsible? For which parts is a worker-node `kubelet` responsible?
* Check out Kubernetes docs on the [Pod Lifecycle](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/). Which pod phase statuses and/or container states did you observe? What is pod "termination"?
* Use kubectl to manually delete a pod, e.g. `kubectl delete pod nginx-deployment-75ffc6d4d-7b8xg`. What happens? Did pod termination happen?

### Explore the `Service` object and service discovery

* Run `kubectl get svc` (or `kubectl get svc -n default` since we're working in the "default" namespace here).
  * Note that both services listed *have* a `CLUSTER-IP`, but the "kubernetes" service, which is of `TYPE` "ClusterIP", does not also have an `EXTERNAL-IP`, while our "nginx-svc" that we just deployed as a "LoadBalancer" type service *does* have an `EXTERNAL-IP`.
  * Attempt to load first the nginx-svc external-IP, then its cluster-IP, in a browser. In one case, you'll get a "Welcome to nginx!" default home-page which means you got through to your web service. In the other case, you'll get an error.
* Use your AWS console to find the load balancer corresponding to that external-IP. (This will be listed in the "EC2 Dashboard" under "Load Balancing", and it will be just a basic "Classic" load balancer rather than a more up-to-date application load balancer or network load balancer like we've looked at before.)
  * How is the the "LoadBalancer" type service using this Classic load balancer to direct a network request from ***outside*** the k8s cluster to the nginx-svc ***inside*** the cluster?
  * Check out Kubernetes docs on [ServiceTypes](https://kubernetes.io/docs/concepts/services-networking/service/#publishing-services-service-types):
    * Understand the difference between ClusterIP (the default), NodePort, and LoadBalancer service types.
    * Understand how a LoadBalancer service still *requires* and uses a cluster-IP.
* Use the nginx-svc cluster-IP to connect to (one of the pods in) the service using the "port-forwarding" trick:
  * Find the name of one of the service pods. Try using the pod label "env: dev" via `kubectl get pods -l env=dev` or `kubectl get pods --selector="env=dev"`. (Note, BTW, how this illustrates k8s labels and label selectors in action.)
  * Port-forward localhost:8080 to your selected pod, e.g. `kubectl port-forward nginx-deployment-75ffc6d4d-lv2h7 8080:80`. (Question: Why use port 80 on the right-hand side?)
  * Load localhost:8080 in your browser. You should see the "Welcome to nginx!" home-page again.
  * Turn off port-forwarding via Ctrl+C.
  * Refresh your browser. Now the home-page is gone. But the nginx-svc external-IP still works.
* Explore service-discovery mechanisms:
  * Use an "exec" command to get a terminal session in one of the nginx-svc pods, e.g. `kubectl exec -it nginx-deployment-75ffc6d4d-lv2h7 -- bash`
  * Print out the environment variables: `env`. Focus on the ones for the NGINX_SVC: `env | grep NGINX`.
  * Check out Kubernetes docs on Discovering services to see how [Environment variables](https://kubernetes.io/docs/concepts/services-networking/service/#environment-variables) can be used for service discovery.
  * Let's see if service discovery is working. Install some basic networking tools in the pod container you've "exec'd" into. This is an Ubuntu-based container, so we'll use `apt`:

        apt update
        apt install dnsutils

    * Our nginx-svc manifest file names the service "nginx-svc" so let's see if we can find it that way: `nslookup nginx-svc`.
    * Compare the IP you get  to the one stored in the NGINX_SVC_PORT_80_TCP_ADDR environment variable and to the cluster-IP reported by the earlier `kubectl get svc` command.
    * Note that `nslookup nginx-svc.default` also works. What does "default" mean here?
    * OK. But is service discovery working because we have those NGINX environment variables set, or because our EKS cluster has a DNS service?
      * Run `kubectl get pods -n kube-system | grep dns` to see if our "system" namespace has a DNS service running.
      * Check out Kubernetes docs on Discovering services to see how [DNS](https://kubernetes.io/docs/concepts/services-networking/service/#dns) can be used for service discovery.
      * Re-connect with that container terminal (via "exec" command) and `unset` all the NGINX_... environment variables. 
      * Does this break our service discovery? Try using `nslookup` again.

# SRE Fundamentals with Google

## Week 3 Project: Dealing with Toil at UpCommerce
Hi!

### Problem 1 : Swype.com Downtime
**Description:** UpCommerce, relying on Swype.com as its payment gateway provider, faced significant downtime. During these periods payments in queue did not roll back, leading to customer claims against UpCommerce. The Engineering Lead tasked the SRE team with preventing such incidents from recurring. To address this issue, two solutions were proposed:

 1. **Code Refactoring:** The team proposed pulling out the payment system implementation from UpCommerce's original implementation and transforming it into a microservice. The proposed implementation details are already captured in the swype-deployment.yml and swype-service.yml files.
 2. **Fail-Safe Mechanism:** Another solution involves implementing a "big red button" service. This service would shut down Swype's payment service in UpCommerce's cluster if it exhibited erratic behavior.

The ticketing system for alert management had been a major point of contention among engineers. Complaints included recurring obsolete issues and a lack of clear prioritization for incoming issues.

#### Task
Implement a "big red button" for UpCommerce by creating a bash script to monitor the Kubernetes deployment dedicated to the Swype microservice defined in swype-deployment.yml. The script should be written in the empty watcher.sh file in the task repo, and trigger if the pod restarts due to network failure more than three times. Here's an optional, pseudocode hint of the tasks your watcher.sh file should perform:

> 1. Define Variables: Set the namespace, deployment name, and maximum number of restarts allowed before scaling down the deployment.
> 2. Start a Loop: Begin an infinite loop that will continue until explicitly broken.
> 3. Check Pod Restarts: Within the loop, use the kubectl get pods command to retrieve the number of restarts of the pod associated with the specified deployment in the specified namespace.
> 4. Display Restart Count: Print the current number of restarts to the console.
> 5. Check Restart Limit: Compare the current number of restarts with the maximum allowed number of restarts.
> 6. Scale Down if Necessary: If the number of restarts is greater than the maximum allowed, print a message to the console, scale down the deployment to zero replicas using the kubectl scale command, and break the loop.
> 7. Pause: If the number of restarts is not greater than the maximum allowed, pause the script for 60 seconds before the next check.
> 8. Repeat: After the pause, the script goes back to step 3. This process repeats indefinitely until the number of restarts exceeds the maximum allowed, at which point the deployment is scaled down and the loop is broken.

#### Solution
When running the script, it was noticed the swype service wasn't running

    /workspaces/sre-week-three (Changes) $ kubectl get all -n sre
    NAME                                  READY   STATUS    RESTARTS   AGE
    pod/swype-app-5574bf65b4-hsq6b        0/1     Pending   0          2m10s
    pod/upcommerce-app-8698d8bd45-tsxdb   1/1     Running   0          2m11s
    
    NAME                             READY   UP-TO-DATE   AVAILABLE   AGE
    deployment.apps/swype-app        0/1     1            0           2m10s
    deployment.apps/upcommerce-app   1/1     1            1           2m11s
    
    NAME                                        DESIRED   CURRENT   READY   AGE
    replicaset.apps/swype-app-5574bf65b4        1         1         0       2m10s
    replicaset.apps/upcommerce-app-8698d8bd45   1         1         1       2m11s
   

    /workspaces/sre-week-three (Changes) $ kubectl describe pod/swype-app-5574bf65b4-hsq6b -n sre
    Name:             swype-app-5574bf65b4-hsq6b
    Namespace:        sre
    Priority:         0
    Service Account:  default
    Node:             <none>
    Labels:           app=swype-app
                      pod-template-hash=5574bf65b4
    Annotations:      <none>
    Status:           Pending
    IP:               
    IPs:              <none>
    Controlled By:    ReplicaSet/swype-app-5574bf65b4
    Containers:
      stripe:
        Image:      uonyeka/stripe:linux-amd64
        Port:       5003/TCP
        Host Port:  0/TCP
        Limits:
          cpu:     1
          memory:  4Gi
        Requests:
          cpu:        1
          memory:     4Gi
        Environment:  <none>
        Mounts:
          /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-9s962 (ro)
    Conditions:
      Type           Status
      PodScheduled   False 
    Volumes:
      kube-api-access-9s962:
        Type:                    Projected (a volume that contains injected data from multiple sources)
        TokenExpirationSeconds:  3607
        ConfigMapName:           kube-root-ca.crt
        ConfigMapOptional:       <nil>
        DownwardAPI:             true
    QoS Class:                   Guaranteed
    Node-Selectors:              <none>
    Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                                 node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
    Events:
      Type     Reason            Age                  From               Message
      ----     ------            ----                 ----               -------
      Warning  FailedScheduling  17s (x2 over 5m47s)  default-scheduler  0/1 nodes are available: 1 Insufficient cpu, 1 Insufficient memory. preemption: 0/1 nodes are available: 1 No preemption victims found for incoming pod..

Noticed swype-deployment.yml was set at it's limit so I modified it to a lesser amount (see [commit](https://github.com/jasoncstrong/sre-week-three/commit/f8142e9f4a641f3ad5348da86fd8101c5c8855ec)). After saving and redeploying, the swype-app service was now up and running.

    /workspaces/sre-week-three (Changes) $ kubectl get deployment -n sre
    NAME             READY   UP-TO-DATE   AVAILABLE   AGE
    swype-app        1/1     1            1           14m
    upcommerce-app   1/1     1            1           14m

From running the watcher.sh script, it worked as expected (once it hits max # restarts, scale down swype-app) :

(Note - I noticed there was no timestamp on nohup.out so I added it to watcher.sh (see [commit](https://github.com/jasoncstrong/sre-week-three/commit/8597dc311ac92c6abf6fee9251ae22a7d7c70121)))

     /workspaces/sre-week-three (Changes) $ tail -f nohup.out 
    Mon Apr 19 12:03:52 UTC 2024 Current number of restarts: 4
    Mon Apr 19 12:04:52 UTC 2024 Current number of restarts: 4
    Mon Apr 19 12:05:52 UTC 2024 Current number of restarts: 4
    Mon Apr 19 12:06:52 UTC 2024 Current number of restarts: 5
    Mon Apr 19 12:06:52 UTC 2024 Maximum number of restarts exceeded. Scaling down the deployment...
    deployment.apps/swype-app scaled

     /workspaces/sre-week-three (Changes) $ kubectl get deployment -n sre
    NAME             READY   UP-TO-DATE   AVAILABLE   AGE
    swype-app        0/0     0            0           26m
    upcommerce-app   1/1     1            1           26m

### Problem 2 : Ticketing System Challenges
Description: The ticketing system for alert management had been a major point of contention among engineers. Complaints included recurring obsolete issues and a lack of clear prioritization for incoming issues.

#### Task
Identify potential solutions or products, whether free or commercial, to address the toil in the ticketing system. These solutions should aim to mitigate issues such as recurring obsolete alerts and lack of prioritization. Create a markdown file and fill these solutions in your markdown file.

### Potential Solutions

1.  **New Relic** - [\[Price\]](https://newrelic.com/pricing) New Relic allows websites and mobile apps to track user interactions and service operators' software and hardware performance.
2.  **PagerDuty** - [\[Price\]](https://www.pagerduty.com/pricing/incident-management/) PagerDuty is a SaaS incident response platform for IT departments. Its platform is designed to alert clients to disruptions and outages.
3.  **Graylog** - [\[Price\]](https://graylog.org/pricing/) Graylog offers alert aggregation and deduplication features to simplify incident response.
4.  **Datadog** - [\[Price\]](https://www.datadoghq.com/pricing/) Datadog provides an observability and security SaaS platform for cloud applications.
5.  **Prometheus** - [\[FREE\]](https://prometheus.io/) An open-source monitoring system with a dimensional data model, flexible query language, efficient time series database and modern alerting approach.
6.  **JIRA** - [\[Price\]](https://www.atlassian.com/software/jira/pricing) Customizable Issue tracking software with querying and integration built in.
7.  **Sentry** - [\[Price\]](https://sentry.io/pricing) Self-hosted and cloud-based application performance monitoring & error tracking that helps software teams see clearer, solve quicker, & learn continuously.

# k8s HPA tests

Demo repository that shows HPA behaviour on deployments with proper `selector.matchLabels` and without.

It contains two deployment files, each two deployment objects:

- One, which consumes zero CPU (`myapp-no-cpu`). The average CPU usage will be 0%.
- Another, which consumes all available CPU (`myapp-full-cpu`). The average CPU usage will be 100%.

It also contains a HPA object that scales the pods if average CPU usage of the deployment that uses no CPU (`myapp-no-cpu`) reaches 40% (which will never happen).

# Setup

You will need a K8s cluster with metrics server. You can use [KiND](https://github.com/kubernetes-sigs/kind).

This repo provides a script to [create the metrics server in KiND](./create-metrics-server.sh), in case you need to create one.

# Testing a proper deployment

The deployment with proper label selector will correctly select their pods. This means that HPA is able to correctly calculate the CPU metrics for the pods, and thus determine that no scaling is required.

Run the following to deploy:

```shell
kubectl apply -f k8s-hpa.yaml -f k8s-proper-deployments.yaml
```

You should see the following, after about 2 minutes:

- 1 pod for the deployment with no cpu
- 1 pod for the deployment with full cpu

```text
> kubectl top po

NAME                              CPU(cores)   MEMORY(bytes)
myapp-full-cpu-75b594fd4b-bld6l   500m         0Mi
myapp-no-cpu-57b4d84f94-2c7rs     0m           0Mi

> kubectl get -f k8s-hpa.yaml

NAME    REFERENCE                 TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
myapp   Deployment/myapp-no-cpu   0%/50%    1         3         1          5m50s

> kubectl get po 

NAME                              READY   STATUS    RESTARTS   AGE
myapp-full-cpu-75b594fd4b-bld6l   1/1     Running   0          6m16s
myapp-no-cpu-57b4d84f94-2c7rs     1/1     Running   0          6m16s

> kubectl describe -f k8s-hpa.yaml             

Warning: autoscaling/v2beta2 HorizontalPodAutoscaler is deprecated in v1.23+, unavailable in v1.26+; use autoscaling/v2 HorizontalPodAutoscaler
Name:                                                  myapp
Namespace:                                             default
Labels:                                                <none>
Annotations:                                           <none>
CreationTimestamp:                                     Thu, 25 Aug 2022 12:38:57 +0100
Reference:                                             Deployment/myapp-no-cpu
Metrics:                                               ( current / target )
  resource cpu on pods  (as a percentage of request):  0% (0) / 50%
Min replicas:                                          1
Max replicas:                                          3
Deployment pods:                                       1 current / 1 desired
Conditions:
  Type            Status  Reason            Message
  ----            ------  ------            -------
  AbleToScale     True    ReadyForNewScale  recommended size matches current size
  ScalingActive   True    ValidMetricFound  the HPA was able to successfully calculate a replica count from cpu resource utilization (percentage of request)
  ScalingLimited  True    TooFewReplicas    the desired replica count is less than the minimum replica count
Events:
  Type     Reason                        Age    From                       Message
  ----     ------                        ----   ----                       -------
  Warning  FailedGetResourceMetric       8m46s  horizontal-pod-autoscaler  failed to get cpu utilization: unable to get metrics for resource cpu: no metrics returned from resource metrics API
  Warning  FailedComputeMetricsReplicas  8m46s  horizontal-pod-autoscaler  invalid metrics (1 invalid out of 1), first error is: failed to get cpu utilization: unable to get metrics for resource cpu: no metrics returned from resource metrics API
  Warning  FailedGetResourceMetric       8m31s  horizontal-pod-autoscaler  failed to get cpu utilization: did not receive metrics for any ready pods
  Warning  FailedComputeMetricsReplicas  8m31s  horizontal-pod-autoscaler  invalid metrics (1 invalid out of 1), first error is: failed to get cpu utilization: did not receive metrics for any ready pods
```

To clean up:

```shell
kubectl delete -f k8s-hpa.yaml -f k8s-proper-deployments.yaml
```

Wait 30 seconds before re-deploying to ensure everything is cleaned.

# Testing a broken deployment

Here, the deployments have a label selector that matches labels of other pods. This causes the HPA to incorrectly calculate the CPU usage metrics. The result is, the deployment with 0 cpu usage is scaled to 3 replicas.

Run the following to deploy:

```shell
kubectl apply -f k8s-hpa.yaml -f k8s-broken-deployments.yaml
```

You should see the following, after about 2 minutes:

- 3 pods for the deployment with no cpu
- 1 pod for the deployment with full cpu

```text
> kubectl top po --use-protocol-buffers

NAME                              CPU(cores)   MEMORY(bytes)
myapp-full-cpu-856d96cb65-pdsxb   501m         0Mi
myapp-no-cpu-99b97b4f5-5hlb4	  0m           0Mi
myapp-no-cpu-99b97b4f5-7ssjt	  0m           0Mi
myapp-no-cpu-99b97b4f5-bprxp	  0m           0Mi

> kubectl get -f k8s-hpa.yaml

NAME    REFERENCE                 TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
myapp   Deployment/myapp-no-cpu   25%/40%   1         3         3          4m19s

> kubectl get po

NAME                              READY   STATUS    RESTARTS   AGE
myapp-full-cpu-856d96cb65-pdsxb   1/1     Running   0          3m55s
myapp-no-cpu-99b97b4f5-5hlb4	  1/1     Running   0          2m55s
myapp-no-cpu-99b97b4f5-7ssjt	  1/1     Running   0          3m55s
myapp-no-cpu-99b97b4f5-bprxp	  1/1     Running   0          2m55s

> kubectl describe -f k8s-hpa.yaml

Warning: autoscaling/v2beta2 HorizontalPodAutoscaler is deprecated in v1.23+, unavailable in v1.26+; use autoscaling/v2 HorizontalPodAutoscaler
Name:                                                  myapp
Namespace:                                             default
Labels:                                                <none>
Annotations:                                           <none>
CreationTimestamp:                                     Thu, 25 Aug 2022 12:55:23 +0100
Reference:                                             Deployment/myapp-no-cpu
Metrics:                                               ( current / target )
  resource cpu on pods  (as a percentage of request):  25% (125m) / 40%
Min replicas:                                          1
Max replicas:                                          3
Deployment pods:                                       3 current / 3 desired
Conditions:
  Type            Status  Reason              Message
  ----            ------  ------              -------
  AbleToScale     True    ReadyForNewScale    recommended size matches current size
  ScalingActive   True    ValidMetricFound    the HPA was able to successfully calculate a replica count from cpu resource utilization (percentage of request)
  ScalingLimited  False   DesiredWithinRange  the desired count is within the acceptable range
Events:
  Type     Reason                        Age    From                       Message
  ----     ------                        ----   ----                       -------
  Warning  FailedGetResourceMetric       4m22s  horizontal-pod-autoscaler  failed to get cpu utilization: unable to get metrics for resource cpu: no metrics returned from resource metrics API
  Warning  FailedComputeMetricsReplicas  4m22s  horizontal-pod-autoscaler  invalid metrics (1 invalid out of 1), first error is: failed to get cpu utilization: unable to get metrics for resource cpu: no metrics returned from resource metrics API
  Normal   SuccessfulRescale             3m37s  horizontal-pod-autoscaler  New size: 3; reason: cpu resource utilization (percentage of request) above target
```

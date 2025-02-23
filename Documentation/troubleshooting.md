---
weight: 209
toc: true
title: Troubleshooting
menu:
    docs:
        parent: operator
lead: ""
images: []
draft: false
description: Guide on troubleshooting the Prometheus Operator.
---

### `CustomResourceDefinition "..." is invalid: metadata.annotations: Too long` issue

When applying updated CRDs on a cluster, you may face the following error message:

```bash
$ kubectl apply -f $MANIFESTS
The CustomResourceDefinition "prometheuses.monitoring.coreos.com" is invalid: metadata.annotations: Too long: must have at most 262144 bytes
```

The reason is that apply runs in the client by default and saves information into the object annotations but there's a hard limit on the size of annotations.

The workaround is to use server-side apply which requires Kubernetes v1.22 at least.

```bash
kubectl apply --server-side --force-conflicts -f $MANIFESTS
```

If using ArgoCD, please refer to their [documentation](https://argo-cd.readthedocs.io/en/latest/user-guide/sync-options/#server-side-apply).

### RBAC on Google Container Engine (GKE)

When you try to create `ClusterRole` (`kube-state-metrics`, `prometheus` `prometheus-operator`, etc.) on GKE Kubernetes cluster running 1.6 version, you will probably run into permission errors:

```
<....>
Error from server (Forbidden): error when creating
"manifests/prometheus-operator/prometheus-operator-cluster-role.yaml":
clusterroles.rbac.authorization.k8s.io "prometheus-operator" is forbidden: attempt to grant extra privileges:
<....>
```

This is due to the way Container Engine checks permissions. From [Google Kubernetes Engine docs](https://cloud.google.com/kubernetes-engine/docs/how-to/role-based-access-control):

> Because of the way Container Engine checks permissions when you create a Role or ClusterRole, you must first create a RoleBinding that grants you all of the permissions included in the role you want to create.
> An example workaround is to create a RoleBinding that gives your Google identity a cluster-admin role before attempting to create additional Role or ClusterRole permissions.
> This is a known issue in the Beta release of Role-Based Access Control in Kubernetes and Container Engine version 1.6.

To overcome this, you must grant your current Google identity `cluster-admin` Role:

```console
# get current google identity
$ gcloud info | grep Account
Account: [myname@example.org]

# grant cluster-admin to your current identity
$ kubectl create clusterrolebinding myname-cluster-admin-binding --clusterrole=cluster-admin --user=myname@example.org
Clusterrolebinding "myname-cluster-admin-binding" created
```

### Troubleshooting ServiceMonitor changes

When creating/deleting/modifying `ServiceMonitor` objects it is sometimes not as obvious what piece is not working properly. This section gives a step by step guide how to troubleshoot such actions on a `ServiceMonitor` object.

#### Overview of `ServiceMonitor` tagging and related elements

A common problem related to `ServiceMonitor` identification by Prometheus is related to the object's labels not matching the `Prometheus` custom resource definition scope, or lack of permission for the Prometheus `ServiceAccount` to *get, list, watch* `Services` and `Endpoints` from the target application being monitored. As a general guideline consider the diagram below, giving an example of a `Deployment` and `Service` called `my-app`, being monitored by Prometheus based on a `ServiceMonitor` named `my-service-monitor`:

<!-- do not change this link without verifying that the image will display correctly on https://prometheus-operator.dev -->

![flow diagram](/img/custom-metrics-elements.png)

Note: The `ServiceMonitor` references a `Service` (not a `Deployment`, or a `Pod`), by labels *and* by the port name in the `Service`. This *port name* is optional in Kubernetes, but must be specified for the `ServiceMonitor` to work. It is not the same as the port name on the `Pod` or container, although it can be.

### Debugging Why Resources Are Not Being Picked Up
The Prometheus Operator will reject invalid resources and not reconcile them in the Prometheus configuration, for instance if the `ServiceMonitor object has a references to a secret which doesn't exist in the cluster. When it happens the Operator emits a Kubernetes event detailing the issue.

To check for events related to a `ServiceMonitor`  object, you can use the following command:
...

{{< alert icon="👉" text="Prometheus Operator can emit Kubernetes events starting with v0.71.0."/>}}

To check for events related to rejected resources, you can use the following command:

```sh
kubectl get events --field-selector=involvedObject.name="<name of PodMonitor resource>" -n "<namespace where resource is deployed>"
```

Use the following metrics to identify rejected resources:

```
prometheus_operator_managed_resources
```

#### Has my `ServiceMonitor` been picked up by Prometheus?

`ServiceMonitor` objects and the namespace where they belong are selected by the `serviceMonitorSelector` and `serviceMonitorNamespaceSelector`of a Prometheus object. The name of a `ServiceMonitor` is encoded in the Prometheus configuration, so you can simply grep whether it is present there. The configuration generated by the Prometheus Operator is stored in a Kubernetes `Secret`, named after the Prometheus object name prefixed with `prometheus-` and is located in the same namespace as the Prometheus object. For example for a Prometheus object called `k8s` one can find out if the `ServiceMonitor` named `my-service-monitor` has been picked up with:

```sh
kubectl -n monitoring get secret prometheus-k8s -ojson | jq -r '.data["prometheus.yaml.gz"]' | base64 -d | gunzip | grep "my-service-monitor"
```

#### It is in the configuration but not on the Service Discovery page

ServiceMonitors pointing to Services that do not exist (e.g. nothing matching `.spec.selector`) will lead to this ServiceMonitor not being added to the Service Discovery page. Check if you can find any Service with the selector you configured.

If you use `.spec.selector.matchLabels` (instead of e.g. `.spec.selector.matchExpressions`), you can use this command to check for services matching the given label:

```
kubectl get services -l "$(kubectl get servicemonitors -n "<namespace of your ServiceMonitor>" "<name of your ServiceMonitor>" -o template='{{ $first := 1 }}{{ range $key, $value := .spec.selector.matchLabels }}{{ if eq $first 0 }},{{end}}{{ $key }}={{ $value }}{{ $first = 0 }}{{end}}')"
```

Note: this command does not take namespaces into account. If your ServiceMonitor selects a single namespace or all namespaces, you can just add that to the `kubectl get services` command (using `-n $namespace` or `-A` for all namespaces).

### Prometheus kubelet metrics server returned HTTP status 403 Forbidden

Prometheus is installed, all looks good, however the `Targets` are all showing as down. All permissions seem to be good, yet no joy. Prometheus pulling metrics from all namespaces expect kube-system, and Prometheus has access to all namespaces including kube-system.

#### Did you check the webhooks?

Issue has been resolved by amending the webhooks to use `0.0.0.0` instead of `127.0.0.1`. Follow the below commands and it will update the webhooks which allows connections to all `clusterIP's` in all `namespaces` and not just `127.0.0.1`.

**Update the kubelet service to include webhook and restart:**

```sh
KUBEADM_SYSTEMD_CONF=/etc/systemd/system/kubelet.service.d/10-kubeadm.conf
sed -e "/cadvisor-port=0/d" -i "$KUBEADM_SYSTEMD_CONF"
if ! grep -q "authentication-token-webhook=true" "$KUBEADM_SYSTEMD_CONF"; then
  sed -e "s/--authorization-mode=Webhook/--authentication-token-webhook=true --authorization-mode=Webhook/" -i "$KUBEADM_SYSTEMD_CONF"
fi
systemctl daemon-reload
systemctl restart kubelet
```

**Modify the kube controller and kube scheduler to allow for reading data:**

```sh
sed -e "s/- --address=127.0.0.1/- --address=0.0.0.0/" -i /etc/kubernetes/manifests/kube-controller-manager.yaml
sed -e "s/- --address=127.0.0.1/- --address=0.0.0.0/" -i /etc/kubernetes/manifests/kube-scheduler.yaml
```

### Using textual port number instead of port name

The ServiceMonitor expects to use the port name as defined on the Service. So, using the Service example from the
diagram above, we have this Service definition:

```yaml mdox-exec="cat example/user-guides/getting-started/example-app-service.yaml"
kind: Service
apiVersion: v1
metadata:
  name: example-app
  labels:
    app: example-app
spec:
  selector:
    app: example-app
  ports:
  - name: web
    port: 8080
```

We would then define the service monitor using `metrics` as the port not `"8080"`. E.g.

**CORRECT**

```yaml mdox-exec="cat example/user-guides/getting-started/example-app-service-monitor.yaml"
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: example-app
  labels:
    team: frontend
spec:
  selector:
    matchLabels:
      app: example-app
  endpoints:
  - port: web
```

**INCORRECT**

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: example-app
  labels:
    team: frontend
spec:
  selector:
    matchLabels:
      app: example-app
  endpoints:
  - port: "8080"
```

The incorrect example will give an error along these lines `spec.endpoints.port in body must be of type string: "integer"`

### Prometheus/Alertmanager pods stuck in terminating loop with healthy start up logs

It is usually a sign that more than one operator wants to manage the resource.

Check if several operators are running on the cluster:

```console
kubectl get pods --all-namespaces | grep 'prom.*operator'
```

Check the logs of the matching pods to see if they manage the same resource.

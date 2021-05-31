# Spark On Ozone Kerberos

## Minikube
When testing in minikube, you usually install minikube on docker.

If minikube is installed on the Kubernetes docker bridge network, pod cannot use its own service name.

Therefore, in order for the Kerberos server,  minikube hairpin mode must be modified.

refer this [link](https://kubernetes.io/docs/tasks/debug-application-cluster/debug-service/#a-pod-fails-to-reach-itself-via-the-service-ip)


```
sudo minikube start \
  --driver=none \
  --extra-config=kubelet.hairpin-mode="promiscuous-bridge"

# enable docker promiscuous mode also 
sudo ip link set docker0 promisc on
```

## Install
```
# namespace for krb-oprator
kubectl create ns kerberos

# namespace for krb-server and ozone
kubectl create ns ozone

helm install -n kerberos krb-operator ./kerberos/operator

helm install -n ozone krb-server ./kerberos/server

helm install -n ozone ozone ./ozone

```
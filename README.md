# Istio Sample

This project contains all about proof of concept for Istio.

## Getting started

### Prerequisites

* [Docker](https://docs.docker.com/install/)
* [Minikube](https://kubernetes.io/docs/getting-started-guides/minikube/)

### Installing

After install Minikube, run:

```bash
minikube start \
	--extra-config=controller-manager.ClusterSigningCertFile="/var/lib/localkube/certs/ca.crt" \
	--extra-config=controller-manager.ClusterSigningKeyFile="/var/lib/localkube/certs/ca.key" \
	--extra-config=apiserver.Admission.PluginNames=NamespaceLifecycle,LimitRanger,ServiceAccount,PersistentVolumeLabel,DefaultStorageClass,DefaultTolerationSeconds,MutatingAdmissionWebhook,ValidatingAdmissionWebhook,ResourceQuota \
	--kubernetes-version=v1.9.0
```

Go to the [Istio release](https://github.com/istio/istio/releases) page to download the installation file corresponding to your OS. If you are using a MacOS or Linux system, you can also run the following command to download and extract the latest release automatically:

```bash
curl -L https://git.io/getLatestIstio | sh -
```

Change directory to istio package. For example, if the package is istio-0.6

```bash
cd istio-0.6
```

Add the `istioctl` client to your PATH. For example, run the following command on a MacOS or Linux system:

```bash
export PATH=$PWD/bin:$PATH
```

Install Istio’s core components. Choose one of the two mutually exclusive options below or alternately install with the [Helm Chart](https://istio.io/docs/setup/kubernetes/helm.html), first install istio without enabling Mutual SSL authentication **OR** second install istio and enable Mutual SSL authentication:

```bash
kubectl apply -f install/kubernetes/istio.yaml
```

```bash
kubectl apply -f install/kubernetes/istio-auth.yaml
```

## Running the tests

Verifying the installation:

Ensure the following Kubernetes services are deployed: `istio-pilot`, `istio-mixer`, `istio-ingress`:

```bash
kubectl get svc -n istio-system

NAME            CLUSTER-IP      EXTERNAL-IP       PORT(S)                       AGE
istio-ingress   10.83.245.171   35.184.245.62     80:32730/TCP,443:30574/TCP    5h
istio-pilot     10.83.251.173   <none>            8080/TCP,8081/TCP             5h
istio-mixer     10.83.244.253   <none>            9091/TCP,9094/TCP,42422/TCP   5h

```

Ensure the corresponding Kubernetes pods are deployed and all containers are up and running: `istio-pilot-*`, `istio-mixer-*`, `istio-ingress-*`, `istio-ca-*`, and, optionally, `istio-sidecar-injector-*`:

```bash
kubectl get pods -n istio-system

NAME                                     READY     STATUS    RESTARTS   AGE
istio-ca-3657790228-j21b9                1/1       Running   0          5h
istio-ingress-1842462111-j3vcs           1/1       Running   0          5h
istio-sidecar-injector-184129454-zdgf5   1/1       Running   0          5h
istio-pilot-2275554717-93c43             1/1       Running   0          5h
istio-mixer-2104784889-20rm8             2/2       Running   0          5h
```

## Deployment

Now we can deploy own application or one of the sample applications provided with the installation like [Bookinfo](https://istio.io/docs/guides/bookinfo.html).

If you started the [Istio-sidecar-injector](https://istio.io/docs/setup/kubernetes/sidecar-injection.html#automatic-sidecar-injection), as shown above, you can deploy the application directly using `kubectl create`.

The Istio-Sidecar-injector will automatically inject Envoy containers into your application pods assuming running in namespaces labeled with `istio-injection=enabled`:

```bash
kubectl label namespace <namespace> istio-injection=enabled
kubectl create -n <namespace> -f <your-app-spec>.yaml
```

If you do not have the Istio-sidecar-injector installed, you must use [istioctl kube-inject](https://istio.io/docs/reference/commands/istioctl.html#istioctl%20kube-inject) to manuallly inject Envoy containers in your application pods before deploying them:

```bash
kubectl create -f <(istioctl kube-inject -f <your-app-spec>.yaml)
```

In this case, in order to deploy bookinfo sample from istio:

```bash
kubectl apply -f <(istioctl kube-inject --debug -f samples/bookinfo/kube/bookinfo.yaml)
```

Confirm all services and pods are correctly defined and running:

```bash
kubectl get services

NAME          TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
details       ClusterIP   10.106.131.107   <none>        9080/TCP   36s
kubernetes    ClusterIP   10.96.0.1        <none>        443/TCP    62d
productpage   ClusterIP   10.111.233.180   <none>        9080/TCP   36s
ratings       ClusterIP   10.111.133.84    <none>        9080/TCP   36s
reviews       ClusterIP   10.96.219.126    <none>        9080/TCP   36s
```

And:

```bash
kubectl get pods

NAME                              READY     STATUS            RESTARTS   AGE
details-v1-7986ddbd99-268k5       0/2       PodInitializing   0          1m
productpage-v1-567857db67-4klkr   0/2       PodInitializing   0          1m
ratings-v1-659fccc755-77r4d       0/2       PodInitializing   0          1m
reviews-v1-74b4ff9c-hl5rc         0/2       PodInitializing   0          1m
reviews-v2-5d687c686c-kb2vl       0/2       PodInitializing   0          1m
reviews-v3-6bbf469f69-pdsv2       0/2       PodInitializing   0          1m
```

Finally, we have deployed something like this:

![Schema](https://istio.io/docs/guides/img/bookinfo/withistio.svg)

View on Minikube dashboard:

```bash
minikube dashboard
```

This command open dashboard on a browser.

## What's next

To confirm that the Bookinfo application is running, run the following `curl` command:

```bash
curl -o /dev/null -s -w "%{http_code}\n" http://${GATEWAY_URL}/productpage

200
```

You can also point your browser to `http://$GATEWAY_URL/productpage` to view the Bookinfo web page. If you refresh the page several times, you should see different versions of reviews shown in productpage, presented in a round robin style (red stars, black stars, no stars), since we haven’t yet used Istio to control the version routing.

You can now use this sample to experiment with Istio’s features for traffic routing, fault injection, rate limitting, etc.. To proceed, refer to one or more of the [Istio Guides](https://istio.io/docs/guides), depending on your interest. [Intelligent Routing](https://istio.io/docs/guides/intelligent-routing.html) is a good place to start for beginners.

### Determining the ingress IP and Port

```bash 
export GATEWAY_URL=$(kubectl get po -l istio=ingress -n istio-system -o 'jsonpath={.items[0].status.hostIP}'):$(kubectl get svc istio-ingress -n istio-system -o 'jsonpath={.spec.ports[0].nodePort}')
```

```bash
echo $GATEWAY_URL

http://192.168.99.100:30098/
```

Open in your browser `http://192.168.99.100:30098/productpage`.


## Authors

* **Sergio Rodríguez Calvo** - *Development* - [serrodcal](https://github.com/serrodcal)






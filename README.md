# Tanzu Service Routing Workshop

## Overview

There are several ways to route traffic to applications. Here's a few examples.

1. Kubernetes Ingress
2. API Gateways
3. Service Meshes

But how do they work? What would the decision tree be for these technologies? Which is the best to use when?

In this workshop the participants will have access to a kubernetes cluster to test out some or all of these technologies.

Participants will be provided login information and access to a Linux host to use kubectl.

## Workshop

### Validate Access

Please login to the terminal/jumpbox server with your user and password (which was provided to you by the workshop team).

* Check pods

```
kubectl get pods
```

Expected output: 

```
$ kubectl get pods
No resources found in $USER namespace.
```

* List namespaces. NOTE: This will NOT work, and shouldn't work.

```
kubectl get ns
```

Expected output:

```
$ kubectl get ns
Error from server (Forbidden): namespaces is forbidden: User "$USER" cannot list resource "namespaces" in API group "" at the cluster scope
```

### OPTIONAL: Copy kubeconfig

The workshop assumes commands will be run from the terminal/jumpbox server, but participants could also copy the kubeconfig from the terminal/jumpbox server to their local workstation if working from there is easier or more familiar.

For most participants the optimal workflow will be to simply use the terminal/jumpbox and copy and paste commands from this README.

### Clone this Repository

* Clone this repository

```
git clone https://github.com/ccollicutt/tanzu-service-routing-workshop
```

### Contour

In this section of the workshop we'll look at Contour, an open source ingress controller, which actually does a lot more than simple ingress.

>Contour is an Ingress controller for Kubernetes that works by deploying the Envoy proxy as a reverse proxy and load balancer. Contour supports dynamic configuration updates out of the box while maintaining a lightweight profile.
>Contour also introduces a new ingress API (HTTPProxy) which is implemented via a Custom Resource Definition (CRD). Its goal is to expand upon the functionality of the Ingress API to allow for a richer user experience as well as solve shortcomings in the original design. - [Contour Website](https://github.com/projectcontour/contour)

#### Use contour-examples

* Ensure you are in the correct directory

```
cd tanzu-service-routing-workshop/contour-examples
```

#### Create Nginx Deployment

* Create an nginx deployment

```
kubectl create -f nginx-deployment.yaml 
```

#### Basic Ingress

* Review the manifest and examine what is configured in it

```
cat basic-ingress.yaml
```

* Deploy the manifest

>NOTE: We must replace the text `USERX` with your Kubernetes user. Either use something like `sed` or edit the file manually and replace all occurrences of USERX with your user name, eg. $USER.

```
sed "s/USERX/$USER/g" basic-ingress.yaml | kubectl create -f - 
```

* Examine the Kubernetes objects

```
kubectl get all,ingress
```

* Test DNS

```
until host nginx.$USER.sr.globalbanque.com; do
  echo "sleeping..."
  sleep 2
done
```

Expected output:

```
$ host nginx.$USER.sr.globalbanque.com
nginx.user1.sr.globalbanque.com has address 18.218.176.52
nginx.user1.sr.globalbanque.com has address 18.221.2.68
```

* Test the ingress

```
until curl -s http://nginx.$USER.sr.globalbanque.com; do
   echo "sleeping..."
   sleep 2
done
```

Expected output: 

```
$ curl -s http://nginx.$USER.sr.globalbanque.com  | grep title
<title>Welcome to nginx!</title>
```

#### HTTPProxy

>The goal of the HTTPProxy Custom Resource Definition (CRD) is to expand upon the functionality of the Ingress API to allow for a richer user experience as well addressing the limitations of the latter’s use in multi tenant environments. - [Contour Documentation](https://projectcontour.io/docs/v1.16.0/config/fundamentals/)

* Deploy a basic HTTPProxy

```
sed "s/USERX/$USER/g" basic-httpproxy.yaml | kubectl create -f -
```

Expected output:

```
$ sed 's/USERX/user1/g' basic-httpproxy.yaml | kubectl create -f -
httpproxy.projectcontour.io/nginx-httpproxy created
```

* Check for httpproxy object

```
kubectl get httpproxies.projectcontour.io 
```

Expected output:

```
$ kubectl get httpproxies.projectcontour.io 
NAME              FQDN                                        TLS SECRET   STATUS   STATUS DESCRIPTION
nginx-httpproxy   nginx-httpproxy.user1.sr.globalbanque.com                valid    Valid HTTPProxy
```

* Check DNS (can take a couple minutes to become live)

```
until host nginx-httpproxy.$USER.sr.globalbanque.com; do
  echo "sleeping..."
  sleep 2
done
```

* Curl the httpproxy URL

```
curl -s nginx-httpproxy.$USER.sr.globalbanque.com | grep title
```

Expected output:

```
$ curl -s nginx-httpproxy.$USER.sr.globalbanque.com | grep title
<title>Welcome to nginx!</title>
```

#### Inclusion and Delegation



>HTTPProxy permits the splitting of a system’s configuration into separate HTTPProxy instances using inclusion.
> Inclusion, as the name implies, allows for one HTTPProxy object to be included in another, optionally with some conditions inherited from the parent. Contour reads the inclusion tree and merges the included routes into one big object internally before rendering Envoy config. Importantly, the included HTTPProxy objects do not have to be in the same namespace. - [Contour Documentation](https://projectcontour.io/docs/main/config/inclusion-delegation/)

This solves the problem of having multiple teams working in the same Kubernetes cluster accidentally stomping on the same (ingress) routes. With inclusion, a higher level admin can create the base "root" domains and delegate underlying routes to other teams.

>Note that the Contour deployment can have the `--root-namespace` configured. When this is configured Contour will  only look for root HTTPProxies in the namspaces set, which would be managed by an admin.

```
$ grep -A 4 -B 4 root-namespace contour.yaml 
        - --contour-cafile=/certs/ca.crt
        - --contour-cert-file=/certs/tls.crt
        - --contour-key-file=/certs/tls.key
        - --config-path=/config/contour.yaml
        - --root-namespaces=root-inclusion
        command: ["contour"]
        image: docker.io/projectcontour/contour:v1.16.0
        imagePullPolicy: IfNotPresent
        name: contour
```

* Example of a root inclusion

>NOTE: When you use an include + prefix, the prefix is not automatically removed, but you can remove it in the included HTTProxy object.

This assumes that an admin user has created the root HTTPProxy, such as the below. Note the `include` section.

```
---
apiVersion: projectcontour.io/v1
kind: HTTPProxy
metadata:
  name: root-inclusion
  namespace: root-inclusion
spec:
  virtualhost:
    fqdn: root.sr.globalbanque.com
  includes:
  - name: user1
    namespace: user1
    conditions:
    - prefix: /user1
...
```

* Create the  inclusion

```
sed "s/USERX/$USER/g" httpproxy-user-inclusion.yaml | k create -f -
```

* Curl it

```
until curl -s root.sr.globalbanque.com/$USER ; do
   echo "sleeping..."
   sleep 2
done
```

Example output:

```
$ curl -s root.sr.globalbanque.com/$USER | grep title
<title>Welcome to nginx!</title>
```

#### Local Rate Limiting

The HTTPProxy API supports defining local rate limit policies that can be applied to either individual routes or entire virtual hosts. Local rate limit policies define a maximum number of requests per unit of time that an Envoy should proxy to the upstream service. Requests beyond the defined limit will receive a 429 (Too Many Requests) response by default. Local rate limit policies program Envoy’s HTTP local rate limit filter. - [Contour Documentation](https://projectcontour.io/docs/v1.16.0/config/rate-limiting/#local-rate-limiting)

* Create the ratelimited HTTPProxy.

```
sed "s/USERX/$USER/g" httpproxy-ratelimiting.yaml | kubectl create -f -
```

* Check DNS

>NOTE: It can take a minute or two for the DNS entry to become available.

```
host rl.$USER.sr.globalbanque.com
```

 * Curl it to find out when it's active

```
until curl -s rl.$USER.sr.globalbanque.com; do
  echo "sleeping..."
  sleep 2
done
```

* Run this curl command several times, eventually you will be rate limited

 ```
 $ curl -s rl.user1.sr.globalbanque.com
local_rate_limited
```

#### TLS

* First, create a certificate

```
sed "s/USERX/$USER/g" certificate.yaml | kubectl create -f - 
```

* Check if the certificate is available

```
kubectl get certificates
```

>NOTE: This can take a few minutes.

Expected output:

```
$ kubectl get certificates
NAME                                 READY   SECRET                               AGE
tls-user1-sr-globalbanque-com-cert   True    tls-user1-sr-globalbanque-com-cert   2m8s
```

* Now we can use that certificate in a HTTPProxy

Note the secret name:

```
    tls:
      secretName: YOUR_TLS_SECRET
```

* Create the HTTPProxy

```
sed "s/USERX/$USER/g" httpproxy-with-tls.yaml | kubectl create -f -
```

* Curl the TLS URL

```
until host tls.$USER.sr.globalbanque.com; do
  echo "sleeping..."
  sleep 2
done
until curl -s https://tls.$USER.sr.globalbanque.com; do
  echo "sleeping..."
  sleep 2
done
```

* Check the cert

```
echo |openssl s_client -showcerts -connect tls.$USER.sr.globalbanque.com:443
```

#### Upstream Weighting

* Create a second service and deployment

```
kubectl create -f httpd-deployment.yaml 
```

* Setup a weighted route

```
sed "s/USERX/$USER/g" httpproxy-weighted-routing.yaml | kubectl create -f -
```

* Check the DNS entry...

```
until host wr.$USER.sr.globalbanque.com; do
  echo "sleeping..."
  sleep 2
done
```

Expected output:

```

```

* Test the ingress

```
until curl -s http://wr.$USER.sr.globalbanque.com; do
   echo "sleeping..."
   sleep 2
done
```

* Keep curling it...

```
curl -s http://wr.$USER.sr.globalbanque.com
```

Approximately 9 out of 10 times the result should be the below, indicating that the httpd service is being used instead of the nginx service.

```
$ curl -s http://wr.$USER.sr.globalbanque.com
<html><body><h1>It works!</h1></body></html>
```

#### Conclusion

After all of this we should have these ingress and httproxies.

```
kubectl get ing,httpproxies
```

```
$ kubectl get ing,httpproxies.projectcontour.io 
NAME                                      CLASS    HOSTS                             ADDRESS                                                                  PORTS     AGE
ingress.networking.k8s.io/nginx-ingress   <none>   nginx.user1.sr.globalbanque.com   af7c4699894b04423906fd0800239329-818259978.us-east-2.elb.amazonaws.com   80, 443   20m

NAME                                          FQDN                                        TLS SECRET   STATUS   STATUS DESCRIPTION
httpproxy.projectcontour.io/nginx-httpproxy   nginx-httpproxy.user1.sr.globalbanque.com                valid    Valid HTTPProxy
httpproxy.projectcontour.io/ratelmiting       rl.user1.sr.globalbanque.com                             valid    Valid HTTPProxy
httpproxy.projectcontour.io/user1                                                                      valid    Valid HTTPProxy
```

#### Cleanup

To remove all the objects from the users namespace:

```
cd ~
kubectl config use-context $USER 
kubectl delete ingress --all
kubectl delete httpproxy --all
kubectl delete svc --all
kubectl delete deploy --all
for s in `kubectl get secrets | grep -v default-token | grep -v NAME | cut -f 1 -d " "`; do
  kubectl delete secret $s
done
rm -f tanzu-service-routing-workshop/
```

Should be no resources left.

```
kubectl get all
```
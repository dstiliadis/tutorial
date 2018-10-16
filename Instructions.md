# Setup Instructions

## Basic setup 
1. In order to follow this tutorial you will need an existing working 
   Kubernetes cluster with version 1.10 or higher. We will assume that 
   you have git and kubectl installed as well. 

2. Download Tutorial
```
git clone https://github.com/dstiliadis/tutorial.git
cd tutorial 
```

3. Installing Istio in the Kubernetes Cluster
This is done in two steps. First create the Custom Resource Definitions
in Kubernetes that will allow us to use the standard kubectl command for 
everything. 
```
kubectl apply -f install/crds.yaml
```

Now we can create all the necessary objects of an Istio deployment. We use 
the simple mechanism of instantiating all components erything here. More advanced options
and customization are available through Helm charts. 

```
kubectl apply -f install/istio-demo-auth.yaml
```

5. First we can start the bookinfo application without injecting the Istio
configuration. You will notice that PODs are created without sidecars.

```
kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml
```

View the sidecar deployment. 
```
kubectl get pods -o wide
```

Before proceeding we will delete the application in order to inject the sidecars.
```
kubectl delete -f samples/bookinfo/platform/kube/bookinfo.yaml
```


6. Automatic sidecar injection can be configured on per namespace basis.
The following command will configure sidecar injection for the default
namespace.

```
kubectl label namespace default istio-injection=enabled
```

7. Apply again the application and see that we now have the sidecars
```
kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml
```

Retrieve the pods and services. Wait until they are all running. 
```
kubectl get pods 
kubectl get services
```

9. In order to be able to access the service from the Internet we need to 
create the configuration for the ingress gateway.

```
kubectl apply -f samples/bookinfo/networking/bookinfo-gateway.yaml
```
9. We must now find the gateway IP and Port 
```
kubectl -n istio-system get service istio-ingressgateway 
```

You will see the public IP under the EXTERNAL-IP column. The port is 80.

10. We did the cardinal sin. We exposed the service over HTTP. We need to upgrade
the service to TLS. In order to achieve that, we will need certificates for the 
service. You can choose your favorite method to create a private CA and certificates
or you can use the instructions bellow. 

11. Create the certificates with the tg utility. This assumes you have Go installed in your laptop. Alternatively,
a pre-compiled binary for Mac OSX is provided in bin/tg.
```
go get -u github.com/aporeto-inc/tg
```

Create a private CA 
```
mkdir certs ; cd certs 
tg cert --name myca --org "Acme Enterprises" --common-name root --is-ca --pass secret
```

Create a server certificate  with the above DNS name. Otherwise, your authentication will fail. Replace the IP below with your gateway IP.

```
tg cert --name myclient --org "Acme Enterprises" \
   --common-name demo-server \
   --auth-server --auth-client \
   --signing-cert myca-cert.pem  \
   --signing-cert-key myca-key.pem \
   --signing-cert-key-pass secret \
   --ip 104.197.141.45
```

You can use openssl to inspect the certificate and understand what we have created:
```
openssl x509 -in myclient-cert.pem -text
```

In order to configure the secure gateway we will need to create a Kubernetes secrets 
object with the certificate and configure the gateway. The example yaml file 
can be seen in: samples/bookinfo/networking/bookinfo-gateway-secure.yaml 

First, delete the previous gateway 
```
kubectl delete -f samples/bookinfo/networking/bookinfo-gateway.yaml
```

Create kubernetes secrets and apply the security gateway service. 
```
kubectl create -n istio-system secret tls istio-ingressgateway-certs \
  --key certs/myclient-key.pem \
  --cert certs/myclient-cert.pem

kubectl apply -f samples/bookinfo/networking/bookinfo-gateway-secure.yaml 
```

Get the gateway and validate 
```
kubectl describe gateway
```

You can now reach the application at https://<your gateway IP>/productpage 
Note, that your browser will not trust the private CA unless if you explicitly
allow it. 

## Basic Traffic Management 

First we need to create the destination rules. This configures mTLS for all
services and it also defines the subsets that can be used by the routing 
rules to route traffic to different versions of the applicaiton. Subsets
are defined using label selectors. 

```
kubectl apply -f samples/bookinfo/networking/destination-rule-all-mtls.yaml
```

We can now create virtual services that will determine that all reviews 
will be routed to a specific version of the application.
```
kubectl apply -f samples/bookinfo/networking/virtual-service-all-v1.yaml
```

Apply this service .. All reviews will be routed to v1. If you reload the 
product page you will notice that there are no ratings (ie stars) in the page. 

We will now apply different routing rule that will allow only a specific 
user to access the reviews version 2. This is an easy demo, but note that 
it is completely insecure. It works, because the applications are populating
the end-user header in their traffic to other applications. This is not 
verified by anyone.

```
kubectl apply -f samples/bookinfo/networking/virtual-service-reviews-test-v2.yaml
```

Login in the application as user jason (no password) and you should see the 
rating again. 

## Circuit Breaking 

Start a simple server and corresponding service. We will limit the number of 
concurrent connections towards a particular service. 

```
kubectl apply -f samples/httpbin/httpbin.yaml
```

Let's start a benchmark client

```
kubectl run -i --tty wrk --image=dimitrihub/wrk --restart=Never
```

Let’s issue the get command
```
wrk -c 5 -t 5 -d 5s http://httpbin:8000/get
```

We get a baseline of performance. Note that there are no dropped requests.

We apply a very basic circuit breaker that allows on concurrent request. 
Basic throttling.   

```
kubectl apply -f samples/httpbin/destinationpolicies/httpbin-circuit-breaker.yaml
```

We can now repeat the test with 5 connections and see the results.
We can also repeat with just one connection and see the results. 
Lots of connections with errors that were throttled (aproximately 20% in the example above) 

## Telemetry 

1. Start Grafana proxy in the background.
```
kubectl -n istio-system port-forward $(kubectl -n istio-system get pod -l app=grafana -o jsonpath='{.items[0].metadata.name}') 3000:3000 &
```

2. Demonstrate briefly integration with OpenTracing (caveats: the application needs to be instrumented .. much larger topic .. we will not cover here in details)

```
kubectl port-forward -n istio-system $(kubectl get pod -n istio-system -l app=jaeger -o jsonpath='{.items[0].metadata.name}') 17000:16686 &
```

Start the browser on localhost 3000 and 17000 respectively. 


## Fault Injection
Our goal is to use mechanisms in the mesh to debug our applications. We will inject a delay in 
the response to see how the application will behave if the downstream node is not 
responding fast enough. 

We will apply a delay in the requests. 

```
kubectl apply -f samples/bookinfo/networking/virtual-service-ratings-test-delay.yaml
```

Login with user Jason and see what happens.

Go to Jaeger and see he fault and analyze what happened 

## Access to External Services 

Start the sleep container so that we can get access 
```bash
kubectl apply -f samples/sleep/sleep.yaml
```

Apply the service entries that we are creating with all the configuration 
Enter the container and try to access the service. It will fail. 
Simple  curl to google.com.

```
kubectl apply -f samples/external/services.yaml
```

Now from the containers we can access the external services. Note that services 
must be whitelisted in this approach.


## Security: Mutual TLS authentication 

First create a new namespace, where we will not inject the sidecars 
```
kubectl create namespace insecure
```

Let's create a basic server in this namespace:
```
kubectl apply -f samples/httpbin/httpbin.yaml -n insecure
```

Create a client container:
```
kubectl apply -f samples/sleep/sleep.yaml -n insecure
```

Enter the sleep container and try a curl to the original service. This will
fail since we are not presenting a certificate.
```
curl -v httpbin.default:8000/get
```

We are getting a failure, since TLS has not be enabled in the other side. The only way out of this
is to reduce the requirements on the service and accept non TLS traffic. We can create an overwirite 

```
kubectl apply -f samples/httpbin/disable-auth.yaml
```

When we try to access the service again, we will see success.

But authentication doesn't mean authorization. Let's also look at the certificates 

```
kubectl exec wrk  -c istio-proxy -- cat /etc/certs/cert-chain.pem | openssl x509 -text
```

The important item to focus is the URI SAN and how this is used. This is the Spiffee certificate. 
Note, that the URI SAN is essentially based on the namespace and service account. 

## Simple Authorization

In order to implement simple authorization we need three components:

1. Create a service account for the services and *redeploy* the services. 
```
kubectl apply -f  samples/bookinfo/platform/kube/bookinfo-add-serviceaccount.yaml
```

This is needed because main authorization policy is bound to the service accounts. Only one service account per deployment.

2. Enable Autorization for Istio. This can be done selectively all across all the deployments. 
```
kubectl apply -f samples/bookinfo/platform/kube/rbac/rbac-config-ON.yaml
```
At this point access is denied, since we enabled authorization, but have created no rules to all this.

Reload the book info app and see the RBAC message 

3. Let's enable a basic authorization between namespaces. This will just say that all PODs are allowed to talk between the namespaces.
```
kubectl apply -f samples/bookinfo/platform/kube/rbac/namespace-policy.yaml
```

4. Now we can do it based on service accounts. First we delete the previous policy 
```
kubectl delete -f samples/bookinfo/platform/kube/rbac/namespace-policy.yaml
```

5. Open up the ingress .. Allow all users to access the product page:
```
kubectl apply -f samples/bookinfo/platform/kube/rbac/productpage-policy.yaml
```
You see access to the main page .. but no access to the reviews or ratings. 
6. Let's give access to some more services 
```
kubectl apply -f samples/bookinfo/platform/kube/rbac/details-reviews-policy.yaml
```

At this point we can see more results .. but not the ratings. Point is presented. 

## Performance 
Delete service accounts and authorization

```
kubectl delete servicerole --all
kubectl delete servicerolebinding --all
kubectl delete -f samples/bookinfo/platform/kube/rbac/rbac-config-ON.yaml
```

Verify that bookinfo is running completely 

1. Create a no Istio namespace 
```
kubectl create namespace noistio
```

2. Create the app in this namespace
```
kubectl create -f  samples/performance/app-no-istio.yaml
```

Notice that there is no sidecar created. 

3. Run the wrk app 
```
kubectl run -n noistio -i --tty wrk --image=dimitrihub/wrk --restart=Never
```

4. Do the performance test
```
 wrk -c 200 -t 200 -d 10s http://demo/public/test
```

5. Repeat the test in the istio namespace. Create the app with sidecars
```
kubectl create -f samples/performance/app-with-istio.yaml
```

6. Create the worker in the default namespace
```
kubectl run -n default -i --tty wrkclient --image=dimitrihub/wrk --restart=Never
```

7. Run the command 
```
wrk -c 200 -t 200 -d 10s http://demo/public/test
```

8. Discuss the results 



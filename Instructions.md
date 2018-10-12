# Setup Instructions

## Basic setup 
1. Start with a Kubernetes cluster 
2. Download Istio 
```
curl -L https://git.io/getLatestIstio | sh -
cd istio-1.0.2
export PATH=$PWD/bin:$PATH
```

3. Apply CRDs. 
```
kubectl apply -f install/kubernetes/helm/istio/templates/crds.yaml
```
5. Install Istio
```
kubectl apply -f install/kubernetes/istio-demo-auth.yaml
```

5. Start the bookinfo application without the injection.
```
kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml
```
6. Set the side car injector and redo
```
kubectl label namespace default istio-injection=enabled
```
7. Apply again the application and see that we now have the sidecars
```
kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml
```
```
kubectl get pods 
kubectl get services
```
9. Create the configuration for the ingress gateway 
```
kubectl apply -f samples/bookinfo/networking/bookinfo-gateway.yaml
```
9. Get the Gateway IP and port 
```
kubectl -n istio-system get service istio-ingressgateway 
```
10. We did the cardinal sin .. (ie ..we are accessing through http ) .. This needs to be disabled by default .. but, we need certificates:

11. Let's create the certificates, or just bin/tg :
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

Show the certificate and validate the IP 

```
openssl x509 -in myclient-cert.pem -text
```

Show the modifications in the secure service samples/bookinfo/networking/bookinfo-gateway-secure.yaml 

Delete the previous gateway 
```
kubectl delete -f samples/bookinfo/networking/bookinfo-gateway.yaml
```

Remove and apply the new service. Create kubernetes secrets and apply the security gateway service. 
```
kubectl create -n istio-system secret tls istio-ingressgateway-certs \
  --key certs/myclient-key.pem \
  --cert certs/myclient-cert.pem
```

Create the new ingress gateway with TLS enabled:
```
kubectl apply -f samples/bookinfo/networking/bookinfo-gateway-secure.yaml
```

Get the gateway and validate 
```
kubectl describe gateway
```


Go to https now ..At least we removed the cardinal sin .. 

## Basic Traffic Management 
12. Traffic Management 
13. First we need to define the subsets (ie the available versions). This is done here:
```
kubectl apply -f samples/bookinfo/networking/destination-rule-all-mtls.yaml
```
Try the service .. everything should work ..

15. Virtual services .. capture the end points
```
kubectl apply -f samples/bookinfo/networking/virtual-service-all-v1.yaml
```
Apply this service .. All reviews will be routed to v1 ..  no more stars in the picture 

```
kubectl apply -f samples/bookinfo/networking/virtual-service-reviews-test-v2.yaml
```

What this means is essentially that the user-identity is an http header .. make sure to mention that. 

## Circuit Breaking 
Start a simple server and corresponding service 

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

We get a baseline. 

We apply a very basic circuit breaker that allows on concurrent request . Basic throttling.  Edit it and point out
that we are enabling Mutual TLS and what would happen if we don't enable mutual TLS. 

```
kubectl apply -f samples/httpbin/destinationpolicies/httpbin-circuit-breaker.yaml
```

We can now repeat the test with 3 connections and see the results .. We can also repeat with just one connection and 
see the results. 
Lots of connections with errors that were throttled (aproximately 20% in the example above) 

## Telemetry 

1. Start Grafana proxy in the background 
```
kubectl -n istio-system port-forward $(kubectl -n istio-system get pod -l app=grafana -o jsonpath='{.items[0].metadata.name}') 3000:3000 &
```

2. Demonstrate briefly integration with OpenTracing (caveats: the application needs to be instrumented .. much larger topic .. we will not see in details)
```
kubectl port-forward -n istio-system $(kubectl get pod -n istio-system -l app=jaeger -o jsonpath='{.items[0].metadata.name}') 17000:16686 &
```

Start the browser on localhost 3000 and 17000 respectively. 


## Fault Injection
We will inject a delay in the response to see how the application will behave if the downstream node is not responding fast enough. 

Apply a delay:

```
kubectl apply -f samples/bookinfo/networking/virtual-service-ratings-test-delay.yaml
```

Login with user JAson and see what happens.

Go to Jaeger and see he fault and analyze what happened 

## Access to External Services 

Start the sleep container so that we can get access 
```bash
kubectl apply -f samples/sleep/sleep.yaml
```

Apply the service entries that we are creating with all the configuration 
Enter the container and try to access the service. It will fail. 

```
kubectl apply -f samples/external/services.yaml
```

Go to the container and exec the command. 
Now it will succeed. 


## Mutual TLS authentication 
First create a new namespace
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

Enter the sleep container and try a curl to the original service 

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

The important item to discuss is the URI SAN and how this is used. This is the spiffee certificate. 
Note, that it gets the default service acccount .. 

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



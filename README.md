# Getting Started with Ambassador

[Ambassador](https://getambassador.io/) is an open-source API Gateway and Ingress controller based on [Envoy Proxy](https://www.envoyproxy.io/).  



In this repository, we have pre-configured a number of Kubernetes manifests so that you can quickly get a deployment of Ambassador up and running to test.

To get started, clone the repository:
```bash
git clone https://github.com/datawire/amb-config
```

## Installing Ambassador with Helm

1. Add repository.

   ```
   helm repo add datawire https://getambassador.io && helm repo update
   ```

2. Install Ambassador API Gateway.

   Helm v3:
   ```bash
   helm install ambassador datawire/ambassador --set image.repository=datawire/ambassador --set enableAES=false --set namespace.name=default
   ```

   Helm v2: (assumes you already have tiller)
   ```bash
   helm install --name ambassador --namespace default datawire/ambassador --set image.repository=datawire/ambassador --set enableAES=false
   ```

3. Test that Ambassador is working by navigating to the Diagnostics page.  Navigate to `http://{{AMBASSADOR_HOST}}/ambassador/v0/diag/` in your browser.  Note this endpoint can be disabled by setting `diagnostics.enabled: false` in an [Ambassador Module](https://www.getambassador.io/docs/latest/topics/running/ambassador/).

## Getting started with Ingresses

Now, we will be deploying a sample service that outputs a simple Quote of the Moment.  We will then create an Ingress that connects to this service to test Ambassador's capacity as an Ingress controller.

### Deploying and connecting to QotM

1. Deploy the Quote microservice.

   ```bash
   kubectl apply -f ingress/quote.yaml
   ```

2. Deploy a simple Ingress.  Note this Ingress applies both pathtypes, a Prefix: `/quote` and an Exact Match: `/exact-quote/` to show the support Ambassador has for the Kubernetes 1.18 ingress enhancements.

   ```bash
   kubectl apply -f ingress/simple-ingress.yaml
   ```

3. Test with curl.

   ```bash
   curl http://{{ambassador_host}}/quote/
   ```
   and

   ```bash
   curl http://{{ambassador_host}}/backend/get-quote/
   ```

## Getting Started with Basic External Auth

In this tutorial, we will deploy a basic Auth service for user validation to Microservices.

1. Deploy the external `auth-service`. This is a simple auth service that listens on port 3000, uses only username: username, pw: password, and only implements auth on `/backend/get-quote/`.  See implementation at: https://github.com/datawire/ambassador-auth-service.

   ```bash
   kubectl apply -f auth/auth-service.yaml
   ```

2. Configure Ambassador to communicate with the external `auth-service` over HTTP/REST. This is done with the `AuthService` CRD.

   ```bash
   kubectl apply -f auth/amb-auth-service.yaml
   ```

3. Test implementation. Without credentials, we'll get an HTTP 401 error:

   ```bash
   curl -Lv http://{{ambassador_host}}/backend/get-quote/
   ```

   We can successfully authenticate to the service by supplying credentials:
   ```bash
   curl -Lv -u username:password http://{{ambassador_host}}/backend/get-quote/
   ```

## Getting started with cert-manager

Up until this point, we have been running Ambassador in a totally unencrypted http environment.  Naturally this is not a good practice 
in a production environment, so we will want to start generating certificates and terminating TLS.  To manage our certificates, we 
will be using `cert-manager` with an http-01 challenge.  If you can't easily register a domain, [nip.io](https://nip.io/) will let you create an FQDN with just an IP address.

1. Add repository.
   ```bash
   helm repo add jetstack https://charts.jetstack.io && helm repo update
   ```

2. Install `cert-manager`.
   
   Helm v3:
   ```bash
   helm install cert-manager jetstack/cert-manager --namespace default --version v0.15.0 --set installCRDs=true
   ```

   Helm v2:
   ```bash
   helm install --name cert-manager --namespace default --version v0.15.0 jetstack/cert-manager --set installCRDs=true
   ```

3. Create a ClusterIssuer.

   ```bash
   kubectl apply -f cert-manager/cluster-issuer.yaml
   ```

4. Create a Certificate.

   ```bash
   kubectl apply -f cert-manager/certificate.yaml
   ```

5. Tell Ambassador where to send the challenge.

   ```bash
   kubectl apply -f cert-manager/cert-ingress.yaml
   ```

6. Create Host.

   ```bash
   kubectl apply -f cert-manager/host.yaml
   ```
7. Test TLS.
   
   ```
   curl -Lv -u username:password http://{{ambassador_host}}/backend/get-quote/
   ```
   You will notice that it redirects from HTTP to HTTPS automatically.  To configure more advanced routing behaviours, see [Host](https://www.getambassador.io/docs/latest/topics/running/host-crd/#secure-and-insecure-requests) documentation.

## SPDY protocol support.

Demonstrating SPDY support is as simple as using `kubectl exec` into an Ambassador pod.  `kubectl exec` uses SPDY 3.1 to connect to a pod.  An increased verbocity level will show a protocol upgrade to SPDY during the `exec` process.

```bash
kubectl exec -v=8 -it {{ambassador pod}} -- ls 2> >(grep SPDY) >> /dev/null
```

Examining the output shows a protocol upgrade to SPDY/3.1.

## Layer 4 TCP Routing

Ambassador can route TCP traffic directly from specified ports to services in your cluster.  To demonstrate this, we will use a busybox deployment to `netcat` from our local machine to the cluster, creating a direct TCP route.

1. Deploy busybox.
   
   ```bash
   kubectl apply -f layer4/busybox.yaml
   ```

2. Update the Ambassador deployment to open the new port, in this case 30080.
   
   ```bash
   helm upgrade ambassador -f layer4/values.yaml datawire/ambassador
   ```

3. Apply TCPMapping.
   
   ```bash
   kubectl apply -f layer4/tcp-mapping.yaml
   ```

4. Start netcat listener on the busybox pod in a separate terminal.

   ```bash
   kubectl exec -it {{busybox_pod}} -- nc -lvnp 30080
   ```

5. Start netcat client.

   ```bash
   nc -v {{ambassador IP}} 30080
   ```

6. Type away!  Any line typed into stdin can be seen in the other terminal that's listening.

## Layer 7 gRPC Routing

Ambassador natively supports gRPC routing.  In this example, we will deploy a simple gRPC hello service and connect to it.  The example client requires clear-text routing, so an additional Host is also needed.  This example also assumes you have the relevant Python [pip modules](https://grpc.io/docs/quickstart/python/) installed for gRPC.

1. Deploy the gRPC server.
   
   ```bash
   kubectl apply -f layer7/hello-grpc.yaml
   ```

2. Create a new host for cleartext.  I used a different hostname than previous examples (i.e. cleartext.ip.nip.io).

   ```bash
   kubectl apply -f layer7/host.yaml
   ```

3. Start client service, make sure to edit greeter_client.py with your hostname.

   ```bash
   python layer7/greeter_client.py
   ```

   A successful response will return 'Hello, you!'.

## SNI Support

SNI support is also natively supported by Ambassador.  In this example we will deploy a hello service alongside the existing quote service.  Two separate hostnames will be used to route traffic to the respective service using Ingresses.

1. Deploy hello service.

   ```bash
   kubectl apply -f sni/hello.yaml
   ```

2. Create new certificates for your hosts.

   ```bash
   kubectl apply -f sni/certificate-sni.yaml
   ```

3. Once the certs are updated, create new hosts.

   ```bash
   kubectl apply -f sni/host-sni.yaml
   ```

4. Create Ingresses to map the services.

   ```bash
   kubectl apply -f sni/ingress-sni.yaml
   ```

5. Test each hostname to see them resolve to different services.

   ```bash
   curl -Lv https://host1.example.com/
   ```

   And

   ```bash
   curl -Lv https://host2.example.com/
   ```
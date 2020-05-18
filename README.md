# Getting Started with Ambassador

[Ambassador](https://getambassador.io/) is an open-source API Gateway and Ingress controller based on [Envoy Proxy](https://www.envoyproxy.io/).  



In this repository, we have pre-configured a number of Kubernetes manifests so that you can quickly get a deployment of Ambassador up and running to test.

To get started, clone the repository:
```bash
git clone https://github.com/datawire/amb-config
```

## Installing Ambassador

1. Apply Ambassador CRDs.

   ```
   kubectl apply -f install/ambassador-crds.yaml
   ```

2. Install Ambassador.

   ```
   kubectl apply -f install/ambassador-rbac.yaml
   ```

4. Install the Ambassador Service.  Looking at the manifest itself, you can see we are creating a LoadBalancer service that routes incoming traffic from port 80 to 8080, which Ambassador is listening on.

   ```
   kubectl apply -f install/ambassador-service.yaml
   ```

5. Test that Ambassador is working by navigating to the Diagnostics page.  Navigate to `http://{{AMBASSADOR_HOST}}/ambassador/v0/diag/` in your browser.  Note this endpoint can be disabled by setting `diagnostics.enabled: false` in an [Ambassador Module](https://www.getambassador.io/docs/latest/topics/running/ambassador/).

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

### Deploy auth-service

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
# k4k8s demo

## Setup a Sample Service

Deploy 'echoserver' service

```bash
kubectl apply -f ./yaml/echo.yml

kubectl get all -A
```

Test :

Open a new terminal and start port forwarding

```bash
kubectl port-forward svc/echo 11080:80
```

Making a request

```bash
curl -i localhost:11080/echo
```

---

## Proxy service by Kong

***1. Setup an Ingress to use Kong Ingress Controller***

Create an `Ingress` with **IngressClass: kong**

```bash
kubectl apply -f ./yaml/api-ingress.yml
```

Open a new terminal and start port forwarding

```bash
kubectl port-forward -n kong service/kong-proxy 10080:80
```

Making a request

```bash
curl -i localhost:10080/demo/hello \
    -H "host: api.example.com"
```

Result

```json
1. Handle request by Kong Ingress Controller and proxy request by Kong Gateway.
2. Request path "/demo/hello" are passed to service.
```

--

***2. Use KongIngress with Ingress resource***

Create a `KongIngress` and apply to an `Ingress`

```bash
kubectl apply -f ./yaml/api-ingress-config.yml

kubectl patch ing api-ingress -p '{"metadata":{"annotations":{"konghq.com/override":"api-ingress-config"}}}'
```

Making a request

```bash
curl -i localhost:10080/demo/hello \
    -H "host: api.example.com"
```

Result

```json
1. Request path "/hello" are passed to service.
2. Expose custom request basepath "/demo" for service.
```

--

***3. Use KongIngress with Service resource***

Create `KongIngress` and apply to an `Service`

```bash
kubectl apply -f ./yaml/echo-service-config.yml

kubectl patch svc echo -p '{"metadata":{"annotations":{"konghq.com/override":"echo-service-config"}}}'
```

Making a request

```bash
curl -i localhost:10080/demo/hello \
    -H "host: api.example.com"
```

Result

```json
1. Request path "/echo/hello" are passed to service.
2. Define "/echo" as the basepath of service.
```

---

## Protect API by API Key

***1. Setup API Key Authentication on Service***

Create a `KongPlugin` "**Key Authentication**" and apply to a `Service`

```bash
kubectl apply -f ./yaml/apikey-plugin.yml

kubectl patch svc echo -p '{"metadata":{"annotations":{"konghq.com/plugins":"apikey-plugin"}}}'
```

Making a request

```bash
curl -i localhost:10080/demo/hello \
    -H "host: api.example.com"
```

Result

```json
1. Return "401 Unauthorized".
2. Return {"message":"No API key found in request"}.
```

--

***2. Create a KongConsumer and setup a API Key Secret***

Generate a random API Key

```bash
export APIKEY=$(openssl rand -base64 24)

echo $APIKEY
```

Create a `Secret` for the API Key

```bash
kubectl create secret generic dennis-apikey  \
  --from-literal=kongCredType=key-auth  \
  --from-literal=key=$APIKEY
```

Create a `KongConsumer` with API Key Secret

```bash
kubectl apply -f ./yaml/dennis-consumer.yml
```

Making a request with credential

```bash
curl -i localhost:10080/demo/hello \
    -H "host: api.example.com" \
    -H "x-api-key: $APIKEY"
```

Result

```json
1. Return "200 OK".
2  There are "consumer-custom-id" and "consumer-username" of the current consumer in the header.
3. Successful access service with a credential.
```

---

## Limit API traffic

Create a `KongPlugin` "**Rate Limit**" and apply to a `Ingress`

```bash
kubectl apply -f ./yaml/ratelimit-plugin.yml

kubectl patch ing api-ingress -p '{"metadata":{"annotations":{"konghq.com/plugins":"ratelimit-plugin"}}}'
```

Making 5 requests in a minute

```bash
curl -i localhost:10080/demo/hello \
    -H "host: api.example.com" \
    -H "x-api-key: $APIKEY"
```

Result

```json
1. Check the rate-limit counter in the response header.
2. After 5 requests in a minute, return { "message": "API rate limit exceeded" }.
```

Wait a while (accroding to the header "**RateLimit-Reset**") and making a request again

```bash
curl -i localhost:10080/demo/hello \
    -H "host: api.example.com" \
    -H "x-api-key: $APIKEY"
```

Result

```json
1. Check the "RateLimit-Reset" in the response header.
2. API quota reset every minute.
```

---

## Restrict IP Addresses of API consumer

***1. Create a api consumer for the vendor***

Generate a random API Key for the vendor

```bash
export APIKEY_V=$(openssl rand -base64 24)

echo $APIKEY_V
```

Create a `Secret` for the API Key of the vendor

```bash
kubectl create secret generic vendor-apikey  \
  --from-literal=kongCredType=key-auth  \
  --from-literal=key=$APIKEY_V
```

Create a `KongConsumer` with API Key Secret for the vendor

```bash
kubectl apply -f ./yaml/vendor-consumer.yml
```

Making a request with the credential of the vendor

```bash
curl -i localhost:10080/demo/hello \
    -H "host: api.example.com" \
    -H "x-api-key: $APIKEY_V"
```

Result

```json
1. Return "200 OK".
2. Successful access service with a credential of the vendor.
```

--

***2. Enable IP address restriction on the vendor api consumer***

Create a `KongPlugin` "**IP Restriction**" with wrong IP address and apply to a `KongConsumer` "**vendor-consumer**"

```bash
kubectl apply -f ./yaml/ip-restriction-plugin.yml

kubectl patch kongconsumer vendor-consumer --type="merge" -p '{"metadata":{"annotations":{"konghq.com/plugins":"ip-restriction-plugin"}}}'
```

Making a request with the credential of the vendor

```bash
curl -i localhost:10080/demo/hello \
    -H "host: api.example.com" \
    -H "x-api-key: $APIKEY_V"
```

Result

```json
1. Return "403 Forbidden".
2. Return {"message":"Your IP address is not allowed"}.
```

Making a request with the credential of non-restricted consumer

```bash
curl -i localhost:10080/demo/hello \
    -H "host: api.example.com" \
    -H "x-api-key: $APIKEY"
```

Result

```json
1. Return "200 OK".
2. IP Restriction only applies to the vendor consunmer.
```

Update the `KongPlugin` "**ip-restriction-plugin**" with correct IP address

```bash
export IP=$(ipconfig getifaddr en0)

echo $IP

kubectl patch kongplugin ip-restriction-plugin --type="merge" -p '{"config":{"allow":["127.0.0.1"]}}'
```

Making a request with the credential of the vendor

```bash
curl -i localhost:10080/demo/hello \
    -H "host: api.example.com" \
    -H "x-api-key: $APIKEY_V"
```

Result

```json
1. Return "200 OK".
2. Successful access service from the allowed IP address.
```

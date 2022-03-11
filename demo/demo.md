# k4k8s demo

## Setup Backend Services

Deploy "**echoserver**" and "**httpbin**" service

```bash
kubectl apply -f ./demo/yaml/httpbin.yml

kubectl apply -f ./demo/yaml/echo.yml

kubectl get all -A
```

Test "**httpbin**" service

Open a new terminal and start port forwarding

```bash
kubectl port-forward svc/httpbin 11080:80
```

Making a request

```bash
curl -i localhost:11080/status/200
```

Test "**echoserver**" service

Open a new terminal and start port forwarding

```bash
kubectl port-forward svc/echo 11880:8080
```

Making a request

```bash
curl -i localhost:11880/echo
```

---

## Proxy service by Kong

***1. Setup an Ingress to use Kong Ingress Controller***

Create an `Ingress` with **IngressClass: kong**

```bash
kubectl apply -f ./demo/yaml/api-ingress.yml
```

Open a new terminal and start port forwarding

```bash
kubectl port-forward -n kong service/kong-proxy 10080:80
```

Making requests

```bash
curl -i localhost:10080/v1/httpbin/status/200 \
    -H "host: api.example.com"

curl -i localhost:10080/v1/echo/hello \
    -H "host: api.example.com"
```

Result

```json
1. Handle request by Kong Ingress Controller and proxy request by Kong Gateway.
2. Request path "/v1/httpbin/status/200" are passed to service but return "404 NOT FOUND".
3. Request path "/v1/echo/hello" are passed to service with real path "/v1/echo/hello".
```

--

***2. Use KongIngress with Ingress resource***

Create a `KongIngress` and apply to an `Ingress`

```bash
kubectl apply -f ./demo/yaml/api-ingress-config.yml

kubectl patch ing/api-ingress -p '{"metadata":{"annotations":{"konghq.com/override":"api-ingress-config"}}}'
```

Making requests

```bash
curl -i localhost:10080/v1/httpbin/status/200 \
    -H "host: api.example.com"

curl -i localhost:10080/v1/echo/hello \
    -H "host: api.example.com"
```

Result

```json
1. Request path "/v1/httpbin/status/200" are passed to service and returns "200 OK" as expected.
2. Request path "/v1/echo/hello" are passed to service with real path "/hello".
3. The basepaths "/v1/httpbin" and "/v1/echo" defined in the "Ingress" are stripped.
4. Expose custom request basepath "/v1/httpbin" and "/v1/echo" for backend services.
5. But all request sent to backend service is matched from basepath "/".
```

--

***3. Use KongIngress with Service resource***

Create `KongIngress` "**echo-service-config**" and apply to `Service` "**echo**"

```bash
kubectl apply -f ./demo/yaml/echo-service-config.yml

kubectl patch svc/echo -p '{"metadata":{"annotations":{"konghq.com/override":"echo-service-config"}}}'
```

Create `KongIngress` "**httpbin-service-config**" and apply to `Service` "**httpbin**"

```bash
kubectl apply -f ./demo/yaml/httpbin-service-config.yml

kubectl patch svc/httpbin -p '{"metadata":{"annotations":{"konghq.com/override":"httpbin-service-config"}}}'
```

Making requests

```bash
curl -i localhost:10080/v1/httpbin/status/200 \
    -H "host: api.example.com"

curl -i localhost:10080/v1/echo/hello \
    -H "host: api.example.com"
```

Result

```json
1. Request path "/v1/httpbin/status/200" are passed to service with real path "/status/200".
2. Define "/" as the basepath of "httpbin" service.
3. Request path "/v1/echo/hello" are passed to service with real path "/echoserver/hello".
4. Define "/echoserver" as the basepath of "echo" service.
```

---

## Protect API by API Key

***1. Setup API Key Authentication on Service***

Create a `KongPlugin` "**Key Authentication**" and apply to a `Service`

```bash
kubectl apply -f ./demo/yaml/apikey-plugin.yml

kubectl patch svc/echo -p '{"metadata":{"annotations":{"konghq.com/plugins":"apikey-plugin"}}}'
```

Making a request to the "**echo**" service

```bash
curl -i localhost:10080/v1/echo/hello \
    -H "host: api.example.com"
```

Result

```json
1. Return "401 Unauthorized".
2. Return {"message":"No API key found in request"}.
3. The "echo" service is protected by API Key.
```

Making a request to the "**httpbin**" service

```bash
curl -i localhost:10080/v1/httpbin/status/200 \
    -H "host: api.example.com"
```

Result

```json
1. Return "200 OK" as normal.
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
kubectl apply -f ./demo/yaml/dennis-consumer.yml
```

Making a request with credential

```bash
curl -i localhost:10080/v1/echo/hello \
    -H "host: api.example.com" \
    -H "x-api-key: $APIKEY"
```

Result

```json
1. Return "200 OK".
2  There are "consumer-custom-id" and "consumer-username" of the current consumer in the header.
3. Successful access "echo" service with a credential.
```

---

## Limit API traffic

Create a `KongPlugin` "**Rate Limit**" and apply to a `Ingress`

```bash
kubectl apply -f ./demo/yaml/ratelimit-plugin.yml

kubectl patch ing/api-ingress -p '{"metadata":{"annotations":{"konghq.com/plugins":"ratelimit-plugin"}}}'
```

Making 6 requests in a minute

```bash
curl -i localhost:10080/v1/echo/hello \
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
curl -i localhost:10080/v1/echo/hello \
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
kubectl apply -f ./demo/yaml/vendor-consumer.yml
```

Making a request with the credential of the vendor

```bash
curl -i localhost:10080/v1/echo/hello \
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
kubectl apply -f ./demo/yaml/ip-restriction-plugin.yml

kubectl patch kc/vendor-consumer --type="merge" -p '{"metadata":{"annotations":{"konghq.com/plugins":"ip-restriction-plugin"}}}'
```

Making a request with the credential of the vendor

```bash
curl -i localhost:10080/v1/echo/hello \
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
curl -i localhost:10080/v1/echo/hello \
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
kubectl patch kp/ip-restriction-plugin --type="merge" -p '{"config":{"allow":["127.0.0.1"]}}'
```

Making a request with the credential of the vendor

```bash
curl -i localhost:10080/v1/echo/hello \
    -H "host: api.example.com" \
    -H "x-api-key: $APIKEY_V"
```

Result

```json
1. Return "200 OK".
2. Successful access service from the allowed IP address.
```

---

## Setup Backend Services Health Check and Circuit-Breaker

Add `Upstream` config with health check for "**httpbin**" service

```bash
kubectl apply -f ./demo/yaml/httpbin-service-hc-config.yml
```

Make 2 request to "**httpbin**" and return "HTTP Status 500"

```bash
curl -i localhost:10080/v1/httpbin/status/500 \
    -H "host: api.example.com"
```

Make a request to "**httpbin**" and return "HTTP Status 200"

```bash
curl -i localhost:10080/v1/httpbin/status/200 \
    -H "host: api.example.com"
```

Result

```json
1. Return "200 OK".
2. Kong has not short-circuited because there were only two consecutive failures.
```

Make 3 request to "**httpbin**" and return "HTTP Status 500"

```bash
curl -i localhost:10080/v1/httpbin/status/500 \
    -H "host: api.example.com"
```

Make a request to "**httpbin**" and return "HTTP Status 200"

```bash
curl -i localhost:10080/v1/httpbin/status/200 \
    -H "host: api.example.com"
```

```json
1. Return "503 Service Temporarily Unavailable".
2. Kong has short-circuited because there were three consecutive failures.
3. Wait for 15 seconds to retry again, return "200 OK".
```
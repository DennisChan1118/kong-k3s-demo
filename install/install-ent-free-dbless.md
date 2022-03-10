# k3d + k4k8s

## Install Tools

kubectl

```bash
brew install kubectl
```

docker

<https://docs.docker.com/get-docker/>

k3d

```bash
brew install k3d
```

---

## Install k3s cluster by k3d

Install k3s cluster

```bash
k3d cluster create kong-ent-free-dbless \
    --k3s-arg "--disable=traefik@server:0"
```

Check result

```bash
docker ps

kubectl get nodes

kubectl get all -A
```

---

## Install Kong for Kubenetes

Install Kong Enterprise in Kubernetes with dbless mode

```bash
kubectl apply -f ./yaml/k4k8s-ent-free-dbless.yml

kubectl get all -A
```

## Test Kong Gateway

Open a new terminal and start port forwarding

```bash
kubectl port-forward -n kong service/kong-proxy 10080:80
```

Test connection

```bash
curl -i localhost:10080
```

You will get a JSON response

```json
{"message":"no Route matched with those values"}
```

## Test Kong Manager

Open a new terminal and start port forwarding

```bash
kubectl port-forward -n kong service/kong-manager 18002:8002
```

Open Kong Manager url in the browser

<http://localhost:18002/dashboard>

Login super admin by password created early. You will enter the Kong Manager dashbroad page.

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
k3d cluster create kong-oss-dbless \
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

k4k8s-oss-dbless

```bash
kubectl apply -f ./yaml/k4k8s-oss-dbless.yml

kubectl get all -A
```

Test :

Open a new terminal and start port forwarding

```bash
kubectl port-forward -n kong service/kong-proxy 11080:80
```

Test connection

```bash
curl -i localhost:11080
```

You will get a JSON response

```json
{"message":"no Route matched with those values"}
```

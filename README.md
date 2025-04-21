# kubeflow-kind-local-setup
Small tutorial to get a local version of kubeflow running

## Prerequisites for Docker Desktop (Settings->Resources)
- 16 GB of RAM recommended.
- 8 CPU cores recommended.
- `kind` version 0.27+.
- `docker` or a more modern tool such as `podman` to run the OCI images for the Kind cluster.
- Linux kernel subsystem changes to support many pods:
    - `sudo sysctl fs.inotify.max_user_instances=2280`
    - `sudo sysctl fs.inotify.max_user_watches=1255360`
 
## step by step tutorial
(adapted from https://github.com/kubeflow/manifests/tree/master?tab=readme-ov-file#install-with-a-single-command)

1. Clone kubeflow manifests repo and cd into it
```
$ git clone https://github.com/kubeflow/manifests/tree/master

$ cd manifests
```

2. Create Kind Cluster
```sh
cat <<EOF | kind create cluster --name=kubeflow --config=-
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  image: kindest/node:v1.32.0@sha256:c48c62eac5da28cdadcf560d1d8616cfa6783b58f0d94cf63ad1bf49600cb027
  kubeadmConfigPatches:
  - |
    kind: ClusterConfiguration
    apiServer:
      extraArgs:
        "service-account-issuer": "https://kubernetes.default.svc"
        "service-account-signing-key-file": "/etc/kubernetes/pki/sa.key"
EOF
```

3. Save Kubeconfig
```sh
kind get kubeconfig --name kubeflow > /tmp/kubeflow-config
export KUBECONFIG=/tmp/kubeflow-config
```

4. Create a Secret Based on Existing Credentials to Pull the Images
```sh
docker login

kubectl create secret generic regcred \
    --from-file=.dockerconfigjson=$HOME/.docker/config.json \
    --type=kubernetes.io/dockerconfigjson
```

5. Copy the `kustomization-xxx.yaml` (based on your needs) from this repo and paste it into your `example/kustomization.yaml` to allow only the jupyter notebook functionalities.


6. Then install all Kubeflow components using this `example/kustomization.yaml`
```sh
while ! kustomize build example | kubectl apply --server-side --force-conflicts -f -; do echo "Retrying to apply resources"; sleep 20; done
```

7. Make sure to check if all required services are online
```
kubectl get pods -n cert-manager
kubectl get pods -n istio-system
kubectl get pods -n auth
kubectl get pods -n oauth2-proxy
kubectl get pods -n knative-serving
kubectl get pods -n kubeflow
kubectl get pods -n kubeflow-user-example-com
```

8. Access the dashboard by port forward
```
$ kubectl port-forward svc/istio-ingressgateway -n istio-system 8080:80
```

9. Log in with default user name `user@example.com` and the password `12341234`.

10. Allow Jupyter notebook access to ml pipelines
```
$ kubectl apply -f access-kfp-notebook-access.yaml
```

11. Add minio to cluster

```
$ kubectl apply -f minio-deployment.yaml
```

```
$ kubectl port-forward -n kubeflow-user-example-com svc/minio 9000:9000 9090:9090
```

log in with username `minioDev` and pw `minioDevPass123` 


12. Congratulations! You can now start experimenting and running your end-to-end ML workflows with Kubeflow and Minio
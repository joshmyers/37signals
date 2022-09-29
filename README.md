<details>
<summary> Running a local development Minikube cluster </summary>

#### Context
We can build our application into a Docker image and deploy it to a local
Minikube cluster for testing.

Note that our Dockerfile is setting `RAILS_ENV=production` as a default when
building our image for Minikube. Why? Because RAILS_ENV changes application 
behaviour and we want to simulate production as much as possible inside our 
cluster. With N pods of our application running inside our Minikube cluster,
a local Sqlite database per pod isn't going to cut it.

#### Requirements
* Docker

#### Minikube setup
We install binaries into `~/.local/bin` so as to not require sudo permissions.
Subsequent Rake tasks assume binaries are in this location, so you may want
to add it to your `$PATH`.

```bash 
❯ bundle exec rake kubectl:install
❯ bundle exec rake minikube:install
```

#### Starting Minikube
```bash 
❯ bundle exec rake minikube:start
😄  minikube v1.27.0 on Debian bullseye/sid
🆕  Kubernetes 1.25.0 is now available. If you would like to upgrade, specify: --kubernetes-version=v1.25.0
✨  Using the docker driver based on existing profile
👍  Starting control plane node minikube in cluster minikube
🚜  Pulling base image ...
🏃  Updating the running docker "minikube" container ...
🐳  Preparing Kubernetes v1.22.3 on Docker 20.10.17 ...
🔎  Verifying Kubernetes components...
    ▪ Using image gcr.io/k8s-minikube/storage-provisioner:v5
🌟  Enabled addons: storage-provisioner, default-storageclass

❗  /home/josh/.local/bin/kubectl is version 1.25.2, which may have incompatibilites with Kubernetes 1.22.3.
    ▪ Want kubectl v1.22.3? Try 'minikube kubectl -- get pods -A'
🏄  Done! kubectl is now configured to use "minikube" cluster and "default" namespace by default
```

At this point we should be able to see all of the pods running in our cluster,
in the `kube-system` namespace.
```
❯ ~/.local/bin/kubectl get pods --all-namespaces
NAMESPACE     NAME                               READY   STATUS    RESTARTS        AGE
kube-system   coredns-78fcd69978-tplgx           1/1     Running   2 (6m23s ago)   1h7m
kube-system   etcd-minikube                      1/1     Running   3 (6m28s ago)   1h7m
kube-system   kube-apiserver-minikube            1/1     Running   2 (6m38s ago)   1h7m
kube-system   kube-controller-manager-minikube   1/1     Running   3 (6m28s ago)   1h7m
kube-system   kube-proxy-7xkts                   1/1     Running   3 (6m28s ago)   1h7m
kube-system   kube-scheduler-minikube            1/1     Running   3 (6m28s ago)   1h7m
kube-system   storage-provisioner                1/1     Running   4 (6m28s ago)   1h7m
```

#### Seeding Kubernetes secrets
Because our Dockerised app defaults to using `RAILS_ENV=production`, we need to
ensure the `SECRET_KEY_BASE` environment variable is set and passed to the app.
To do this we generate a secure string and store as a Kubernetes secret, that 
the application has access to.

In reality k8s secrets are just base64 encoded strings, so in production, we'd
likely want to use something else e.g. [Hashicorp Vault](https://github.com/hashicorp/vault) or [Bitnami Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets)

</details>

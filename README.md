## Minikube
<details>
<summary> Running a local development Kubernetes cluster </summary>
<br>
We can build our application into a Docker image and deploy it to a local
Minikube cluster for testing.

Note that our Dockerfile is setting `RAILS_ENV=production` as a default when
building our image for Minikube. Why? Because RAILS_ENV changes application 
behaviour and we want to simulate production as much as possible inside our 
cluster. With N pods of our application running inside our Minikube cluster,
a local Sqlite database per pod isn't going to cut it.

## Requirements
* Docker

## The tl;dr version
#### Bootstrap
```
❯ bundle exec rake kubectl:install
❯ bundle exec rake minikube:install
❯ bundle exec rake minikube:start
❯ bundle exec rake app:seed_secret_key_base
❯ bundle exec rake app:apply
❯ bundle exec rake app:db_setup
```
#### Development workflow to deploy local changes
```
❯ bundle exec rake app:deploy
```

## The longer version

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
the application has access to:
```
❯ bundle exec rake app:seed_secret_key_base
secret/secret-key-base created
```

In reality k8s secrets are just base64 encoded strings, so in production, we'd
likely want to use something else for secrets management e.g. [Hashicorp Vault](https://github.com/hashicorp/vault) or [Bitnami Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets).


#### Applying our Kubernetes manifests
Next up is deploying our k8s manifests to the cluster to create our Service and
Deployment.
```
❯ bundle exec rake app:apply
Sending build context to Docker daemon  7.401MB
Step 1/28 : ARG RUBY_VERSION=2.7.1
Step 2/28 : ARG BASE_IMAGE=ruby:$RUBY_VERSION-alpine3.11
Step 3/28 : ARG RAILS_ENV=production
Step 4/28 : ARG NODE_ENV=production
Step 5/28 : FROM $BASE_IMAGE AS base
2.7.1-alpine3.11: Pulling from library/ruby
** SNIP **
Successfully built d6a25ad15d6c
Successfully tagged ops-heycamp-josh:latest
deployment.apps/ops-heycamp-josh created
service/ops-heycamp-josh created
deployment.apps/mysql created
service/mysql created
persistentvolumeclaim/mysql-pv-claim created
==> Service available at http://192.168.0.1:32029
```

This will build our application image within Minikube Docker context and apply
our manifests.

We can see the final line of output contains the endpoint we can reach our 
service on, in this case `http://192.168.0.1:32029`

If we look at the pods now deployed to our cluster:
```
❯ ~/.local/bin/kubectl get pods --all-namespaces
NAMESPACE     NAME                                READY   STATUS    RESTARTS        AGE
default       mysql-b94cb54c5-wdkqf               1/1     Running   0               3m28s
default       ops-heycamp-josh-55b94874db-57vwc   0/1     Running   3 (3m7s ago)    3m28s
default       ops-heycamp-josh-55b94874db-5ljzd   0/1     Running   3 (3m7s ago)    3m28s
default       ops-heycamp-josh-55b94874db-7j6zf   0/1     Running   3 (3m7s ago)    3m28s
default       ops-heycamp-josh-55b94874db-ck6qw   0/1     Running   3 (3m7s ago)    3m28s
default       ops-heycamp-josh-55b94874db-fd4wk   0/1     Running   3 (3m7s ago)    3m28s
default       ops-heycamp-josh-55b94874db-hlv5b   0/1     Running   3 (3m10s ago)   3m28s
default       ops-heycamp-josh-55b94874db-st5f7   0/1     Running   3 (3m6s ago)    3m28s
default       ops-heycamp-josh-55b94874db-wb8zt   0/1     Running   3 (3m7s ago)    3m28s
default       ops-heycamp-josh-55b94874db-xswmd   0/1     Running   3 (3m7s ago)    3m28s
default       ops-heycamp-josh-55b94874db-zgdpr   0/1     Running   3 (3m7s ago)    3m28s
kube-system   coredns-78fcd69978-tplgx            1/1     Running   2 (31m ago)     5h32m
kube-system   etcd-minikube                       1/1     Running   3 (31m ago)     1h32m
kube-system   kube-apiserver-minikube             1/1     Running   2 (31m ago)     1h32m
kube-system   kube-controller-manager-minikube    1/1     Running   3 (31m ago)     1h32m
kube-system   kube-proxy-7xkts                    1/1     Running   3 (31m ago)     1h32m
kube-system   kube-scheduler-minikube             1/1     Running   3 (31m ago)     1h32m
kube-system   storage-provisioner                 1/1     Running   4 (31m ago)     1h32m
```
We can see our application and db pods running. The application is running but
failing readiness checks, because we haven't created our DB yet, let's do that
now.


#### DB setup/migrations
```
❯ bundle exec rake app:db_setup
Created database 'ops-heycamp-josh_production'
D, [2022-09-29T13:48:59.975568 #31] DEBUG -- :    (0.2ms)  SET  @@SESSION.sql_mode = CONCAT(CONCAT(@@sql_mode, ',STRICT_ALL_TABLES'), ',NO_AUTO_VALUE_ON_ZERO'),  @@SESSION.sql_auto_is_null = 0, @@SESSION.wait_timeout = 2147483
-- create_table("microposts", {:force=>:cascade})
D, [2022-09-29T13:48:59.980900 #31] DEBUG -- :    (0.2ms)  SET  @@SESSION.sql_mode = CONCAT(CONCAT(@@sql_mode, ',STRICT_ALL_TABLES'), ',NO_AUTO_VALUE_ON_ZERO'),  @@SESSION.sql_auto_is_null = 0, @@SESSION.wait_timeout = 2147483
** SNIP**
D, [2022-09-29T13:49:09.892606 #31] DEBUG -- :    (2.3ms)  COMMIT
D, [2022-09-29T13:49:09.892763 #31] DEBUG -- :    (0.1ms)  BEGIN
D, [2022-09-29T13:49:09.893725 #31] DEBUG -- :   SQL (0.1ms)  INSERT INTO `relationships` (`follower_id`, `followed_id`, `created_at`, `updated_at`) VALUES (41, 1, '2022-09-29 13:49:09', '2022-09-29 13:49:09')
D, [2022-09-29T13:49:09.895993 #31] DEBUG -- :    (2.2ms)  COMMIT
```

Let's have another look at our running pods:
```
❯ ~/.local/bin/kubectl get pods --all-namespaces
NAMESPACE     NAME                                READY   STATUS    RESTARTS        AGE
default       mysql-b94cb54c5-wdkqf               1/1     Running   0               8m13s
default       ops-heycamp-josh-55b94874db-57vwc   1/1     Running   3 (7m52s ago)   8m13s
default       ops-heycamp-josh-55b94874db-5ljzd   1/1     Running   3 (7m52s ago)   8m13s
default       ops-heycamp-josh-55b94874db-7j6zf   1/1     Running   3 (7m52s ago)   8m13s
default       ops-heycamp-josh-55b94874db-ck6qw   1/1     Running   3 (7m52s ago)   8m13s
default       ops-heycamp-josh-55b94874db-fd4wk   1/1     Running   3 (7m52s ago)   8m13s
default       ops-heycamp-josh-55b94874db-hlv5b   1/1     Running   3 (7m55s ago)   8m13s
default       ops-heycamp-josh-55b94874db-st5f7   1/1     Running   3 (7m51s ago)   8m13s
default       ops-heycamp-josh-55b94874db-wb8zt   1/1     Running   3 (7m52s ago)   8m13s
default       ops-heycamp-josh-55b94874db-xswmd   1/1     Running   3 (7m52s ago)   8m13s
default       ops-heycamp-josh-55b94874db-zgdpr   1/1     Running   3 (7m52s ago)   8m13s
kube-system   coredns-78fcd69978-tplgx            1/1     Running   2 (36m ago)     1h37m
kube-system   etcd-minikube                       1/1     Running   3 (36m ago)     1h37m
kube-system   kube-apiserver-minikube             1/1     Running   2 (36m ago)     1h37m
kube-system   kube-controller-manager-minikube    1/1     Running   3 (36m ago)     1h37m
kube-system   kube-proxy-7xkts                    1/1     Running   3 (36m ago)     1h37m
kube-system   kube-scheduler-minikube             1/1     Running   3 (36m ago)     1h37m
kube-system   storage-provisioner                 1/1     Running   4 (36m ago)     1h37m
```
Great, the application is now in a ready state. We can now load up the app in a
browser at `http://192.168.0.1:32029` and confirm that health checks are
passing:
```
❯ curl http://192.168.0.1:32029/health/liveness
{"ok":true,"description":"Service is up and running"}

❯ curl http://192.168.0.1:32029/health/readiness
{"ok":true,"description":"Service and DB are up and running"}
```

#### Local develoment workflow
OK, so I now want to make some local changes and deploy those out to Minikube,
how can I do this?
```
❯ bundle exec rake app:deploy
Sending build context to Docker daemon  7.401MB
Step 1/28 : ARG RUBY_VERSION=2.7.1
Step 2/28 : ARG BASE_IMAGE=ruby:$RUBY_VERSION-alpine3.11
Step 3/28 : ARG RAILS_ENV=production
Step 4/28 : ARG NODE_ENV=production
Step 5/28 : FROM $BASE_IMAGE AS base
** SNIP **
Successfully built d6a25ad15d6c
Successfully tagged ops-heycamp-josh:latest
deployment.apps/ops-heycamp-josh patched
```

This will rebuild our image with local changes and deploy it to Minikube using
a rolling update strategy to ensure no downtime.

</details>

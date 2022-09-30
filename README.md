## Beta deployment work
<details>
<summary> Deploying to EKS with a MySQL backend </summary>
<br>

## MySQL
In 57542a4 we switched to using MySQL if RAILS_ENV=production. In order to use
a MySQL backend for all environments, we need to:
* Add similar changes to `config/database.yaml` for development and testing
  environments.
* Remove sqlite3 from the development and test group in our Gemfile, replace
  with mysql2 (not just for the production group).
* As this adds a new dependency, rather than a flat sqlite3 DB file, we
  probably want a way to stand up MySQL for local testing (before Minikube)
  so we may want to add a docker-compose.yaml to spin up a local MySQL
  container.
* Remove sqlite3 flat files from `/db` directory.


## Provisioning
There are plenty of good off the shelf, generalised and flexible Terraform
modules[1], as well as documentation, to provision EKS clusters with 
managed/unmanaged node pools. As a first pass, I'd see how far we can get 
using one of these, rather than writing a custom module.

Once we have our EKS cluster up and running, we also need our database. Again,
I'd reach for an off the shelf RDS module[2] to provision Aurora MySQL.

Assuming we have working EKS/RDS clusters, with a valid kube config and AWS
credentials to access the cluster, we can now deploy our application to the
EKS cluster.

Bonus points if these resources are provisioned using a CI/CD pipeline/tool 
like Atlantis[3], rather than being applied locally. For provisioning across
multiple environments/regions, Terragrunt[4] can help keep our Terraform 
configuration DRY. Additional bonus points for Terratest/OPA tests for the
Terraform module, even if it only ends up being a wrapper module around a few
upstream module invocations.


## Building and pushing our image to ECR
In order to deploy our application to EKS, we need to push the image to a 
remote repo that the EKS kubelet can pull from. Ideally we don't use Dockerhub
due to rate limiting. ECR should suffice.

The current `app:build` Rake task will build our Docker image using Minikubes
Docker socket. If we want to build against our local Docker socket, tag and 
push to a remote repo, we probably want to change this behaviour to build 
locally and `minikube image load $image:$tag` instead. Then something like:

```bash
docker build -t ops-heycamp-josh .
docker tag $remote_repo/ops-heycamp-josh:latest
docker push $remote_repo/ops-heycamp-josh:latest
```

Ideally this is done in a build pipeline, rather than local, when master merges
/Github release happen. If we have isolated enough environments we may be able
to spin up applications in EKS for branch builds too, so may want to push on
every commit.


## Deloying
Applying raw k8s manifests maybe good enough for our local Minikube cluster, but
when it comes to deploying actual applications to production we want to manage 
these manifests more intelligently.

While there are many deployment tools to provide higher layers of abstraction,
Helm[5] is probably the most widely used. Helm is a package manager for 
Kubernetes. It helps deploy complex application by bundling necessary resources
into Charts, which contain all the information to run images in a cluster.
Helm provides templating functionality in Charts using values files and
understands the concept of a release.

We may already have a Helm chart for deploying RoR applications, where we can
plug in some values and have our resources installed to the cluster. If we 
don't, we need to create the chart using `helm create` and then deploy the 
chart using `helm upgrade --install`. Again, plenty of docs out there how to
deploy to Kubernetes using Helm.


## TODO
* Multiarch Docker builds. Do we want to build multi arch images (manifests)
  for e.g. Linux amd64/Darwin ARM (read: M1/M2)
* The application currently connects to `mysql` as the root user. No bueno.
  We should create a new MySQL user with limited permissions and have the 
  application connect as that user.
* Do we have a production go live checklist?
* Application logs to STDOUT, so we can fetch logs via `kubectl logs` but we
  probably want to ship these logs somewhere. Maybe a SaaS offering like 
  DataDog. To do this you could run a datadog agent image as a daemon set (so
  we get one per node) in the cluster or perhaps something like Fluentd for
  future flexibility about where we want these outputted to.
* What security concerns do we have for this application? Are there other 
  applications/services in the cluster that we don't want it to be able to talk
  to? If so, look into network policies.
* If we want CarrierWave to upload images to S3 using fog (see dbe35ad) we need
  an IAM role for the service account and an IAM cluster role binding for our 
  application.
* Pass in our RDS host (or a CNAME for it) to our application environment as
  `DB_HOST`.
* How are we monitoring our application/gathering application and business level
  metrics? We could deploy the Prometheus operator[6] and add annotations to our
  application deloyment/service to have a `/metrics` endpoint scraped as well as
  HTTP/Kubernetes service endpoint probes etc. See [7] for instrumentation 
  documentation. We can also monitor our Aurora instance using a Prometheus RDS
  exporter and if we have unmanaged EKS node pools, the Blackbox node exporter
  can provide node level instrumentation. Wherever we have monitoring, we 
  should also have alerting e.g. Alert manager. DataDog could also be used here
  but $$$.
* RDS Backup/restore process. What is it? How often do we practice restores?
* Do we have a run book for what an on-call engineer is expected to do should
  they get paged about this application?
* Define the business level impact for this application, should there be issues.
* Do we have SLOs/SLAs for this application? How were they determined, what is
  the impact if they are breached?
* Do we have Horizontal/Vertical Pod Autoscalers (to scale up the number of replicas)
  and or Node Autoscaler if running with unmanaged nodes?
* Do we want to remove the DB dependency check from our application readiness
  probe? See 8f58536
* Do we have RBAC setup in the cluster?
* What egress do this application require? Lock down egress from the application via
  network policies or security groups. Access to S3 could be provided by a 
  VPC endpoint, so not traversing the internet.


[1]
https://github.com/cloudposse/terraform-aws-eks-cluster
https://github.com/terraform-aws-modules/terraform-aws-eks

[2]
https://github.com/terraform-aws-modules/terraform-aws-rds-aurora
https://github.com/cloudposse/terraform-aws-rds-cluster

[3]
https://github.com/runatlantis/atlantis

[4]
https://github.com/gruntwork-io/terragrunt

[5]
https://github.com/helm/helm

[6]
https://github.com/prometheus-operator/prometheus-operator

[7]
https://github.com/prometheus/client_ruby

</details>

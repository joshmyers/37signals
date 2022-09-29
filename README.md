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
<details>
<summary> Installing Kubectl and Minikube to ~/.local/bin </summary>

```bash 
rake kubectl:install
rake minikube:install
```
</details>



</details>

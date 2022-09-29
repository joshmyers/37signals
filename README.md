## Minikube

We can build our application into a Docker container and deploy it to a local Minikube cluster for testing.

<details>
<summary> There are some helper Rake tasks to do this </summary>

```bash 
rake kubectl:install                    # Install Kubectl to ~/.local/bin/kubectl
rake minikube:dashboard                 # Start Minikube Dashboard
rake minikube:install                   # Install Minikube to ~/.local/bin/minikube
rake minikube:purge                     # Purge Minikube
rake minikube:start                     # Start Minikube
rake minikube:stop                      # Stop Minikube
rake app:apply                          # Apply manifests to Minikube
rake app:build                          # Build App image
rake app:db_migrate                     # Run db:migrate in Minikube
rake app:db_seed                        # Run db:seed in Minikube
rake app:db_setup                       # Run db:setup in Minikube
rake app:deploy                         # Deploy app to Minikube
rake app:seed_secret_key_base           # Seed secret-key-base secret
```
</details>

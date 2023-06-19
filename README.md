# A Blueprint for building a DBaaS for an Internal Developer Platform
### (demo for MDB.local NYC 2023)

---
## Prerequisites
1. A running Kubernetes cluster
2. Atlas CLI installed locally (v1.7 or higher) and setup with your Atlas account

   (There are alternative ways to install the Atlas Kubernetes Operator - via Helm or Kubectl)

## Step 1 - Installing the Atlas Kubernetes Operator
1. Install the operator via the Atlas cli
```properties
atlas kubernetes operator install
```
2. Verify installation and setup
```properties
kubectl get pods
```
```properties
kubectl get secrets
```

## Step 2 [Optional] - Deploy Atlas resources manually
1. Create an AtlasProject Custom Resource
```properties
kubectl apply -f dbaas/atlas-project.yaml
```
2. Create an AtlasCluster Custom Resource
```properties
kubectl apply -f dbaas/atlas-mongodb-cluster.yaml
```

## Step 3 - Deploy Atlas resources through GitOps with ArgoCD
1. Install ArgoCD
```properties
kubectl create namespace argocd
```
```properties
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```
2. Create the Argo Application
   
Edit the source in `gitops/argo-application.yaml` to the repository, revision, and directory to be watched for dbaas file changes
```properties
  source:
    repoURL: https://github.com/giohan/dotlocaldemo.git
    targetRevision: HEAD
    path: ./dbaas/
 ```
Deploy the Application Custom Resource
```properties
kubectl apply -f gitops/argo-application.yaml
``` 
3. Create Atlas resources through the git IaC repository
   (these steps can be performed without cluster access)
```properties
git add dbaas/atlas-project.yaml
git commit -m 'gitops project'
git push -u origin main
``` 
```properties
git add dbaas/atlas-mongodb-cluster.yaml
git commit -m 'dev mongodb cluster'
git push -u origin main
``` 

## Step 4 - Connect to the database
1. Create a secret with a password to log into the Atlas cluster
```properties
kubectl create secret generic the-user-password --from-literal="password=P@@sword%"  # change this to your password
kubectl label secret the-user-password atlas.mongodb.com/type=credentials
``` 
2. Create the user through the git IaC repository
```properties
git add dbaas/dbuser.yaml
git commit -m 'database user'
git push -u origin main
``` 
3. Retrieve the secret that Atlas Kubernetes Operator created to connect to the cluster.
The secret is created with a name pattern `{project}-{cluster}-{database-user}`
```properties
kubectl get secret atlas-project-first-cluster-dbuser -o json | jq -r '.data | with_entries(.value |= @base64d)';
``` 
4. Use the secret as an environment variable for your deployments. Example: `sample-app/hello-atlas.yaml`
```properties
    env:
        - name: "MDB_CONNECTION_STRING"
        valueFrom:
            secretKeyRef:
                name: atlas-project-first-cluster-dbuser
                key: connectionStringStandardSrv
 ``` 

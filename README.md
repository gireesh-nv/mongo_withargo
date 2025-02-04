ArgoCD setup in KIND

1) **STEP 1**: First, isntall the kind cluster with given conf file
- Download config file
```
curl -Lo kind-config.yaml https://raw.githubusercontent.com/gireesh-nv/mongo_withargo/refs/heads/main/mongoargo_kind.yaml
```

- install kind cluster
```
kind create cluster --config kind-config.yaml --retain --image "kindest/node:v1.21.14
```

2) **STEP 2**: Installing and configuring ArgoCD
- After Kind is set up, install Argo CD:
```
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```


3) **STEP 3**: Run the following command to change ArgoCD service to match your Kind port mappings:
```
kubectl patch svc argocd-server -n argocd -p '{
  "spec": {
    "type": "NodePort",
    "ports": [
      {
        "name": "http",
        "port": 80,
        "targetPort": 8080,
        "nodePort": 30090
      },
      {
        "name": "https",
        "port": 443,
        "targetPort": 8080,
        "nodePort": 30091
      }
    ]
  }
}'
```

#verify the change
```
kubectl get svc argocd-server -n argocd
```

4) **STEP 4**: Now, access ArgoCD UI using your EC2 public IP:
```
  • HTTP: `http://<EC2-PUBLIC-IP>:8082`
  • HTTPS: `https://<EC2-PUBLIC-IP>:8444`
  ```

- Retrieve argocd admin password
```
kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 --decode
```

5) **STEP 5**: Install mongodb with HELM
```
helm repo add mongodb https://mongodb.github.io/helm-charts;
helm install enterprise-operator mongodb/enterprise-operator --namespace mongodb --create-namespace;
kubectl describe deployments mongodb-enterprise-operator -n mongodb;
```

6) **Step6**: 
Copy your GIT repository URL (make sure the Git repo is well organised)
```
https://github.com/gireesh-nv/mongo_withargo.git
```

### In the above Repo there are two directories OpsManager (owning opsmanager yaml file) and Replicasets (owning replicaset yaml file)

**Note**: The Opsmanager needs a pre created secret file called `organization-secret`. Create it on the kubernetes cluster
```
kubectl create secret generic ops-manager-admin-secret \
  --from-literal=Username="admin@test.com" \
  --from-literal=Password="Adminn@123" \
  --from-literal=FirstName="admin" \
  --from-literal=LastName="admin"
```

7) **Step7**: 
- Create Opsmanager application 
```
argocd app create mongodb-opsmanager \
  --repo https://github.com/gireesh-nv/mongo_withargo.git \
  --path opsmanager \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace mongodb \
  --sync-policy automated
```
This will ask for username and password. username is admin and password is extracted in step4. 

- Create a MongoDB replicaset (make sure you have already applied configmap.yaml and secret yaml)
```
  argocd app create mongodb-replicaset \
  --repo https://github.com/gireesh-nv/mongo_withargo.git \
  --path replicasets \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace mongodb \
  --sync-policy automated
```


### Argo useful command lines
  1) Modify same application to use different repo path,
```
    argocd app set mongodb-opsmanager --path replicasets
```
2) Get the current admin password
```
kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 --decode
```
3) Rotate the admin password,
```
kubectl patch secret argocd-secret -n argocd -p '{"data": {"admin.password": null, "admin.passwordMtime": null}}'       // sets current password to null
kubectl delete pods -n argocd -l app.kubernetes.io/name=argocd-server      // restart the argocd pod
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d      // new password
```
4) Change a user password
```
argocd account update-password --account "newuser" --current-password "oldpassword" --new-password "newsecurepassword" 
```  




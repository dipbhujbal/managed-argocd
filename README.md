# Managed ArgoCD
This repo will let you run self managed ArgoCD

## Setup
### Prerequisites
- [Kustomize](https://kustomize.io/)
- [Bitnami SealedSecrets Controller](https://github.com/bitnami-labs/sealed-secrets) and Kubeseal utility

### Public Repo Deployment
You can use this repo to run self managed ArgoCD from public repo, follow the below steps for set things up.
1. Fork this repo and and change the visibility to public.
2. Replace the repo url with your forked repo url in below files.
> - argo-cd/repo-secrets/managed-argocd-repo.yaml
> - self-managed-apps/apps/argocd-apps.yaml
> - self-managed-apps/apps/argocd-projects.yaml
> - self-managed-apps/apps/argocd.yaml
3. Once you change repo url in above files push the changes to your repo.
4. navigate to argo-cd folder and run the below command
```sh
kustomize build .|kubectl apply -f -
```
above command will create argocd namespace if not present already on your cluster and will install the required ArgoCD components.

5. Once all the ArgoCD components up and running grab the initial admin password by running below command and login to ArgoCD ui, in ArgoCD ui click on settings icon from left side navigation pane which will open a settings page, on settings page clieck on ```Repositories``` and check your managed argocd repo should showup there with successful connection status
```sh
#grab initial admin password
kubectl get secrets argocd-initial-admin-secret -o jsonpath='{.data.password}' -n argocd |base64 -d

#to access ArgoCD ui do the port-forward of argocd-server service and access the argocd ui on forwaded port
kubectl port-forward svc/argocd-server 80:80 -n argocd
```
6. If your argocd repo is connected successfully run the below commands to let ArgoCD manage itself.
```sh
#create a argocd app-project, this is one time manual project creation, for creating app-projects afterwards you can place your app-project defination manifest inside projects/ folder and  ArgoCD will automatically create a app-project.
kustomizi build projects/ |kubectl apply -f -

#now add apps to ArgoCD which will allow ArgoCD to manage itself and other apps which we will be deploying afterwards.
kustomize build self-managed-apps/ |kubectl apply -f -
```
upon successful execution of above command all apps will appear in ArgoCD ui including ArgoCD itself.

### Private Repo Deployment
You can use this repo to run self managed ArgoCD from private repo as well, follow the below steps for set things up.
1. Fork this repo and and change the visibility to private
2. Since private repos require authentication so we have to create a ArgoCD repo secret with required credential for the authenctication, there are many ways to authenticate to github repo. In this example we will be using ssh keys for github authentication, you can use your choice of authentication method but always make sure to use any external secrets manager for storing secrets in github repo, we will be using ```bitnami sealed secrets``` in this example.
3. Generate a ssh key pair and grab the ssh public key and add it to your github repo's [deploy keys](https://docs.github.com/en/developers/overview/managing-deploy-keys#deploy-keys), we are using deploy keys because it will provide radonly access to ArgoCD and readonly access is sufficient.
4. Create a k8s secret manifest with your private ssh key and other data fields required to authenticate private github repo in ArgoCD, you can check the declarative way of adding git repos in ArgoCD [here](https://argo-cd.readthedocs.io/en/stable/operator-manual/declarative-setup/#repositories).
5. Seal the above k8s secret manifest using the kubeseal utility which will generate the ```sealedsecret```, copy the generated ```sealedsecret``` manifest to ```argo-cd/repo-secrets``` folder with name as ```argocd-managed-repo.yaml```.
6. Replace the repo url with your forked repo url in below files.
> - self-managed-apps/apps/argocd-apps.yaml
> - self-managed-apps/apps/argocd-projects.yaml
> - self-managed-apps/apps/argocd.yaml
>  **Note**: Since we are using ssh keys for the authentication then auth protocol will also be chaged to ssh from https so you have put repo url something like this ```git@github.com:your-gihub-account/your-repo.git``` in above files.

7. Once you change repo url in above files push the changes to your repo.
8. navigate to argo-cd folder and run the below command
```sh
kustomize build .|kubectl apply -f -
```
above command will create argocd namespace if not present already on your cluster and will install the required ArgoCD components.

9. Once all the ArgoCD components up and running grab the initial admin password by running below command and login to ArgoCD ui, in ArgoCD ui click on settings icon from left side navigation pane which will open a settings page, on settings page clieck on ```Repositories``` and check your managed argocd repo should showup there with successful connection status
```sh
#grab initial admin password
kubectl get secrets argocd-initial-admin-secret -o jsonpath='{.data.password}' -n argocd |base64 -d

#to access ArgoCD ui do the port-forward of argocd-server service and access the argocd ui on forwaded port
kubectl port-forward svc/argocd-server 80:80 -n argocd
```
10. If your argocd repo is connected successfully run the below commands to let ArgoCD manage itself.
```sh
#create a argocd app-project, this is one time manual project creation, for creating app-projects afterwards you can place your app-project defination manifest inside projects/ folder and  ArgoCD will automatically create a app-project.
kustomizi build projects/ |kubectl apply -f -

#now add apps to ArgoCD which will allow ArgoCD to manage itself and other apps which we will be deploying afterwards.
kustomize build self-managed-apps/ |kubectl apply -f -
```
upon successful execution of above command all apps will appear in ArgoCD ui including ArgoCD itself.

## Post-Setup
Once you successfully completes ArgoCD self managed setup, you can manage ArgoCD operations like adding, updating, deleting ArgoCD applications, repos, clusters and other stuff from git.

### Adding Sample Guetbook ArgoCD application.
1. Create a repo secret and add this repo url https://github.com/argoproj/argocd-example-apps to secret, place the repo secret into the ```argo-cd/repo-secrets/``` folder and update the kustomization.yaml from same folder to add secret manifest file name to resources list.
2. Create a guestbook folder inside the ```argocd-apps/``` folder and create ArgoCD application manifest file like below in the guestbook folder
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: guestbook
  namespace: argocd
spec:
  destination:
    namespace: guestbook
    server: https://kubernetes.default.svc
  project: default
  source:
    path: guestbook
    repoURL: https://github.com/argoproj/argocd-example-apps.git
    targetRevision: HEAD
  syncPolicy:
    syncOptions:
       - CreateNamespace=true
    automated:
      prune: true
      selfHeal: true
```
3. eidt the kustomization.yaml file from ```argocd-apps/``` folder and add the ArgoCD application manifest file name into it like below
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- guestbook/application-manifest-file.yaml
```
4. push the changes to github
5. login to ArgoCD ui and go into ```arogcd-apps``` app and sync the app, once sync is complete it will deploy the new app with name guestbook.
6. check the sync status and health of guestbook app in ArgoCD ui as well as check the resource created by guestbook app in your k8s cluster in guestbook namespace.

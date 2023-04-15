## Introduction

On s'est tous posé au moins une fois la question de l'odre de déploiement des fichiers de déploiement dans Kubernetes, à minima lors de notre premier déploiement en dehors du package par défaut !

Il existe plusieurs stratégies pour gérer cela, je vous propose d'en étudier quelques une.

## Scenario

Nous allons illustrer la problématique avec les éléments suivants :
* Déploiement de l'application `Nginx`
* Dans un namespace dédié
* Avec un service account spécifique
* Utilisant un secret pour accéder à la registry
* Exposé par un Service de type `LoadBalancer`

## Approche impérative

Nous allons dans un premier temps explorer différentes solutions possibles avec une approche impérative.

### Pas de gestion de l'ordre

Dans ce premier exemple, j'ai créé un fichier par objet Kubernetes à deployer.
L'ordre d'application des fichiers se fait de manière lexicographique.
Si l'on cherche à appliquer les éléments de ce dossier, le déploiement échoue car le namespace n'est pas créé.

```bash
$ kubectl apply -f ./deploy/01_no-order
namespace/demo created
secret/registry-credential created
serviceaccount/demo created
service/nginx created
Error from server (NotFound): error when creating "deploy/01_no-order/deployment.yaml": namespaces "demo" not found
``` 

### Déploiement séquentiel

On peut envisage de demander l'exécution de nos fichiers les uns à la suite des autres de manière séquentielle.

```bash
$ kubectl apply -f ./deploy/01_no-order/namespace.yaml
namespace/demo created
$ kubectl apply -f ./deploy/01_no-order/secret.yaml
secret/registry-credential created
$ kubectl apply -f ./deploy/01_no-order/service-account.yaml
serviceaccount/demo created
$ kubectl apply -f ./deploy/01_no-order/deployment.yaml
deployment.apps/nginx created
$ kubectl apply -f ./deploy/01_no-order/service.yaml
service/nginx created
``` 

Il est possible d'accéder au nginx déployé à l'adresse suivante:
```bash
$ SERVICE_IP=$(kubectl get -o template service/nginx -n demo --template '{{ range (index .status.loadBalancer.ingress 0) }}{{ . }}{{ end }}')
$ curl $SERVICE_IP:8080
(...)
<title>Welcome to nginx!</title>
(...)
``` 

Si l'on a besoin de vérifier le status d'un objet, on pourrait également s'appuyer sur `wait` de kubectl entre les différentes étapes.

```bash 
$ kubectl wait --namespace demo \
>    --for=condition=ready pod \
>    --selector=app=nginx \
>    --timeout=30s
pod/nginx-6885d874fd-jc8j5 condition met
``` 

Le problème ici, c'est qu'à chaque fois qu'on va ajouter/supprimer un fichier il faudra maintenir ce script de déploiement!

### Tout dans le même fichier

On peut régler notre problème, en posant tout dans le même fichier en séparant les objets par `---`
Les objets sont alors instanciés dans l'ordre déclaré dans le fichier.

```bash
$ kubectl apply -f ./deploy/02_one-file/deploy.yaml
namespace/demo created
secret/registry-credential created
serviceaccount/demo created
deployment.apps/nginx created
service/nginx created
``` 
C'est une question de goût, mais pour le coup je trouve qu'avoir des fichier dédiés est plus pratique à l'usage!

### Utilisation de Kustomize

Si l'on souhaite conserver nos fichiers séparés, il est possible de s'appuyer sur [Kustomize](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/).
A noter qu'ici j'utilise Kustomize uniquement pour illustrer la gestion de l'odre, il y aurait bien mieux à faire pour bénéficier de Kustomize. Si vous ne l'avez pas encore fait, je vous invite fortement à y jetter un coup d'oeil !

J'ai volontairement mis l'ordre de fichiers dans le désordre dans le fichier `kustomization.yaml`.

Dans un premier temps, regardons ce que génère Kustomize :

```bash 
$ kubectl kustomize ./deploy/03_kustomize/
apiVersion: v1
kind: Namespace
metadata:
  name: demo
(...)
``` 

On constate que par défaut Kustomize va gérer l'ordre pour nous.
Il y a un long thread à ce sujet [ici](https://github.com/kubernetes-sigs/kustomize/issues/3794).

Si l'on souhaite gérer nous même l'ordre, il est possible de le préciser à Kustomize :

```bash 
$ kubectl kustomize ./deploy/03_kustomize/ --reorder none
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  namespace: demo
(...)
``` 

Déployons maintenant nos composants :

```bash
$ kubectl apply -k ./deploy/03_kustomize/
namespace/demo created
serviceaccount/demo created
secret/registry-credential created
service/nginx created
deployment.apps/nginx created
```

Il est possible de combiner cette approche Kustomize avec l'approche [séquentielle](#déploiement-séquentiel) présenter précédemment.
Cela permet de gérer `manuellement` des relations de dépendances entre nos différents composants et systèmes.

## Approche GitOps

Nous venons de voir l'approche impérative, étudions maintenant comment gérer cela avec une Approche GitOps.
Nous allons voir que nous avons à notre disposition le même type de stratégies !

Dans la suite de l'article, nous utiliserons [Flux CD](https://fluxcd.io/) comme outil de déploiement.

### Pas de gestion de l'ordre


## Introduction

Nous nous sommes tous posés au moins une fois la question de l'ordre de déploiement des composants dans Kubernetes, à minima lors de notre premier déploiement en dehors du package par défaut !

Il existe plusieurs stratégies pour gérer cela, je vous propose d'en étudier quelques une.

## Scenario

Nous allons illustrer la problématique avec les éléments suivants :
* Déploiement de l'application `Nginx`
* Dans un namespace dédié
* Avec un service account spécifique
* Utilisant un secret pour accéder à la registry
* Exposé par un Service de type `LoadBalancer`

## Pas de gestion de l'ordre

Dans ce premier exemple, j'ai créé un fichier par objet Kubernetes à deployer.
L'ordre d'application des fichiers se fait de manière lexicographique.
Si l'on cherche à appliquer les éléments de ce dossier, le déploiement échoue car le namespace cible du déploiement n'est pas créé.

```bash
$ kubectl apply -f ./deploy/01_no-order
namespace/demo created
secret/registry-credential created
serviceaccount/demo created
service/nginx created
Error from server (NotFound): error when creating "deploy/01_no-order/deployment.yaml": namespaces "demo" not found
``` 

## Déploiement séquentiel

On peut envisager de demander l'exécution de nos fichiers les uns à la suite des autres de manière séquentielle.

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

Nous pouvons vérifier que cette fois-ci, notre nginx est bien déployé :
```bash
$ SERVICE_IP=$(kubectl get -o template service/nginx -n demo --template '{{ range (index .status.loadBalancer.ingress 0) }}{{ . }}{{ end }}')
$ curl $SERVICE_IP:8080
(...)
<title>Welcome to nginx!</title>
(...)
``` 
Si l'on avait besoin d'attendre le déploiement effectif d'un composant, on pourrait également s'appuyer sur `wait` de kubectl entre les différentes étapes :

```bash 
$ kubectl wait --namespace demo \
    --for=condition=ready pod \
    --selector=app=nginx \
    --timeout=30s
pod/nginx-6885d874fd-jc8j5 condition met
``` 

Une limite ici, c'est qu'à chaque fois que l'on va ajouter/supprimer un fichier il faudra maintenir ce script de déploiement ! On va essayer de faire mieux que cela.

## Tout dans un même fichier

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

C'est une question de goût, mais pour le coup je trouve qu'avoir des fichiers dédiés est plus pratique à l'usage !

## Utilisation de Kustomize

Si l'on souhaite conserver nos fichiers séparés, il est possible de s'appuyer sur [Kustomize](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/). 
À noter qu'ici j'utilise Kustomize uniquement pour illustrer la gestion de l'ordre, il y aurait bien mieux à faire pour bénéficier de Kustomize. Si vous ne l'avez pas encore fait, je vous invite fortement à y jeter un coup d'œil, d'autant qu'il est intégré directement dans avec votre kubectl.

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
Cela permet de gérer manuellement des relations de dépendances entre nos différents composants et systèmes.

## Approche GitOps

Nous venons de voir différentes approches qui nous permmettent dé gérer cet ordre de manière impérative.
La philosophie GitOps est dans l'air du temps, alors voyons comment nous pouvons nous y prendre avec cette philosophie.
Nous allons voir que nous avons à notre disposition le même type de stratégies !

Dans la suite de l'article, nous utiliserons [Flux CD](https://fluxcd.io/) comme outil de déploiement. 
Ce document ne présente pas l'installation et la configuration de Flux, je vous invite à regarder la documentation officielle pour cela.

Déclaration du repository Git auprès de Flux :

```bash
$ kubectl apply -f ./deploy/04_gitops-init/git-repository.yaml
gitrepository.source.toolkit.fluxcd.io/k8s-order created

$ kubectl get gitrepository -n flux-system
NAME          URL                                          AGE   READY   STATUS
k8s-order     https://github.com/beeNotice/k8s-order       10m   True    stored artifact for revision 'main@sha1:359d1d72890122ed72891bb20d306cccbc06d61b
(...)
```

## Kustomize avec GitOps

Il est possible d'intégrer directement [Kustomize avec Flux](https://fluxcd.io/flux/components/kustomize/kustomization/), nous repartons donc de que nous avions vu [précédemment](#utilisation-de-kustomize).

```bash
$ kubectl apply -f ./deploy/05_gitops-kustomize/kustomization.yaml
kustomization.kustomize.toolkit.fluxcd.io/k8s-order created

$ flux get kustomizations
NAME            REVISION                SUSPENDED       READY   MESSAGE
k8s-order       main@sha1:359d1d72      False           True    Applied revision: main@sha1:359d1d72
(...)
```

Nous n'avons rien de particulier à réaliser, nous nous appuyons ce que nous avons vu précédemment, nous l'intégrons dans Flux, et hop le service est déployé !

## Gestion de dépendances dans un contexte GitOps

Imaginons maintenant que l'on veille gérer l'ordre de déploiement dans une approche GitOps.
Il est possible d'adresser cela en nous appuyant sur les [dependencies ](https://fluxcd.io/flux/components/kustomize/kustomization/#dependencies) de Flux. 

Pour l'illustrer, nous allons prendre un exemple qui n'a pas beaucoup de sens fonctionnel, mais là n'est pas le propos ! 

Dans ce nouvel exemple, nous déploierons la même chose que précédemment dans le namespace `after`, mais uniquement lorsque le déploiement aura été finalisé dans le namespace `demo`.

Appliquons ce nouveau déploiement :

```bash
$ kubectl apply -f ./deploy/06_gitops-dependency/kustomizations.yaml
kustomization.kustomize.toolkit.fluxcd.io/k8s-order created
kustomization.kustomize.toolkit.fluxcd.io/k8s-order-after created

$ flux get kustomizations
NAME            REVISION                SUSPENDED       READY   MESSAGE
k8s-order       main@sha1:359d1d72      False           True    Applied revision: main@sha1:359d1d72
k8s-order-after                         False           False   dependency 'flux-system/k8s-order' is not ready
(...)
```
Une fois le déploiement de nginx dans le namespace `demo` réalisé, notre nouveau déploiement est réalisé !

```bash
$ flux get kustomizations
NAME            REVISION                SUSPENDED       READY   MESSAGE
k8s-order       main@sha1:359d1d72      False           True    Applied revision: main@sha1:359d1d72
k8s-order-after main@sha1:359d1d72      False           True    Applied revision: main@sha1:359d1d72
(...)
```

## Conclusion

Nous venons de voir différentes manières de gérer l'ordre de déploiement des objets dans Kubernetes.
Je dirais que de façon générale, si on peut éviter d'avoir à le gérer manuellement, c'est bien, mais si nous n'avons pas le choix, alors nous avons différentes stratégies à notre disposition pour nous en sortir !

## Contribution

Les contributions sont toujours les bienvenues !
N'hésitez pas à ouvrir des issues ou à envoyer des PR.

N'hésitez pas à me [contacter](https://www.linkedin.com/in/martinfabien/) pour que l'on en discute ;)


## License

Copyright &copy; 2023 [VMware, Inc. or its affiliates](https://vmware.com).

This project is licensed under the [Apache Software License version 2.0](https://www.apache.org/licenses/LICENSE-2.0).

# kubernetes-security-workshop

Le but du workshop est d'apprendre comment sécuriser son cluster kubernetes par la pratique. Nous allons aborder les sujets suivant :
 - Les bonnes pratiques de sécurité des images de conteneur
 - La gestion des droits d'accès à l'API Kubernetes avec le RBAC
 - Cloisonner les composants d'un cluster Kubernetes
 - La mise à jour d'un cluster suite à une CVE
 - Détecter des comportements anormaux au run

Le workshop commencera d'abord par une présentation générale des différents concepts abordés avant de vous laisser avancer à votre rythme.

Pour pouvoir continuer à avancer même si une étape est problématique, nous vous fournissons les solutions de chacune des étapes.

### Connexion à l'espace de travail

Au début du workshop, nous vous avons donné les informations pour vous connecter. La commande à lancer pour se connecter :

```bash
ssh <user>@<host>
```

### Construire des images de conteneurs en appliquant les bonnes pratiques de sécurité

La sécurité d'un cluster Kubernetes commence par la sécurité des applications. Nous allons illustrer comment sécuriser une application vulnérable à une faille publiquement connue. L'exemple utilisé ici est une faille de rails publiée au début de 2019 (https://nvd.nist.gov/vuln/detail/CVE-2019-5418)

#### Construction et déploiement

Nous vous fournissons déjà l'application bootstrapée, le Dockerfile et les descripteurs Kubernetes pour déployer l'application.

Les commandes à lancer :

Construction de l'image :
```bash
docker image build -t eu.gcr.io/<projet>/rails-with-cve:1 .
```

Publication de l'image :
```bash
docker image push eu.gcr.io/<projet>/rails-with-cve:1
```

Déploiement de l'image :
```bash
kubectl apply -f k8s
```

#### Exploitation de la faille

Une fois l'application démarrée, vous pouvez la requêter normalement pour obtenir le README du projet :

```bash
# Pour obtenir l'ip des noeuds
kubectl get nodes -o wide
# Utiliser une external ip
curl <node-external-ip>/rails/chybeta
```

Mais avec la faille présente dans cette version de rails, nous pouvons facilement récupérer n'importe quel fichier du conteneur !

```bash
curl <node-external-ip>/rails/chybeta -H 'Accept: ../../../../../../../../../../etc/shadow{{'
```

Vous pouvez retrouver une explication de la faille : https://chybeta.github.io/2019/03/16/Analysis-for%E3%80%90CVE-2019-5418%E3%80%91File-Content-Disclosure-on-Rails/

Cette faille peut être exploitée aussi pour ouvrir un shell à distance dans le conteneur, un exemple d'exploit: https://github.com/mpgn/Rails-doubletap-RCE

Pour mitiger le problème, il y a deux étapes.

#### Changement d'utilisateur

La première étape consiste à changer l'utilisateur avec lequel s'exécute l'application pour limiter les accès fichiers possibles via l'application. 

Rajouter l'instruction USER à l'image Docker (ref: https://docs.docker.com/engine/reference/builder/#user)
```Dockerfile
USER rails
```
Modifier l'instruction COPY pour refléter ce changement de droit (ref: https://docs.docker.com/engine/reference/builder/#copy)

Pour rappel, pour créer un utilisateur sous debian, il faut exécuter la commande suivante : 
```bash
groupadd --gid 1000 rails && useradd --uid 1000 --gid rails --shell /bin/bash --create-home rails
```

Construire, publier et redéployer l'image avec le tag :
```
eu.gcr.io/<projet>/rails-with-cve:2
```

Une fois la nouvelle version de l'application déployée, le curl précédent ne fonctionne plus pour récupérer `/etc/shadow` :
```bash
curl <node-external-ip>/rails/chybeta -H 'Accept: ../../../../../../../../../../etc/shadow{{'
```

Mais on peut toujours requêter d'autres fichiers :
```bash
curl <node-external-ip>/rails/chybeta -H 'Accept: ../../../../../../../../../../demo/Gemfile{{'
```
À noter que cette pratique permet de mitiger d'éventuelles autres failles qui n'auraient pas encore de correctifs.
#### Mise à jour des dépendances

La seconde étape va être de mettre à jour la version de rails qui contient le fix de la CVE. Mettre à jour le fichier Gemfile avec une version fixée de rails.

Construire, publier et redéployer l'image avec le tag :
```
eu.gcr.io/<projet>/rails-without-cve:1
```

Une fois la nouvelle version de l'application déployée, le curl précédent ne fonctionne plus pour récupérer des fichiers du conteneur indépendamment de leur propriétaire :
```bash
curl <node-external-ip>/rails/chybeta -H 'Accept: ../../../../../../../../../../demo/Gemfile{{'
```

Une fois le test effectué supprimer le déploiement.

#### Conclusion

La sécurité se joue a plusieurs niveaux pour une application :
 - Son code
 - Ses dépendances
 - L'utilisateur avec lequel tourne l'application

Nous n'avons illustré ici que les deux derniers points, la sécurisation du code reste à faire côté développement :)

Vous pouvez retrouver d'autres pratiques pour améliorer la sécurité de vos images de conteneurs : https://res.cloudinary.com/snyk/image/upload/v1551798390/Docker_Image_Security_Best_Practices_.pdf

### 02 : Cloisonner les composants d’un cluster

Lorsqu'on utilise Kubernetes, les 
[Namespaces](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/#)
permettent d'organiser et de partager les ressources disponibles sur un cluster
kubernetes entre plusieurs équipes.
Afin que cette cohabition se passe au mieux, l'API Kubernetes expose des
ressources utilisées pour limiter et maîtriser ce que chaque équipe peut faire.

#### 02.01 : Quotas / Limits

Une première étape consiste à limiter les ressources utilisables dans un
Namespace. Cela permet de se prémunir d'une utilisation excessive de ressources
dans ce Namespace sans nuire aux autres Namespaces dans le cas d'une
manipulation accidentelle (application qui surconsomme des ressources en cas de
problème) ou d'une attaque extérieure (récupération d'un token d'accès au
cluster).

Un exemple est disponible à déployer sur votre cluster :
`kubectl apply -f 02-partition/01-quota/malicious-deployment.yml`

Ce déploiement va créer un Pod qui consommera toutes la mémoire du noeud
`node3`.
- Lancez `kubectl get nodes -w`

Au bout de quelques minutes vous verrez :
`node3   NotReady   <none>   15h   v1.15.3`

L'ensemble de la mémoire disponible sur ce noeud a été consommé et il n'est
plus disponible.

Nous allons voir comment éviter ce type de comportement en définissant des
limites de ressources disponibles pour les Pods et leurs Containers. Mais avant
tout nous allons supprimer cette application.

- `kubectl delete deployment exhauster`

Et relancer le noeud afin de le réparer (il doit repasser en Ready) :

- `gcloud compute instances reset worker-1`

Avec Kubernetes, comme avec Docker, il est possible de définir des limites
de ressources affectées aux conteneurs. Pour plus de renseignements sur ce 
mécanisme, vous pouvez consulter la docupmentation officielle [ici](https://kubernetes.io/docs/concepts/configuration/manage-compute-resources-container/).

Néanmoins, ce mécanisme seule ne suffit pas. En effet il est toujours possible
de créer des Pods sans déclarer les `limits` associées.
Les objets `LimitRange` permettent de s'assurer que dans un Namespace donné,
tous les objets créés définiront les limites de ressources tout en respectant
des valeurs minimum et maximum.

Créez une `LimitRange` afin de s'assurer que lorsqu'un Pod est créé il ne
puisse pas prendre toutes les ressources disponibles.

Recréez le Pod avec la commande ci-après, et vérifiez que cette fois ci il est
supprimé lorsqu'il occupe trop de ressources.

Malheureusement, même ainsi, il est toujours possible d'occuper toutes les
ressources du cluster en augmentant le nombre d'instances du Pod qui tournent
en même temps.
Faites le test en lançant la commande (définissez une valeur suffisament élevée
pour occuper toute la mémoire) :  
`kubectl scale --replicas=10 deploy/exhauster`

À nouveau, lancez `kubectl get nodes -w`

Et attendez quelques minutes de voir :
`node3   NotReady   <none>   15h   v1.15.3`

Nous allons voir comment empêcher la création d'un trop grand nombre de Pods.
Mais avant tout, nous allons supprimer cette application.

- `kubectl delete deployment exhauster`

Et relancer le noeud afin de le réparer (il doit repasser en Ready) :

- `gcloud compute instances reset worker-1`

En vous inspirant des exemples disponibles 
[ici](https://kubernetes.io/docs/tasks/administer-cluster/manage-resources/quota-memory-cpu-namespace/),
créez les Quotas afin d'empêcher que la multiplication des instances 
d'`exhauster` n'occupent toutes les ressources.

Tester votre solution en appliquant à nouveau le déploiement :
`kubectl apply -f 02-partition/01-quota/malicious-deployment.yml`

Et en multipliant le nombre d'instances désirées :
`kubectl scale --replicas=10 deploy/exhauster`

#### 02.02 : NetworkPolicy

### 03 : Bien exploiter le RBAC

Lors des interactions avec un cluster Kubernetes, toutes les requêtes sont
authentifiées :
- les requêtes effectuées depuis votre poste utilisent un compte associé à un certificat
  vous pouvez le consulter à l'aide de la commande suivante :
  `grep client-certificate-data .kube/config | awk '{ print $2 }' | base64 -d | openssl x509 -in - -noout -text | grep "Subject:"`
- les requêtes effectuées dans le cluster sont réalisées par des 
  [ServiceAccount](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/)
  Pour en savoir plus sur les ServiceAccount vous pouvez également consulter 
  [cette page](https://kubernetes.io/docs/reference/access-authn-authz/service-accounts-admin/)

Afin de s'assurer qu'une requête est autorisée sur le cluster, Kubernetes 
utilise le principe de Role-Based Access Control (RBAC).

Un rôle donne des droits sur les ressources de l'API (lecture, modification, ...).

L'action est autorisé si au moins un rôle donne les droits nécessaires à
l'utilisateur sur la ressource concernée.

Exemple de rôle permettant de consulter les `Namespace`:
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  # "namespace" omitted since ClusterRoles are not namespaced
  name: namespace-reader
rules:
- apiGroups: [""]
  resources: ["namespace"]
  verbs: ["get", "watch", "list"]
```

Exemple d'affectation du rôle à un utilisateur:
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: read-namespace
subjects:
- kind: User
  name: "alice@example.com"
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: namespace-reader
  apiGroup: rbac.authorization.k8s.io
```

Certaines ressources sont transverses au cluster, elles sont :
- gérées via les `ClusterRole`
- attribuées via les `ClusterRoleBinding`

Certaines ressources sont cloisonnées par Namespace, elles sont :
- gérées via les `Role`
- attribuées via les `RoleBinding` ou les `ClusterRoleBinding`

Trouvez comment le rôle `cluster-admin` est associé aux requêtes effectuées via
`kubectl` depuis la machine `shell`

TODO: Ajouter la description du montage des SA
TODO: Ajouter l'énoncé du TP - création d'un Pod dans le ns rbac et création
      des role/rolebinding nécessaires pour l'utiliser et lister les pods du ns

### 04 : PodSecurityPolicy

Lors de la première étape, nous avons vu qu'il n'était pas souhaitable de
laisser les utilisateurs lancer des conteneurs en tant qu'utilisateur `root`.

Il est possible, et nécessaire de sensibiliser les équipes à ce sujet, mais il
est également possible d'activer un mécanisme de Kubernetes permettant de
s'assurer que les conteneurs ne fonctionnent pas avec l'utilisateur `root`
ainsi que d'autres contraintes de sécurité.

Un premier mécanisme, nommé 
[SecurityContext](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/)
permet d'ajouter des informations aux Pods et Deployment afin, notamment, de
préciser l'utilisateur avec lequel le conteneur s'exécutera et autres
contraintes.

Exemple de Pod avec un `SecurityContext` :
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: security-context-demo-2
spec:
  containers:
  - name: sec-ctx-demo-2
    image: gcr.io/google-samples/node-hello:1.0
    securityContext:
      runAsUser: 2000
      allowPrivilegeEscalation: false
```

Le second mécanisme s'appuie sur un plugin de Kubernetes qui va valider et
autoriser ou interdire les requêtes de création des Pods :

- Un [Admission Controller](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/).

![Admission controller phases](images/admission-controller-phases.png)

L'admission plugin est nommé comme la ressource qu'il exploite : 
[PodSecurityPolicy](https://kubernetes.io/docs/concepts/policy/pod-security-policy/)

Vous trouverez ci-dessous un example de `PodSecurityPolicy` qui interdit :
- D'exécuter des conteneurs en mode privilégié
- De monter des répertoire de l'hôte dans les conteneurs

```yaml
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: example
spec:
  privileged: false  # Don't allow privileged pods!
  seLinux:
    rule: RunAsAny
  supplementalGroups:
    rule: RunAsAny
  runAsUser:
    rule: RunAsAny
  fsGroup:
    rule: RunAsAny
  volumes:
    - 'configMap'
    - 'emptyDir'
    - 'projected'
    - 'secret'
    - 'downwardAPI'
    - 'persistentVolumeClaim'
    # Don't allow to mount hostPath volume types and expose host dirs in 
    # containers
    # - 'hostPath' 
```

Ces plugins sont activables et désactivables au lancement de l'API Server
Kubernetes.
Malheureusement `PodSecurityPolicy` n'est pas activé par défaut et il va falloir
l'activer avant de pouvoir l'utiliser pour sécuriser votre cluster.
_Attention_ néanmoins, une fois que ce plugin est activé, il n'est plus
possible de créer de Pod sans avoir défini les droits pour autoriser
l'utilisation d'une `PodSecurityPolicy` validant la création du Pod.

Commencez par déployer une `PodSecurityPolicy` permissive et autoriser
le service account par défaut du Namespace `kube-system` à l'utiliser.
En effet, il s'agit de l'espace du cluster réservé au déploiement des
conteneurs utilisés pour l'administration du cluter.

- `kubectl apply -f 04-psp/kube-system-psp.yaml`
- `kubectl apply -f 04-psp/default-psp.yaml`

Connectez-vous en ssh au serveur qui héberge le control-plane du cluster.

- `ssh controller`

En tant qu'utilisateur `root`, éditez le fichier 
`/etc/kubernetes/manifests/kube-apiserver.yaml`

Remplacer la ligne
`- --enable-admission-plugins=NodeRestriction`
En
`- --enable-admission-plugins=NodeRestriction,PodSecurityPolicy`

Sauvegardez le fichier

L'API Server doit se relancer et cela peut nécessiter quelques minutes.
En attendant vous pouvez avoir un message du type :
```bash
ubuntu@shell:~$ kubectl get pods
The connection to the server 10.132.0.21:6443 was refused - did you specify the right host or port?
```

Une fois l'API Server redémarré, essayer de re-déployer la version vulnérable
de la première application.
Vous devriez constater qu'aucun Pod n'est créé.
Essayons de savoir pourquoi avec la commande suivante :
`kubectl describe replicaset`

Vous devriez voir un message du type :
`Warning  Failed     2m27s (x8 over 3m50s)  kubelet, node2     Error: container has runAsNonRoot and image will run as root`

En vous aidant de la documentation, modifiez le déploiement pour y ajouter un
SecurityContext à la définition des Pods afin que les conteneurs fonctionnent
avec un autre utilisateur que `root` et puissent être créés et démarrés. 

Une fois ces tests réalisés :
- Supprimez le déploiement
- Désactivez l'admission plugin `PodSecurityPolicy`

### 05 : détecter des comportements non souhaités au runtime 
 - Opa (policy, only signed images?)
 - Falco

### 06 : Comprendre l’importance des mises à jours suite à une CVE
 - Exploiter une cve fixée (maj cluster)

jak w nazwie repo - omawiamy tu próbę realizacji BuildShipRun dla tandemu argocd+helm 

BSR to w tym wypadku i w tym kontekście :

- rozdział artefaktu generycznego od konfiguracji
- artefakt generyczny ma być pozbawiony konfiguracji
- konfiguracja per środowisko (prod, test, dev) ma być dostarczana wraz z release artefaktu. 




Przedstawione jest 5 podejść do proby realizacji tego via argocd zintegrowane z helm-charts:

- pierwsze rozwiązanie - 1 repo z helm-chart a w nim 3 x values
- drugie rozwiązanie - 3 x repo z helm-chart
- trzecie rozwiązanie - umbrella chart
- czwarte rozwiązanie - zmienne nadpisywane przez argo-aplikacje
- piate rozwiązanie - 2 osobne repozytoria - jedno na helm-chart drugie na konfig  


  
<br>  
<br>  
<br>  
<br>  
<br>  
<br>  
  
  
  


Zaczynamy 


Oto proste repozytorium helma typu all-in-one - zawiera helm-chart + values:
https://github.com/slawekgh/argo-helm/tree/main/test-chart


sam helm-chart to:
- k8s-deployment , k8s-svc i k8s-cm zrobione w formie templates 
- jest też tester
- z racji tego że to all-in-one jest również values.yaml 


```
├── Chart.yaml
├── templates
│   ├── configmap.yaml
│   ├── deployment.yaml
│   ├── NOTES.txt
│   ├── service.yaml
│   └── tests
│       └── test-connection.yaml
└── values.yaml
```


Zadanie polega na podziale frameworku opartego o tandem argocd+helm na środowiska DEV, TEST i PROD i zrobienie emulacji BuildShipRun na tymże tandemie (czyli jeden helm-chart ale wdrażany wielokrotnie , za każdym razem inaczej i bazujący za każdym razem na innych wartościach w helmowym values.yaml) 

Trzeba zatem ten sam helm-chart wdrożyć oddzielnie dla DEV, dla TEST i dla PROD 

Na potrzeby tego LAB będziemy używać jednego GKE i zrobimy podział na Namespaces dev, test i prod - z grubsza odpowiada to normalnemu podziałowi na 3 osobne GKE 

Tzw "różnice między środowiskami DEV, TEST i PROD" będą zaemulowane innymi portami k8s-svc i inną ilością replik (odpowiednio dla DEV : k8s-svc-port=2222 i 2repliki , dla TEST 3333+3repliki i dla PROD 4444+4r) 

W rzeczywistych środowiskach różnice są inne i jest ich znacznie więcej - tzn. diff między helmowym values-dev.yaml a values-prod.yaml może liczyć kilkadziesiąt parametrów - tu dla nas nie ma to znaczenia bo nas interesuje mechanizm a nie zawartość

Jednym słowem dążymy do modelu gdzie helm-chart jest jeden ale ma 3 różne metody wdrożenia - czyli upraszczając 3 różne pliki values (odpowiednio dla DEV, dla TEST i dla PROD)


## pierwsze rozwiązanie - 1 repo z helm-chart a w nim 3 x values

pierwsze intuicyjne (ale bardzo słabe) rozwiązanie jakie przychodzi do głowy to oczywiście dodać do repo 3 x plik Values i zdefiniować 3 aplikacje argo:

rozbijamy zatem w naszym repo values.yaml na 3 różne pliki 

```
├── Chart.yaml
├── templates
│   ├── configmap.yaml
│   ├── deployment.yaml
│   ├── NOTES.txt
│   ├── service.yaml
│   └── tests
│       └── test-connection.yaml
├── values-dev.yaml
├── values-prod.yaml
└── values-test.yaml
```

te values-X.yaml różnią się ilością replik i portami dla k8s-svc:

```
$ cat values-dev.yaml | grep -E "replicaCount|servicePort|namespace"
replicaCount: 2
namespace: dev
servicePort : 2222
$ cat values-test.yaml | grep -E "replicaCount|servicePort|namespace"
replicaCount: 3
namespace: test
servicePort : 3333
$ cat values-prod.yaml | grep -E "replicaCount|servicePort|namespace"
replicaCount: 4
namespace: prod
servicePort : 4444
```

startujemy z czystym argo i dodajemy repo + 3 x argo-app:

```
argocd@argocd-server-5fff657769-fhml5:~$ argocd repo list 
TYPE  NAME  REPO  INSECURE  OCI  LFS  CREDS  STATUS  MESSAGE  PROJECT
argocd@argocd-server-5fff657769-fhml5:~$ argocd app list 
NAME  CLUSTER  NAMESPACE  PROJECT  STATUS  HEALTH  SYNCPOLICY  CONDITIONS  REPO  PATH  TARGET

argocd@argocd-server-5fff657769-fhml5:~$ argocd repo add https://github.com/slawekgh/argo-helm
Repository 'https://github.com/slawekgh/argo-helm' added

argocd@argocd-server-5fff657769-fhml5:~$ argocd app create argo-helm-dev --repo https://github.com/slawekgh/argo-helm --path test-chart --dest-namespace dev --dest-server https://kubernetes.default.svc --auto-prune --sync-policy automated --release-name test-release --values values-dev.yaml
application 'argo-helm-dev' created

argocd@argocd-server-5fff657769-fhml5:~$ argocd app create argo-helm-test --repo https://github.com/slawekgh/argo-helm --path test-chart --dest-namespace test --dest-server https://kubernetes.default.svc --auto-prune --sync-policy automated --release-name test-release --values values-test.yaml
application 'argo-helm-test' created

argocd@argocd-server-5fff657769-fhml5:~$ argocd app create argo-helm-prod --repo https://github.com/slawekgh/argo-helm --path test-chart --dest-namespace prod --dest-server https://kubernetes.default.svc --auto-prune --sync-policy automated --release-name test-release --values values-prod.yaml
application 'argo-helm-prod' created
```

po pewnym czasie wszystko wygląda na zgodne z założeniami, argo-apps się zsynchronizowały, zaś w każdym NS jest tyle podów ile miało być i k8s-svc są na tych portach na których miały być:

```
argocd@argocd-server-5fff657769-fhml5:~$ argocd app list 
NAME                   CLUSTER                         NAMESPACE  PROJECT  STATUS  HEALTH   SYNCPOLICY  CONDITIONS  REPO                                   PATH        TARGET
argocd/argo-helm-dev   https://kubernetes.default.svc  dev        default  Synced  Healthy  Auto-Prune  <none>      https://github.com/slawekgh/argo-helm  test-chart  
argocd/argo-helm-prod  https://kubernetes.default.svc  prod       default  Synced  Healthy  Auto-Prune  <none>      https://github.com/slawekgh/argo-helm  test-chart  
argocd/argo-helm-test  https://kubernetes.default.svc  test       default  Synced  Healthy  Auto-Prune  <none>      https://github.com/slawekgh/argo-helm  test-chart  

$ kk get po,svc -n dev 
NAME                                      READY   STATUS    RESTARTS   AGE
pod/test-release-deploy-7c9c7669c-dvjfc   1/1     Running   0          107s
pod/test-release-deploy-7c9c7669c-q9g9h   1/1     Running   0          107s

NAME                   TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
service/test-release   ClusterIP   10.108.8.247   <none>        2222/TCP   2m30s
$ kk get po,svc -n test
NAME                                      READY   STATUS    RESTARTS   AGE
pod/test-release-deploy-7c9c7669c-hf5nj   1/1     Running   0          2m28s
pod/test-release-deploy-7c9c7669c-rbs49   1/1     Running   0          2m28s
pod/test-release-deploy-7c9c7669c-zk948   1/1     Running   0          2m28s

NAME                   TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
service/test-release   ClusterIP   10.108.8.138   <none>        3333/TCP   2m28s
$ kk get po,svc -n prod
NAME                                      READY   STATUS    RESTARTS   AGE
pod/test-release-deploy-7c9c7669c-2vdzr   1/1     Running   0          2m26s
pod/test-release-deploy-7c9c7669c-n2kkd   1/1     Running   0          2m26s
pod/test-release-deploy-7c9c7669c-qt7bp   1/1     Running   0          2m26s
pod/test-release-deploy-7c9c7669c-zfhlb   1/1     Running   0          2m26s

NAME                   TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)    AGE
service/test-release   ClusterIP   10.108.2.95   <none>        4444/TCP   2m27s
```


rozwiązanie to mimo że proste i działa od ręki to jednak jest skrajnie niedoskonałe - po pierwsze zakłada że repo z helm-chart należy do nas i możemy sobie do niego wrzucać swoje pliki (a tak może nie być, może go utrzymywać zupełnie inna osoba/grupa/firma/community) , po drugie burzy ideę BuildShipRun gdzie artefakt ma być generyczny i pozbawiony konfiguracji która to konfiguracja jest różna dla prod/test/dev oraz dogrywana podczas release na dev czy na prod, po trzecie uniemożliwia jakąkolwiek separację danych między PROD a resztą niższych środowisk , może np. dojść do sytuacji że developer robiący jakieś swoje poc na dev wrzuci omyłkowo coś do prod itd itp - rozwiązanie to zatem kompletnie odpada 



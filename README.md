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
- piate rozwiązanie - 2 osobne repozytoria - jedno na helm-chart drugie na konfig\\\\\\ 


Zaczynamy 


Oto proste repozytorium helma typu all-in-one - zawiera helm-chart + values:
https://github.com/slawekgh/argo-helm/tree/main/test-chart


sam helm-chart to:
- k8s-deployment , k8s-svc i k8s-cm zrobione w formie templates 
- jest też tester
- z racji tego że to all-in-one również values.yaml 


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
tzw "różnice między środowiskami Dev, TEST i PROD" będą zaemulowane innymi portami k8s-svc i inną ilością replik (odpowiednio dla DEV : k8s-svc-port=2222 i 2repliki , dla TEST 3333+3repliki i dla PROD 4444+4r) 

w rzeczywistych środowiskach różnice są inne i jest ich znacznie więcej - tzn. diff między helmowym values-dev.yaml a values-prod.yaml może liczyć kilkadziesiąt parametrów - tu dla nas nie ma to znaczenia bo nas interesuje mechanizm a nie zawartość

Jednym słowem dążymy do modelu gdzie helm-chart jest jeden ale ma 3 różne metody wdrożenia - czyli upraszczając 3 różne pliki values (odpowiednio dla DEV, dla TEST i dla PROD)


#pierwsze rozwiązanie - 1 repo z helm-chart a w nim 3 x values
pierwsze intuicyjne (ale bardzo słabe) rozwiązanie jakie przychodzi do głowy to oczywiście dodać do repo 3 x plik Values i zdefiniować 3 aplikacje argo:

# raz

## dwa 

### trzy

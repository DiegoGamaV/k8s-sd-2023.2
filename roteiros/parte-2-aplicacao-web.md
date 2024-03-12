# Parte 2 - Implantando Aplicação Web com Banco de Dados

Essa segunda parte, como diz o título, foca em como implantar uma aplicação web e seu banco de dados. Mais especificamente, uma aplicação Node.js acompanhada de um MongoDB. Esse exemplo vai utilizar a imagem `adnanrahic/boilerplate-api` para uma aplicação Node.js *boilerplate* (ou a clássica "aplicação bestinha"). É usada pois é o suficiente para ilustrar o exemplo, e também é usada no [post](https://blog.sourcerer.io/a-kubernetes-quick-start-for-people-who-know-just-enough-about-docker-to-get-by-71c5933b4633) no qual essa parte foi inspirado. Ela é [código aberto](https://github.com/adnanrahic/boilerplate-api), caso queira dar uma olhada.

## Subindo as aplicações

Muito bem, vamos mostrar as coisas importantes! Assim como na [parte anterior](./parte-1-comandos-basicos.md), nós vamos utilizar de Deployments. Por que? Eles oferecem o poder de controle de réplicas para o Kubernetes que nos é interessante, mas não oferece identidades únicas persistentes para as réplicas, o que não tem problema pois não é importante para servidores web (considerando APIs REST).

Podemos criar da mesma forma de antes, mas que tal começarmos a dar uma olhada em **manifestos Kubernetes**? "Manifestos" são os arquivos de configuração YAML para um ou mais recursos em Kubernetes, e são chamados assim para denotar o caráter declarativo deles. No fim, a criação de todo recurso em Kubernetes produz um manifesto que fica armazenado no Control Plane, então mesmo utilizando comandos como `kubectl create deployment`, um manifesto para o Deployment é de fato criado, mas não vemos pois nos é simplificado isso. Todavia, em casos reais, precisamos de mais controle e menos simplicidade nessas configurações.

Então vamos lá. Por simplicidade, já temos pronto o arquivo `manifestos/webapp-deployment.yaml`, que é esse abaixo:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
spec:
  selector:
    matchLabels:
      app: webapp
      tier: backend
  replicas: 1
  template:
    metadata:
      labels:
        app: webapp
        tier: backend
    spec:
      containers:
        - name: webapp
          image: adnanrahic/boilerplate-api:latest
          ports:
            - containerPort: 3000
```

Olha que interessante. Nele podemos ver que há diversas configurações. Há o `kind`, que se refere a que tipo de recurso é esse, e também temos o nome do Deployment em `metadata.name`. Em `spec.template` há as informações que devem ser aplicadas pra cada uma das `replicas`, o que inclui os containers que devem ser executados no Pod, portas que devem ser abertas, nome dos containers e etc. 

Interessantemente há também uma seção de `selector.matchLabels`. Essa parte é útil pois há recursos em Kubernetes que se conectam a outros baseado em casamento de padrões. Isto é, recursos que façam match com os labels especificados podem se conectar. A natureza dessa conexão varia de acordo com os recursos, mas veremos como Services se conectam mais para frente.

Beleza, então vamos mandar o Kubernetes aplicar esse manifesto! Para isso, basta utilizar o comando abaixo:
```bash
kubectl apply -f manifestos/webapp-deployment.yaml
```

Dando uma olhada nos Pods, veremos que a aplicação está subindo. Se demorar um pouco, será porque o Kubernetes ainda está fazendo pull da imagem Docker que executará nos containers.
```bash
kubectl get pods
```
```
NAME                      READY   STATUS    RESTARTS   AGE
webapp-6779fcf474-ljnxj   1/1     Running   0          30s
```

Beleza, legal. Parece tudo certo. Mas que tal darmos uma olhada nos logs, só por precaução?
```bash
kubectl logs -f <pod-name>
```
```

> boilerplate-api@0.1.0 prod /usr/src/app
> node app.js

2024-03-12T12:17:23.435Z info: Server running on port 3000
(node:24) UnhandledPromiseRejectionWarning: MongoNetworkError: failed to connect to server [mongo:27017] on first connect [MongoNetworkError: getaddrinfo EAI_AGAIN mongo mongo:27017]
    at Pool.<anonymous> (/usr/src/app/node_modules/mongodb-core/lib/topologies/server.js:564:11)
    at Pool.emit (events.js:182:13)
    at Connection.<anonymous> (/usr/src/app/node_modules/mongodb-core/lib/connection/pool.js:317:12)
    at Object.onceWrapper (events.js:273:13)
    at Connection.emit (events.js:182:13)
    at Socket.<anonymous> (/usr/src/app/node_modules/mongodb-core/lib/connection/connection.js:246:50)
    at Object.onceWrapper (events.js:273:13)
    at Socket.emit (events.js:182:13)
    at emitErrorNT (internal/streams/destroy.js:82:8)
    at emitErrorAndCloseNT (internal/streams/destroy.js:50:3)
    at process._tickCallback (internal/process/next_tick.js:63:19)
(node:24) UnhandledPromiseRejectionWarning: Unhandled promise rejection. This error originated either by throwing inside of an async function without a catch block, or by rejecting a promise which was not handled with .catch(). (rejection id: 1)
(node:24) [DEP0018] DeprecationWarning: Unhandled promise rejections are deprecated. In the future, promise rejections that are not handled will terminate the Node.js process with a non-zero exit code.
```

Opa! Não está tudo certo não. Na verdade, parece um erro específico de Node que aponta que a aplicação não está conseguindo se conectar com o MongoDB. Mais especificamente, que não encontra nenhuma instância de MongoDB com o hostname `mongo` na porta `27017`. E parando para pensar isso faz total sentido. Afinal, não subimos nenhum MongoDB.

Então temos um exemplo claro de dependência aqui. Essa aplicação não funcionará corretamente sem satisfazer essa dependência. Em coisas como Docker Compose é possível expressar dependências, de maneira que uma aplicação não sobe se não for possível subir suas dependências. Porém, até o momento, Kubernetes não tem nenhum equivalente. Nesse caso, precisamos subir o MongoDB. Poderíamos apenas adicionar outro container no Pod, já que Pods aceitam vários containers. Mas já que o MongoDB é semanticamente um serviço a parte, é uma melhor prática fazer com que o Kubernetes o trate como tal. Então criaremos um Deployment para ele.

Também já temos esse manifesto pronto:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mongodb
spec:
  selector:
    matchLabels:
      app: mongodb
      tier: backend
  replicas: 1
  template:
    metadata:
      labels:
        app: mongodb
        tier: backend
    spec:
      containers:
        - name: mongo
          image: mongo:4.4
          ports:
            - containerPort: 27017
```

Esse manifesto expressa que tem que haver um Deployment, com 1 réplica, e que réplicas precisam de um container executando a imagem do MongoDB na versão 4.4. Aqui, diferentemente da aplicação web, está sendo exposta a porta 27012. Vamos executar:

```bash
kubectl apply -f manifestos/db-deployment.yaml
```

E vejamos agora nossos Pods. Após alguns instantes deve haver ambos disponíveis:
```bash
kubectl get pods
```
```
NAME                      READY   STATUS    RESTARTS   AGE
mongodb-d496f5c99-4sjpk   1/1     Running   0          44s
webapp-6779fcf474-ljnxj   1/1     Running   0          25m
```

E os logs do Pod do MongoDB? Será que está tudo certo? Vou deixar você olhar essa, até por que os logs são muito grandes para mostrar aqui no roteiro! Mas, como poderá ver, está sim tudo certo.

Mas e se revisitarmos os logs de `webapp`? Será que algo mudou? Dê uma olhadinha.

E não, infelizmente nada mudou. Esse é um exemplo de aplicação que já quebrou, e não está mais buscando pelo MongoDB, e portanto não vai mais funcionar. O que deveria ter acontecido é que se má configuração (como a ausência de MongoDB, por exemplo) quebra o serviço, ele deveria de fato morrer. Todavia, o erro está apenas sendo impresso nos logs, a aplicação não está falhando, e isso vai de contra a lógica de Fail Fast. 

Vamos fazer um favorzinho ao webapp e deletá-lo! Assim o Kubernetes criará um novo Pod, e esse enxergará o MongoDB:
```bash
kubectl delete pod <pod-name>

kubectl get pods
```
```
NAME                      READY   STATUS    RESTARTS   AGE
mongodb-d496f5c99-4sjpk   1/1     Running   0          7m1s
webapp-6779fcf474-fq8ch   1/1     Running   0          4s
```

Okay, okay. Até agora tudo bem. Vamos olhar os logs de novo
```bash
kubectl logs -f <pod-name>
```
```

> boilerplate-api@0.1.0 prod /usr/src/app
> node app.js

2024-03-12T12:17:23.435Z info: Server running on port 3000
(node:24) UnhandledPromiseRejectionWarning: MongoNetworkError: failed to connect to server [mongo:27017] on first connect [MongoNetworkError: getaddrinfo EAI_AGAIN mongo mongo:27017]
    at Pool.<anonymous> (/usr/src/app/node_modules/mongodb-core/lib/topologies/server.js:564:11)
    at Pool.emit (events.js:182:13)
    at Connection.<anonymous> (/usr/src/app/node_modules/mongodb-core/lib/connection/pool.js:317:12)
    at Object.onceWrapper (events.js:273:13)
    at Connection.emit (events.js:182:13)
    at Socket.<anonymous> (/usr/src/app/node_modules/mongodb-core/lib/connection/connection.js:246:50)
    at Object.onceWrapper (events.js:273:13)
    at Socket.emit (events.js:182:13)
    at emitErrorNT (internal/streams/destroy.js:82:8)
    at emitErrorAndCloseNT (internal/streams/destroy.js:50:3)
    at process._tickCallback (internal/process/next_tick.js:63:19)
(node:24) UnhandledPromiseRejectionWarning: Unhandled promise rejection. This error originated either by throwing inside of an async function without a catch block, or by rejecting a promise which was not handled with .catch(). (rejection id: 1)
(node:24) [DEP0018] DeprecationWarning: Unhandled promise rejections are deprecated. In the future, promise rejections that are not handled will terminate the Node.js process with a non-zero exit code.
```

Ué. O mesmo erro? Mas por que será, se agora há um MongoDB? E ainda mais, o nome do container executando dentro dos Pods do Deployment se chama exatamente `mongo`. O que será? 

Vamos falar sobre serviços agora.

## Expondo aplicações

Em Kubernetes, Pods são completamente isolados a não ser que dito o contrário. Dentro de um mesmo Pod, containers podem se comunicar entre si através de portas expostas ou volumes configurados (já que todos os containers executam em um mesmo Pod, que é executado em um nó). Pods também podem conversar entre si, desde que façam parte de um mesmo controle (isto é, Deployments, Daemonsets... e etc). Todavia, essas brechas propositais no isolamento não permitem Pods de controles diferentes conversarem entre si.

O que está acontecendo é que Pods do Deployment `webapp` estão tentando se comunicar com Pods do Deployment `mongodb`. Mais especificamente, esperando que algum Pod do Deployment `mongodb` possua a porta 27017 exposta e atenda pelo nome **mongo**. E isso não vai acontecer por padrão. Para quebrar o isolamento natural de Pods, existem os **Services**.

Services são outro tipo de recurso Kubernetes que envolvem Pods em outro isolamento, fazendo com que os Pods contidos nele sejam acessíveis através de determinadas portas. Pense nisso como uma espécie de proxy: você passa a falar com o Service, e o Service intermedia você e os Pods. Isso faz com que seja possível a comunicação de aplicações, tanto internamente quanto externamente ao cluster. Afinal, se Pods não são totalmente livres dentro do cluster, eles também não são acessíveis de fora do cluster. No geral, Services são bastante flexíveis e poderosos, e são essenciais para aplicações em Kubernetes que falam entre si, ou precisam que o mundo fale com elas.

Vamos aplicar o seguinte manifesto:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: mongo
  labels:
    app: mongo
    tier: backend
spec:
  ports:
    - port: 27017
      targetPort: 27017
  selector:
    app: mongodb
    tier: backend

```

Esse manifesto é do tipo Service, e possui metadados similares aos Deployments, porém seu `spec` não detalha Pods, e sim regras de rede e de seletores. Lembra quando mencionamos que há recursos que fazem casamento de padrões? Service é um deles. O mapa `spec.seletor` diz os labels que um Pod precisa ter para que o Service o inclua no roteamento. Se um Pod não possuir esses dois seletores, com esses dois exatos valores, ele não fará parte do Service. Além disso, aqui a lista `spec.ports` declara um mapeamento de portas. `spec.ports[0].port` se refere à porta que o Service irá expor, e `spec.ports[0].targetPort` se refere à porta dos Pods para a qual o Service enviará o tráfego. Ou seja, se containers executando nos Pods não expuserem a porta de mesmo número que `spec.ports[0].targetPort`, o tráfego de rede não alcançará eles. Por sorte, o container do MongoDB expõe sim a mesma porta.

Então vamos aplicar:
```yaml
kubectl apply -f manifestos/db-service.yaml
```

Vamos dar uma olhada nesse serviço:
```bash
kubectl get services
```
```
NAME         TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)     AGE
kubernetes   ClusterIP   10.96.0.1      <none>        443/TCP     42h
mongo        ClusterIP   10.98.249.80   <none>        27017/TCP   5m32s
```

Aqui podemos ver que há dois Services disponíveis. O padrão `kubernetes`, para funcionamento interno do cluster, e o `mongodb` que criamos. Veja que há um tipo para eles. O tipo de ambos é ClusterIP, o que significa que os Services são acessíveis a partir de um IP Privado do Cluster, e portanto acessível apenas dentro do próprio cluster. Você pode ver também que isso é corroborado pelo fato de EXTERNAL-IP estar vazio. Isso é perfeito para um banco de dados, para evitar que aplicações ou pessoas não autorizadas acessem o banco. Todavia, isso não é adequado para a aplicação web, que inclusive nem chegamos a olhar! Mas olharemos em breve.

Vamos dar uma olhada breve no `describe` desse Service:
```bash
kubectl describe service mongodb
```
```
Name:              mongo
Namespace:         default
Labels:            app=mongo
                   tier=backend
Annotations:       <none>
Selector:          app=mongodb,tier=backend
Type:              ClusterIP
IP Family Policy:  SingleStack
IP Families:       IPv4
IP:                10.98.249.80
IPs:               10.98.249.80
Port:              <unset>  27017/TCP
TargetPort:        27017/TCP
Endpoints:         10.244.0.12:27017
Session Affinity:  None
Events:            <none>
```

Aqui há informações bem mais digeríveis do que as do Deployment e de nós, e são informações de rede. A talvez mais importante a ser observar é a lista de `Endpoints`, que mostra todos os Pods conectados ao Service, e a porta que está conectada. Esse IP privado é exatamente o IP do Pod do MongoDB. Logo, agora nosso Pod `webapp` conseguirá conversar com o Pod `mongodb`! 

E antes que se pergunte "*mas como ele vai conseguir, se ele está tentando falar com `mongo` e na verdade endpoint é `10.244.0.12:27017`?*. Lembre-se: os Pods continuam isolados, e portanto não falaremos diretamente com o Pod, e sim com o Service! Assim, precisaremos falar com `<service-name>:27017`. E não por acaso, o Service foi nomeado como `mongo`, para que `webapp` consiga se comunicar com `mongo:27107`, e o nosso Service roteie isso para o Pod `mongodb`.

Vamos reiniciar a aplicação web e ver isso na prática. Em comandos consecutivos:

```bash
kubectl get pods # para ver a lista de pods

kubectl delete pod <pod-name-antigo> # para deletar webapp-

kubectl get pods # para ver o nome do novo pod

kubectl logs -f <pod-name-novo> # para ver os logs
```
```

> boilerplate-api@0.1.0 prod /usr/src/app
> node app.js

2024-03-12T13:21:01.539Z info: Server running on port 3000
```

E veja só! a aplicação agora não apresenta mais o erro. Seria ótimo de fato enxergar a aplicação web *pela web*, não é mesmo? Para expor ela também, precisamos utilizar algum tipo diferente do tipo padrão de Service, que é ClusterIP. Os tipos alternativos são **NodePort** e **LoadBalancer**.

NodePort é a maneira mais básica de expor aplicações para fora do cluster, porém não é tão adequada para produção e portanto mais usada em desenvolvimento. Um Service do tipo NodePort aloca uma determinada porta de **todos os nós** do cluster, e escuta por tráfego nessa porta. Ou seja, se você possui um cluster com 5 nós, e tem um Service NodePort na porta alta de 30123, isso significa que em todos os nós a porta 30123 será alocada, e o Service estará escutando nessa porta, em *cada um dos nós*. Essa porta pode ser especificada, ou Kubernetes pode decidir qual porta usar dinamicamente. Visto que expõe-se uma porta do nó, desde que algum dos nós seja visível para o mundo, o Service fica acessível em `<NODE_IP>:<NODE_PORT>`.

Uma alternativa a isso é o LoadBalancer, que possui um IP público próprio. Mas como? Isso porque Kubernetes é geralmente utilizado em grandes provedores de nuvem. Quando se cria um Service do tipo LoadBalancer, o Kubernetes pede que o provedor de nuvem crie um balanceador de carga dele, diga o IP público daquele balanceador de carga, e usa esse IP para o Service. Nesse caso, não é necessário expor portas altas de nó, e consequentemente o Service fica exposto em `<LOAD_BALANCER_IP>:<PORT>`. Isso pode, obviamente, custar mais caro, porém também é uma das maneiras mais viáveis de ter um IP público confiável (e consequentemente um domínio que aponte para esse IP público). Como isso exige apoio da nuvem, não é usado em ambientes de desenvolvimento ou em Kubernetes bare metal.

Como estamos, todavia, fazendo a simulação em Minikube, ele por sorte simplifica o uso de LoadBalancer! Então utilizaremos ele para exemplificar:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: webapp
  labels:
    app: webapp
    tier: backend
spec:
  type: LoadBalancer
  ports:
    - port: 80
      targetPort: 3000
  selector:
    app: webapp
    tier: backend
```

Aplique o Service:
```bash
kubectl apply -f manifestos/webapp-service.yaml
```

Já que o Minikube é um ambiente simulado, ele exige que seja aberto um túnel para simular a presença de um balanceador de carga. Abra um novo terminal, execute o comando abaixo e deixe o terminal aberto:
```bash
minikube tunnel
```

Volte ao seu terminal principal e dê uma olhada nos serviços
```bash
kubectl get services
```
```
NAME         TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
kubernetes   ClusterIP      10.96.0.1       <none>        443/TCP        42h
mongo        ClusterIP      10.96.16.87     <none>        27017/TCP      25m
webapp       LoadBalancer   10.108.191.57   127.0.0.1     80:31817/TCP   4s
```

Veja que o tipo é de fato LoadBalancer, e diferentemente dos outros, ele possui sim um IP Externo: localhost. Isso se dá graças ao Minikube, que está usando a máquina local como um balanceador de carga. Mais especificamente, Minikube está usando a porta 31817 da máquina local para isso. Vamos finalmente olhar nossa aplicação: acesse http://127.0.0.1:80/api.

O resultado esperado é:
```
Api Works.
```

Vamos fazer ela usar o banco de dados agora! Faça uma requisição para registrar um usuário. Aqui está um exemplo com cURL:
```bash
curl --location 'http://127.0.0.1:80/api/auth/register' \
--header 'Content-Type: application/json' \
--data-raw '{
    "email": "diegogama@lsd.ufcg.edu.br",
    "name": "Diego Gama",
    "password": "supersecret123"
}'
```

O retorno esperado é algo assim (o `token` pode ser diferente)
```json
{
    "auth": true,
    "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6IjY1ZjA1ZmNjOGFlN2VjMDAxODI2ODIxMCIsImlhdCI6MTcxMDI1MTk4MCwiZXhwIjoxNzEwMzM4MzgwfQ.1IP6ODwImzielYgHo1gkJQwek1EalQUEMPrGGpY-cgk"
}
```

Legal! Que tal se autenticar? Substitua o token abaixo pelo token que recebeu.

```bash
curl --location 'http://127.0.0.1:80/api/auth/me' \
--header 'x-access-token: eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6IjY1ZjA1ZmNjOGFlN2VjMDAxODI2ODIxMCIsImlhdCI6MTcxMDI1MTk4MCwiZXhwIjoxNzEwMzM4MzgwfQ.1IP6ODwImzielYgHo1gkJQwek1EalQUEMPrGGpY-cgk'
```

O retorno esperado é algo assim:
```json
{
    "_id": "65f05fcc8ae7ec0018268210",
    "name": "Diego Gama",
    "email": "diegogama@lsd.ufcg.edu.br",
    "__v": 0
}
```

Beleza, barece funcionar bem. Mas e se derrubarmos o banco de dados? Será que o usuário registrado se mantém? Vamos testar
```bash
kubectl get pods # listar os pods

kubectl delete pod <pod-name> # nome do pod do mongodb

kubectl get pods # veja e aguarde o novo pod estar pronto
```

Depois que o novo Pod que o substituir estiver pronto, vamos tentar nos autenticar de novo:

```bash
curl --location 'http://127.0.0.1:80/api/auth/me' \
--header 'x-access-token: eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6IjY1ZjA1ZmNjOGFlN2VjMDAxODI2ODIxMCIsImlhdCI6MTcxMDI1MTk4MCwiZXhwIjoxNzEwMzM4MzgwfQ.1IP6ODwImzielYgHo1gkJQwek1EalQUEMPrGGpY-cgk'
```
```
No user found.
```

Opa! Parece que os dados não ficaram salvos, não é? Mas isso não é aceitável. Vamos falar sobre volumes.

## Persistência

Em Kubernetes, os containers continuam com o mesmo isolamento de ambiente que possuem quando executados diretamente com Docker ou Docker Compose. Isto é, o sistema de arquivo deles é independente do sistema de arquivos do host, e consequentemente qualquer informação armazenada se perde quando o container para de executar. Assim como em Docker, Kubernetes permite o uso de volumes para mapear diretórios ou arquivos do container, para diretórios ou arquivos do host, através de Volumes.

Entretanto, Volumes em Kubernetes são bem mais complexos que Volumes em Docker, já que em Kubernetes Pods podem ser orquestrados em diversas máquinas. Por isso, há vários tipos de Volumes. Vários *mesmo*. Para uma lista extensa de volumes, dê uma olhadinha na [documentação](https://kubernetes.io/docs/concepts/storage/volumes/). Mas para lhe popuar um clique, caso não esteja muito curioso, a variedade se dá pelo fato de Kubernetes mapear tipos de volume do Kubernetes para os tipos de volume ofertados por provedores de nuvem. Então há diversos volumes da AWS, diversos da Azure e por aí vai. Nada disso é interessante para contexto de quem não usa nuvem pública, e logo não para esse roteiro.

Para nós, é interessante saber de três tipos: **emptyDir**, **hostPath** e **PersistentVolumes**. 

EmptyDir é o mais simples, pois é ironicamente um volume efêmero. Isso significa que é um volume que começa vazio e que é deletado quando o Pod morre. Mas para que isso? É bastante útil quando o objetivo é compartilhar arquivos entre containers num mesmo Pod, e quando não se quer nada persistente.

HostPath é sim persistente, pois é análogo ao volume do Docker. Ele mapeia o container diretamente para um local do nó. Isso, todavia, é bastante inseguro, visto que o ataques ao nó podem comprometer diretamente o container em caso de invasão bem sucedida. Todavia, é bastante útil para desenvolvimento e testes.

PersistentVolumes, por sua vez, são o que o nome diz: persistentes. Eles são dentre os mais complexos, mas também os mais versáteis. Eles são mais usados em produção e em nuvens públicas. Fora de ambientes com integração direta a Kubernetes, é responsabilidade do administrador provisionar os volumes antes de avisar ao Kubernetes que eles existem. Por sorte, o Minikube simula esse provisionamento por parte da nuvem, o que nos permite mostrar esse tipo de volume que é tão importante.

Precisamos especificar os requisitos do nosso volume, para que o Kubernetes o provisione. Para isso, utilizamos um PersistentVolumeClaim, um manifesto com esses requisitos.

Vamos usar esse PersistentVolumeClaim abaixo:
```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: mongo-pv-claim
spec:
  storageClassName: standard
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 3Gi
```

Ele pede 3 GB de espaço de armazenamento, e que o volume dê ao Pod permissões de leitura e escrita. Especificamos também uma **StorageClass** em `spec.storageClassName` para que o Kubernetes saiba como queremos que o volume seja. Utilizamos a `standard` pois é a ofertada pelo Minikube para provisionar volumes. Vamos aplicá-lo para que Kubernetes saiba que esses requisitos existem.

```bash
kubectl apply -f manifestos/db-pvc.yaml

kubectl get pvc
```
```
NAME             STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
mongo-pv-claim   Pending                                      manual         3s
```

Podemos ver que o PersistentVolumeClaim está pendente. Isto é, ele não foi vinculado a ninguém, e quem quer que seja que apareça primeiro para declarar que usa este Claim, se vinculará a ele.

Vamos editar agora o manifesto do nosso banco de dados para montar um volume, e expressar que o volume precisa seguir os requisitos desse claim:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mongodb
spec:
  selector:
    matchLabels:
      app: mongodb
      tier: backend
  replicas: 1
  template:
    metadata:
      labels:
        app: mongodb
        tier: backend
    spec:
      containers:
        - name: mongo
          image: mongo:4.4
          ports:
            - containerPort: 27017
          volumeMounts:
            - name: data
              mountPath: /data/db
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: mongo-pv-claim
```

Observe que adicionamos `template.spec.containers[0].volumeMounts` e `template.spec.volumes`. O `volumeMounts` diz exatamente qual diretório do container deve ser tratado como um volume, e o nome desse volume. O `volumes` diz *como* aquele volume deve ser tratado, e declara que deve ser tratado conforme especificado pelo nosso PersistentVolumeClaim. Vamos aplicar isso e ver a magia acontecer.

```bash
kubectl apply -f manifestos/db-persistent-deployment.yaml
```
```
NAME                       READY   STATUS    RESTARTS   AGE
mongodb-5b8c574cf8-j7pvk   1/1     Running   0          20s
webapp-6779fcf474-zptj5    1/1     Running   0          59m
```

Vamos ver como está o nosso PersistentVolumeClaim agora, e ver se surgiu algum PersistentVolume:
```bash
kubectl get pvc
kubectl get pv
```
```
NAME             STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
mongo-pv-claim   Bound    pvc-6f2ee187-d9b8-4c1c-90aa-91e33909ef0b   3Gi        RWO            standard       7m50s

NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                    STORAGECLASS   REASON   AGE
pvc-6f2ee187-d9b8-4c1c-90aa-91e33909ef0b   3Gi        RWO            Delete           Bound    default/mongo-pv-claim   standard                7m50s
```

Podemos ver que nosso Claim agora está com status *BOUND*, e que um PersistentVolume foi provisionado por parte do Minikube para suportar os requisitos desejados. Que tal testarmos a aplicação agora?

Criando novamente um usuário:
```bash
curl --location 'http://127.0.0.1:80/api/auth/register' \
--header 'Content-Type: application/json' \
--data-raw '{
    "email": "diegogama@lsd.ufcg.edu.br",
    "name": "Diego Gama",
    "password": "supersecret123"
}'
```

O retorno é o mesmo daquela vez:
```json
{
    "auth":true,
    "token":"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6IjY1ZjA2ZTU2OGFlN2VjMDAxODI2ODIxMSIsImlhdCI6MTcxMDI1NTcwMiwiZXhwIjoxNzEwMzQyMTAyfQ.2hmt3Qj6ezV6zj2M8uT6A-2q3GbcVsy5WjZR4P2mXzQ"
}
```

Agora tentando nos autenticar (não esquecer de ajustar o token):
```bash
curl --location 'http://127.0.0.1:80/api/auth/me' \
--header 'x-access-token: eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6IjY1ZjA2ZTU2OGFlN2VjMDAxODI2ODIxMSIsImlhdCI6MTcxMDI1NTcwMiwiZXhwIjoxNzEwMzQyMTAyfQ.2hmt3Qj6ezV6zj2M8uT6A-2q3GbcVsy5WjZR4P2mXzQ'
```
```json
{
    "_id":"65f06e568ae7ec0018268211",
    "name":"Diego Gama",
    "email":"diegogama@lsd.ufcg.edu.br",
    "__v":0
}
```

Beleza, tudo certo até aqui. Que tal reiniciarmos o banco, e ver se tudo continua certo mesmo?

```bash
kubectl get pod # para ver o nome do pod do mongodb

kubectl delete pod <pod-name: # com o nome do pod do mongodb

kubectl get pod # aguarde outro pod ser iniciado
```

Vamos tentar nos autenticar com o mesmo token provido anteriormente, o usuário deveria se manter:
```bash
curl --location 'http://127.0.0.1:80/api/auth/me' \
--header 'x-access-token: eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6IjY1ZjA2ZTU2OGFlN2VjMDAxODI2ODIxMSIsImlhdCI6MTcxMDI1NTcwMiwiZXhwIjoxNzEwMzQyMTAyfQ.2hmt3Qj6ezV6zj2M8uT6A-2q3GbcVsy5WjZR4P2mXzQ'
```
```json
{
    "_id":"65f06e568ae7ec0018268211",
    "name":"Diego Gama",
    "email":"diegogama@lsd.ufcg.edu.br",
    "__v":0
}
```

E agora sim!

## Conclusões

Nessa Parte 2 mostramos como é possível implantar uma aplicação web passo a passo, que possua como dependência um banco de dados. De pouco em pouco fomos adicionando **Services** para expor os Pods (tanto internamente quanto externamente), e **PersistentVolumes** para armazenar os dados do banco em caso de falha.

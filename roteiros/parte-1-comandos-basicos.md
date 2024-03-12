# Parte 1 - Uso Básico do Kubernetes

Essa primeira parte foca em como utilizar o `kubectl` (a CLI do Kubernetes) para enxergar e contorlar os recursos do cluster. Alguns comandos serão mencionados no roteiro, bem como a criação de pods. Se você já estiver familiarizado com `kubectl` e como Deployments funcionam, pode pular essa parte e seguir direto para a [parte 2](./parte-2-aplicacao-web.md).

## Explicação Geral

O `kubectl` é uma CLI utilitária que permite a um administrador, dentro ou fora do cluster, acessar as funcionalidades do Control Plane do Kubernetes. O `kubectl` simplifica a interação, de maneira a evitar a necessidade de interagir através da API REST contida no Control Plane. No geral, é necessário um arquivo de configurações que contenha a **Service Account** que `kubectl` irá utilizar, isto é, as credenciais para acesso ao cluster. Sem essas configurações, o `kubectl` não sabe onde nem qual é o cluster ao qual você deseja emitir comandos. Ambientes simplificados como o **Minikube** oferecem atalhos para acessar o cluster, sem a necessidade de realizar configurações penosas. No nosso caso, que utilizamos **Minikube**, existe o comando `minikube kubectl` que envolve a CLI `kubectl` de maneira a sempre fornecer a ela a configuração do ambiente que está rodando de forma automática.

Uma recomendação: execute `alias kubectl="minikube kubectl --"` antes de prosseguir, pois os comandos abaixo utilizarão apenas `kubectl` para encurtar comandos.

### Nós

Nós são os principais componentes de um cluster Kubernetes, pois um cluster Kubernetes é definido por um conjunto de nós conectados e controlados pelo Control Plane. Um nó pode ser uma máquina física, uma máquina virtual ou até mesmo um container. No caso do Minikube, há apenas um nó, chamado **minikube**, visto que é um ambiente simplificado para desenvolvimento.

Para visualizá-lo, execute o comando abaixo. 
```bash
kubectl get nodes
```

A saida esperada deve ser similar à seguinte:
```
NAME       STATUS   ROLES           AGE   VERSION
minikube   Ready    control-plane   28h   v1.28.3
```

Aqui é possível ver o nome do nó, seu status, seus papéis, sua idade e sua versão (o que pode ser interessante para garantir compatibilidade com todas as funcionalidades desejadas do Kubernetes). O status *Ready* é autoexplicativo, e mostra que o nó está pronto para ser controlado (caso não fosse ele mesmo o painel de controle).

É possível ver mais detalhes do nó utilizando `kubectl describe node <node-name>`. Com esse comando diversas informações são exibidas, desde metadados sobre o nó (como IP, data de criação e recursos de hardware), um histórico de eventos pelos quais ele passou, e até mesmo todos os pods alocados nele. Há bastante informação aqui que é mais avançada, então só dê uma olhada caso tenha curiosidade.

```bash
kubectl describe node minikube
```

A saída com as informações citadas é a seguinte:
```
Name:               minikube
Roles:              control-plane
Labels:             beta.kubernetes.io/arch=amd64
                    beta.kubernetes.io/os=linux
                    kubernetes.io/arch=amd64
                    kubernetes.io/hostname=minikube
                    kubernetes.io/os=linux
                    minikube.k8s.io/commit=8220a6eb95f0a4d75f7f2d7b14cef975f050512d
                    minikube.k8s.io/name=minikube
                    minikube.k8s.io/primary=true
                    minikube.k8s.io/updated_at=2024_03_10T15_53_11_0700
                    minikube.k8s.io/version=v1.32.0
                    node-role.kubernetes.io/control-plane=
                    node.kubernetes.io/exclude-from-external-load-balancers=
Annotations:        kubeadm.alpha.kubernetes.io/cri-socket: unix:///var/run/cri-dockerd.sock
                    node.alpha.kubernetes.io/ttl: 0
                    volumes.kubernetes.io/controller-managed-attach-detach: true
CreationTimestamp:  Sun, 10 Mar 2024 15:53:08 -0300
Taints:             <none>
Unschedulable:      false
Lease:
  HolderIdentity:  minikube
  AcquireTime:     <unset>
  RenewTime:       Mon, 11 Mar 2024 20:45:44 -0300
Conditions:
  Type             Status  LastHeartbeatTime                 LastTransitionTime                Reason                       Message
  ----             ------  -----------------                 ------------------                ------                       -------
  MemoryPressure   False   Mon, 11 Mar 2024 20:45:14 -0300   Sun, 10 Mar 2024 15:53:08 -0300   KubeletHasSufficientMemory   kubelet has sufficient memory available
  DiskPressure     False   Mon, 11 Mar 2024 20:45:14 -0300   Sun, 10 Mar 2024 15:53:08 -0300   KubeletHasNoDiskPressure     kubelet has no disk pressure
  PIDPressure      False   Mon, 11 Mar 2024 20:45:14 -0300   Sun, 10 Mar 2024 15:53:08 -0300   KubeletHasSufficientPID      kubelet has sufficient PID available
  Ready            True    Mon, 11 Mar 2024 20:45:14 -0300   Sun, 10 Mar 2024 15:53:08 -0300   KubeletReady                 kubelet is posting ready status
Addresses:
  InternalIP:  192.168.49.2
  Hostname:    minikube
Capacity:
  cpu:                12
  ephemeral-storage:  1055762868Ki
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             12119192Ki
  pods:               110
Allocatable:
  cpu:                12
  ephemeral-storage:  1055762868Ki
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             12119192Ki
  pods:               110
System Info:
  Machine ID:                 a4fbd4131a014af6beb7e193c49d27d2
  System UUID:                a4fbd4131a014af6beb7e193c49d27d2
  Boot ID:                    565d87a1-1ef4-4cc8-970e-a7c0fac9d402
  Kernel Version:             5.15.133.1-microsoft-standard-WSL2
  OS Image:                   Ubuntu 22.04.3 LTS
  Operating System:           linux
  Architecture:               amd64
  Container Runtime Version:  docker://24.0.7
  Kubelet Version:            v1.28.3
  Kube-Proxy Version:         v1.28.3
PodCIDR:                      10.244.0.0/24
PodCIDRs:                     10.244.0.0/24
Non-terminated Pods:          (7 in total)
  Namespace                   Name                                CPU Requests  CPU Limits  Memory Requests  Memory Limits  Age
  ---------                   ----                                ------------  ----------  ---------------  -------------  ---
  kube-system                 coredns-5dd5756b68-vh5bj            100m (0%)     0 (0%)      70Mi (0%)        170Mi (1%)     28h
  kube-system                 etcd-minikube                       100m (0%)     0 (0%)      100Mi (0%)       0 (0%)         28h
  kube-system                 kube-apiserver-minikube             250m (2%)     0 (0%)      0 (0%)           0 (0%)         28h
  kube-system                 kube-controller-manager-minikube    200m (1%)     0 (0%)      0 (0%)           0 (0%)         28h
  kube-system                 kube-proxy-5rgxr                    0 (0%)        0 (0%)      0 (0%)           0 (0%)         28h
  kube-system                 kube-scheduler-minikube             100m (0%)     0 (0%)      0 (0%)           0 (0%)         28h
  kube-system                 storage-provisioner                 0 (0%)        0 (0%)      0 (0%)           0 (0%)         28h
Allocated resources:
  (Total limits may be over 100 percent, i.e., overcommitted.)
  Resource           Requests    Limits
  --------           --------    ------
  cpu                750m (6%)   0 (0%)
  memory             170Mi (1%)  170Mi (1%)
  ephemeral-storage  0 (0%)      0 (0%)
  hugepages-1Gi      0 (0%)      0 (0%)
  hugepages-2Mi      0 (0%)      0 (0%)
Events:
  Type    Reason                   Age                From             Message
  ----    ------                   ----               ----             -------
  Normal  Starting                 26m                kube-proxy
  Normal  Starting                 26m                kubelet          Starting kubelet.
  Normal  NodeHasSufficientMemory  26m (x9 over 26m)  kubelet          Node minikube status is now: NodeHasSufficientMemory
  Normal  NodeHasNoDiskPressure    26m (x7 over 26m)  kubelet          Node minikube status is now: NodeHasNoDiskPressure
  Normal  NodeHasSufficientPID     26m (x7 over 26m)  kubelet          Node minikube status is now: NodeHasSufficientPID
  Normal  NodeAllocatableEnforced  26m                kubelet          Updated Node Allocatable limit across pods
  Normal  RegisteredNode           25m                node-controller  Node minikube event: Registered Node minikube in Controller
```

No fim, os nós listados serão os responsáveis por hospedar e executar as demais abstrações do Kubernetes, bem como os próprios componentes internos. Por exemplo, na saída do `describe` é possível ver componentes como `etcd-minikube`, `storage-provisioner` e `coredns`. Eles estão sendo chamados de **Pods** aqui, então vamos seguir para a próxima etapa.

### Pods e Namespaces

Em Kubernetes, a unidade mínima de execução é chamada de Pod, que se trata de um conjunto de containers executando dentro de um determinado isolamento. Dentro de um Pod, os containers conseguem se comunicar entre si, mas Pods por padrão não se comunicam entre si. No `describe` anterior foi possível ver vários Pods em execução. Esses são componentes inerentes ao funcionamento do Kubernetes, e por isso foram criados sem a intervenção de um operador.

É interessante, entretanto, notar que próximo a alguns deles há a informação **Namespace**. Em Kubernetes, um Namespace é uma definição de escopo que permite organizar determinados recursos em um mesmo contexto. Recursos que são limitados por Namespace, como Pods, podem interagir apenas com recursos neste mesmo Namespace. Isso permite adicionar ainda mais isolamento, para garantir que aplicações de contextos diferentes não interajam entre si. O Namespace que vemos na saída do `describe` é o **kube-system**, que é reservado para componentes do Kubernetes.

Para colocar em prática, vamos buscar os Pods disponíveis no sistema.
```bash
kubectl get pods
```

Considerando que você não tenha feito nada além da instalação do Minikube, a saída deveria ser vazia:
```
No resources found in default namespace.
```

"*Mas por que, se vimos vários Pods em execução?*", talvez você pergunte. Se olhar direitinho, a saída diz que não há nenhum pod no Namespace *default*. Isso porque a busca de pods permite um parâmetro `-n` para Namespace, que por padrão usa literalmente o namespace padrão, ou *default*. Esse é o Namespace que é utilizado em tudo que for criado pelo administrador quando nada for especificado.

Vamos buscar melhor, no Namespace `kube-system`
```bash
kubectl get pods -n kube-system
```

A saída deveria ser a seguinte agora:
```
NAME                               READY   STATUS    RESTARTS      AGE
coredns-5dd5756b68-vh5bj           1/1     Running   1 (28h ago)   29h
etcd-minikube                      1/1     Running   1 (28h ago)   29h
kube-apiserver-minikube            1/1     Running   1 (28h ago)   29h
kube-controller-manager-minikube   1/1     Running   1 (28h ago)   29h
kube-proxy-5rgxr                   1/1     Running   1 (28h ago)   29h
kube-scheduler-minikube            1/1     Running   1 (28h ago)   29h
storage-provisioner                1/1     Running   2 (44m ago)   29h
```

Olha só! Aqui podemos ver todos aqueles que vimos anteriormente, incluindo algumas informações extras. Conseguimos ver o nome do Pod, a quantidade de containers que ele possui internamente (todos só possuem 1), seu status, idade e por fim a quantidade de reinicializações. O status *Running* aqui, informa que o Pod está saudável e executando, e isso juntamente com *Ready* significa que o Pod pode se comunicar se necessário. Há diversos status para Pods em Kubernetes, e *Running* é normalmente o ideal. Veremos mais sobre eles na [Parte 3](./parte-3-checagem-de-saude.md). Por fim, a informação do número de restarts é crucial para entender se a aplicação falha com muita frequência, e evidencia a capacidade do Kubernetes de reinciá-la caso necessário.    

Adicionalmente, nem todos esses Pods estão soltos. E quando me refiro a soltos, significa "gerenciados individualmente pelo Kubernetes". Na maioria das vezes, são usados recursos para agrupar vários Pods de maneira a tratá-los como um grande conjunto, ao invés de individualmente. Isso é útil para replicação, e aplicar mudanças para o conjunto inteiro de Pods. Há diversas maneiras de fazer isso, e as mais populares são **Deployment**, **DaemonSet** e **StatefulSet**.

Com **Deployments**, é possível tratar um conjunto de Pods como réplicas uma da outra sem identidade específica. Isso é o caso padrão, onde pode se desejar controlar a quantidade de réplicas. As réplicas são distribuídas ao longo do cluster conforme decidido pelo Control Plane (apesar de que há configurações avançadas para que essa decisão seja tendenciosa). Ele é, inclusive, utilizado para controlar o **coredns**. Dê uma olhada:

```bash
kubectl get deployments -n kube-system
```
```
NAME      READY   UP-TO-DATE   AVAILABLE   AGE
coredns   1/1     1            1           29h
```

A saída desse comando parece bastante com o `get pods`, porém mostra informação sobre o **conjunto de réplicas**, ao invés dos **containers de cada pod**. *Ready* aqui se refere à quantidade de Pods coredns, assim como *Up-to-Date* (para saber quais estão na versão mais atualizada, útil para ver o progresso de um update) e *Available* (para saber se há algum que está desconectado da rede por algum motivo).

Com **Daemonsets**, é possível tratar também réplicas sem identidade única, porém garantindo a presença de ao menos um Pod por nó. Ou seja, aumentar a quantidade de réplicas, é aumentar a quantidade de réplicas *por nó*. Esse é o caso do kube-proxy, um componente essencial para o roteamento de rede do nó (e por isso necessário em cada um deles).

```bash
kubectl get daemonsets -n kube-system
```
```
NAME         DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
kube-proxy   1         1         1       1            1           kubernetes.io/os=linux   29h
```

Por fim, há o **StatefulSet**, que garante que cada réplica possua uma identidade persistente, o que pode ser útil para determinadas aplicações. Trataremos disso mais adiante no roteiro.

## Treinando com Pods

Para entender melhor, vamos colocar a mão na massa, porém de leve por enquanto. Vamos começar criando um Pod para um servidor [NGINX](https://www.nginx.com/learn/). Caso não esteja ciente, NGINX é um servidor web bastante utilizado para proxy reverso e balanceamento de carga. É bastante poderoso e popular, e é um caso de uso fáci para exemplificar a implantação de uma aplicação em Kubernetes.

### Criando e observando um Deployment

Vamos criar um **Deployment** para o NGINX utilizando o comando abaixo:
```bash
kubectl create deployment nginx --image=nginx
```
```
deployment.apps/nginx created
```

Esse comando cria por padrão um Deployment chamado *nginx*. Estamos passando o argumento `--image=nginx` para especificar que a imagem que deve executar nos containers dos Pods desse Deployment seja a imagem oficial do NGINX. Como não especificamos nenhum registro Docker, é utilizado o [Docker Hub](https://hub.docker.com/), e como não especificamos nenhuma Tag, é utilizada a tag `latest`. 

Observe também que não foi especificado o Namespace, o que significa que foi criado no *default*. Vamos listar os Deployments contidos nele.
```bash
kubectl get deployment nginx
```
```
NAME    READY   UP-TO-DATE   AVAILABLE   AGE
nginx   1/1     1            1           25s
```

E lá está ele. Ótimo. Já dá para saber também que há apenas uma réplica do NGINX executando atavés desse Deployment. Vamos confirmar:
```bash
kubectl get pods
```
```
NAME                     READY   STATUS    RESTARTS   AGE
nginx-7854ff8877-4jmwl   1/1     Running   0          83s
```

Observe que o Pod possui dois códigos hexadecimais associados a ele, após o nome do Deployment. O primeiro, `7854ff8877`, é o código do Deployment em si; já o segundo, `4jmwl`, é o código deste Pod. Já que se trata de um Deployment, vamos aumentar a quantidade de réplicas. Esses códigos serão aleatórios, e com certeza serão diferentes para o seu caso.

```bash
kubectl scale deployment nginx --replicas=2
```
```
deployment.apps/nginx scaled
```

Vamos ver agora
```bash
kubectl get pods
```
```
NAME                     READY   STATUS    RESTARTS   AGE
nginx-7854ff8877-4jmwl   1/1     Running   0          18m
nginx-7854ff8877-lb9tn   1/1     Running   0          40s
```

Veja que o código do meio se mantém o mesmo na nova réplica, porém o último muda. Observe também as informações do Deployment em si agora.

```bash
kubectl get deployment
```
```
NAME    READY   UP-TO-DATE   AVAILABLE   AGE
nginx   2/2     2            2           22m
```

A informação das duas réplicas está aqui! E também que ambas estão saudáveis e executando. Vamos reduzir novamente para uma para continuar de forma mais particular na que sobrar.

```bash
kubectl scale deployment nginx --replicas=1
```
```
deployment.apps/nginx scaled
```

Vamos dar uma olhada nas informações do Pod em si. Talvez tenha algo interessante. Copie o nome completo do Pod (incluindo o código), e descreva-o com `describe pod`:
```bash
kubectl describe pod <pod-name>
```

Um monte de informação deve ter saído aí! É tão detalhado quanto a descrição de nós, porém obviamente contém informações diferentes. Eventos pelo qual o Pod passou, volumes anexados a ele (que serão explicados na [Parte 2](./parte-2-aplicacao-web.md)), metadados, containers que executa... e por aí vai. Assim como no caso da descrição do nó, aqui há muita informação avançada que não vale a atenção no momento.
```
Name:             nginx-7854ff8877-4jmwl
Namespace:        default
Priority:         0
Service Account:  default
Node:             minikube/192.168.49.2
Start Time:       Mon, 11 Mar 2024 21:33:57 -0300
Labels:           app=nginx
                  pod-template-hash=7854ff8877
Annotations:      <none>
Status:           Running
IP:               10.244.0.5
IPs:
  IP:           10.244.0.5
Controlled By:  ReplicaSet/nginx-7854ff8877
Containers:
  nginx:
    Container ID:   docker://9b95db4e8e0bbf3412de216bb9c36aabcbbf77c3d3640210f908bb83c69fe6b4
    Image:          nginx
    Image ID:       docker-pullable://nginx@sha256:c26ae7472d624ba1fafd296e73cecc4f93f853088e6a9c13c0d52f6ca5865107
    Port:           <none>
    Host Port:      <none>
    State:          Running
      Started:      Mon, 11 Mar 2024 21:33:59 -0300
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-z6c5c (ro)
Conditions:
  Type              Status
  Initialized       True
  Ready             True
  ContainersReady   True
  PodScheduled      True
Volumes:
  kube-api-access-z6c5c:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   BestEffort
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  25m   default-scheduler  Successfully assigned default/nginx-7854ff8877-4jmwl to minikube
  Normal  Pulling    25m   kubelet            Pulling image "nginx"
  Normal  Pulled     25m   kubelet            Successfully pulled image "nginx" in 1.135s (1.135s including waiting)
  Normal  Created    25m   kubelet            Created container nginx
  Normal  Started    25m   kubelet            Started container nginx
```

Mas que tal ver informações específicas da aplicação? Como será que está o comportamento dela? Para isso, Kubernetes permite facilmente visualizar seus logs. Vamos dar uma olhada. De maneira similar ao comando anterior, utilize o nome do Pod no comando abaixo.
```bash
kubectl logs <pod-name>
```

A saída deve mostrar os logs específicos do NGINX. Isto é, os logs que o container de fato está exibindo na saída padrão! Aqui estão os meus. **OBS: é possível usar o argumento `-f` para continuar observando os logs indeterminadamente**.
```
/docker-entrypoint.sh: /docker-entrypoint.d/ is not empty, will attempt to perform configuration
/docker-entrypoint.sh: Looking for shell scripts in /docker-entrypoint.d/
/docker-entrypoint.sh: Launching /docker-entrypoint.d/10-listen-on-ipv6-by-default.sh
10-listen-on-ipv6-by-default.sh: info: Getting the checksum of /etc/nginx/conf.d/default.conf
10-listen-on-ipv6-by-default.sh: info: Enabled listen on IPv6 in /etc/nginx/conf.d/default.conf
/docker-entrypoint.sh: Sourcing /docker-entrypoint.d/15-local-resolvers.envsh
/docker-entrypoint.sh: Launching /docker-entrypoint.d/20-envsubst-on-templates.sh
/docker-entrypoint.sh: Launching /docker-entrypoint.d/30-tune-worker-processes.sh
/docker-entrypoint.sh: Configuration complete; ready for start up
2024/03/12 00:33:59 [notice] 1#1: using the "epoll" event method
2024/03/12 00:33:59 [notice] 1#1: nginx/1.25.4
2024/03/12 00:33:59 [notice] 1#1: built by gcc 12.2.0 (Debian 12.2.0-14)
2024/03/12 00:33:59 [notice] 1#1: OS: Linux 5.15.133.1-microsoft-standard-WSL2
2024/03/12 00:33:59 [notice] 1#1: getrlimit(RLIMIT_NOFILE): 1048576:1048576
2024/03/12 00:33:59 [notice] 1#1: start worker processes
2024/03/12 00:33:59 [notice] 1#1: start worker process 29
2024/03/12 00:33:59 [notice] 1#1: start worker process 30
2024/03/12 00:33:59 [notice] 1#1: start worker process 31
2024/03/12 00:33:59 [notice] 1#1: start worker process 32
2024/03/12 00:33:59 [notice] 1#1: start worker process 33
2024/03/12 00:33:59 [notice] 1#1: start worker process 34
2024/03/12 00:33:59 [notice] 1#1: start worker process 35
2024/03/12 00:33:59 [notice] 1#1: start worker process 36
2024/03/12 00:33:59 [notice] 1#1: start worker process 37
2024/03/12 00:33:59 [notice] 1#1: start worker process 38
2024/03/12 00:33:59 [notice] 1#1: start worker process 39
2024/03/12 00:33:59 [notice] 1#1: start worker process 40
```

Beleza, legal! Mas talvez isso não seja o suficiente, talvez seja necessário *ver* lá dentro. Como será que está o container mesmo? O sistema de arquivo deles talvez dê uma dica. Vai que a aplicação que você está executando não joga na saída padrão, só escreve em arquivos. O Kubernetes permite executar comandos remotos nos Pods, o que nos permite abrir uma sessão de terminal lá dentro (claro, isso depende completamente da imagem conter algum terminal). Por sorte, a imagem do NGINX possui o bash, então vamos entrar.

O comando abaixo pede que o comando seja interativo (por isso o `-it`), e portanto a sessão aberta seja mantida enquanto o comando não retornar completamente. O comando passado é `/bin/bash`, que vem diretamente após os `--`.
```bash
kubectl exec -it <pod-name> -- /bin/bash
```

Novamente, a saída pode ser um pouco diferente a depender da aplicação. No caso do NGINX, ela deve ser mais ou menos assim. Aproveitei e já dei um `ls` lá dentro! Para sair, basta executar `exit`.

```bash
root@nginx-7854ff8877-4jmwl:/# ls
bin   dev                  docker-entrypoint.sh  home  lib64  mnt  proc  run   srv  tmp  var
boot  docker-entrypoint.d  etc                   lib   media  opt  root  sbin  sys  usr
root@nginx-7854ff8877-4jmwl:/#
```

### Deletando e atualizando o Deployment

Agora é uma boa hora para perguntar: "*como que o Kubernetes lida quando eu de fato atualizar alguma coisa?*" Sábia pergunta!

A comunicação com Kubernetes é declarativa, você talvez já tenha entendido isso bem. Isso significa que se diz para o Kubernetes o estado que você deseja, e não a operação que você quer que ele faça. Obviamente, isso não é sempre verdade, afinal existem comandos como `describe` e `log`. Esses (e mais alguns como `delete`) são exceções, até mesmo para permitir uma boa usabilidade. Mas é importante entender que no geral, é o Kubernetes que toma decisões de como controlar os recursos, e não o administrador.

Vamos entender agora como ele lida com atualizações. Primeiro, que tal se atualizarmos a versão do NGINX? Vamos mudar a Tag da imagem para `:alpine-perl` ao invés de `:latest` (que está implícita, mas é a usada). O comando abaixo irá mudar a imagem de todos os containers (apesar de no nosso caso haver apenas um) do nosso Deployment para a imagem especificada.

```bash
kubectl set image deployment nginx *=nginx:alpine-perl
```
```
deployment.apps/nginx image updated
```

Rápido! Olhe logo os Pods, e veja algo interessante
```bash
kubectl get pods
```
```
NAME                     READY   STATUS              RESTARTS   AGE
nginx-76779476db-z8mqw   0/1     ContainerCreating   0          6s
nginx-7854ff8877-4jmwl   1/1     Running             0          42m
```

Olha que interessante. Nós não pedimos para aumentar uma réplica, mas o Kubernetes criou um novo Pod ao invés de atualizar o anterior. E além disso, o status dele está *ContainerCreating*, o que sugere que está apenas começando. Mas por quê? Bem, dê alguns instantes e execute o comando de novo. A saída deve estar assim:
```
NAME                     READY   STATUS    RESTARTS   AGE
nginx-76779476db-z8mqw   1/1     Running   0          3m43s
```

E agora só há o novo! O motivo: Kubernetes não quer deletar algo enquanto não há um substituto. Então ele cria um novo, aguarda até o novo estar pronto e saudável (representado pelo `Ready`), e só então deleta o antigo. Enquanto a nova réplica atualizada não fica saudável, a réplica antiga não é tocada. O nome dessa estratégia é **RollingUpdate**, e ajuda a evitar downtime de aplicações sendo atualizadas! Bem legal, né?

**OBS:** se quiser ver algo legal, defina a quantidade de réplicas como 3 e depois reverta a imagem novamente para `:latest` com os comandos anteriores. Você vai ver o *RollingUpdate* em ação e **em escala**.

Tá bem, mas deixando atualizações a parte, e se uma aplicação estiver falha? Digamos que ela não está funcionando como deveria por causa de um Bug. Bem, isso é configurável, e veremos isso na [Parte 3](./parte-3-checagem-de-saude.md). Mas, em resumo, Kubernetes adere a uma boa prática de sistemas distribuídos chamada **Fail Fast**. Isto é, assim que a aplicação apresente sintomas de problemas, mate-a e a substitua. Isso permite evitar danos por causa do comportamento falho, e se feito rapidamente, causa pouco downtime por falta de serviço. Aplicações portadas para Kubernetes deveriam ser tolerantes a falha, e logo capazes de se recuperarem após terem seus processos terminados abruptamente.

Aí você pode dizer: "*tá, mas eu preciso esperar o NGINX falhar para ver isso?*, e a resposta é "Não!". Você pode usar sua curiosidade e falta de paciência a seu favor. Utilize o comando `delete` e dê uma olhada nesse comportamento em ação. Seja rápido para olhar, entretanto, pois o Kubernetes é bastante rápido!
```bash
kubectl delete pod <pod-name>
```

```bash
kubectl get pods
```
```
NAME                     READY   STATUS    RESTARTS   AGE
nginx-76779476db-27nxx   1/1     Running   0          3s
```

Veja que outro Pod já foi colocado no seu lugar. O Kubernetes busca completar os Pods que faltam para suprir o número de réplicas declarado.

Bom, então para finalizar, vamos agora de fato dar um fim nisso. Vamos deletar o Deployment, pois assim o Kubernetes entenderá que não precisa mais orquestrar aqueles recursos.

```bash
kubectl delete deployment nginx
```
```
deployment.apps "nginx" deleted
```

* Como entrar no pod e ver dentro dele
* Como deletar pod e porque isso não remove permanentemente (k8s só cria outro)
* Como atualizar o pod
* Como deletar deployment


## Conclusões

Na Parte 1 nós vimos que o Kubernetes é declarativo e permite ver várias informações legais sobre os recursos e como estão organizados. Também é possível usar comandos imperativos utilizando `kubectl` e ter acesso a informações que evidenciem o comportamento declarativo da orquestração.

Na [parte a seguir](./parte-2-aplicacao-web.md) exploraremos um exemplo prático com servidor web e banco de dados, onde subimos uma aplicação utilizando manifestos Kubernetes, ao invés de comandos imperativos do `kubectl`.
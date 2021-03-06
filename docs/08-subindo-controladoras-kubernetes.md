# Subindo o Plano de Controle do Kubernetes

Nesse lab você irá subir o Plano de Controle do Kubernetes em três instâncias computacionais e configurar as mesmas para alta disponibilidade. Você também vai criar um balanceador de carga que expõe os Servidores de API do Kubernetes para clientes remotos. Os seguintes componentes serão instalados em cada nó: Servidor de API do Kubernetes, Agendador e Gerenciador de Controladora.

## Pré-requisitos

Os comandos nesse lab devem ser executados em cada instância de controladora:  `controller-0`, `controller-1`, e `controller-2`. Conecte em cada controladora utilizando o comando `gcloud`. Exemplo:

```
gcloud compute ssh controller-0
```

### Executando comandos com tmux
O [tmux](https://github.com/tmux/tmux/wiki) pode ser utilizado para executar comandos em várias instâncias computacionais ao mesmo tempo. Veja o [Executando comandos em paralelo com o tmux](01-pre-requisitos.md#executando-comandos-em-paralelo-com-o-tmux) Na sessão de prerequisitos

## Provisione o Plano de Controle do Kubernetes


Crie o diretório de configuração do kubernetes:

```
sudo mkdir -p /etc/kubernetes/config

```


### Faça o Download e Instale os Binários da Controladora do Kubernetes

Faça o download dos binários oficiais:

```
wget -q --show-progress --https-only --timestamping \
  "https://storage.googleapis.com/kubernetes-release/release/v1.13.6/bin/linux/amd64/kube-apiserver" \
  "https://storage.googleapis.com/kubernetes-release/release/v1.13.6/bin/linux/amd64/kube-controller-manager" \
  "https://storage.googleapis.com/kubernetes-release/release/v1.13.6/bin/linux/amd64/kube-scheduler" \
  "https://storage.googleapis.com/kubernetes-release/release/v1.13.6/bin/linux/amd64/kubectl"



```

Instale os binários do Kubernetes:

```
{
  chmod +x kube-apiserver kube-controller-manager kube-scheduler kubectl
  sudo mv kube-apiserver kube-controller-manager kube-scheduler kubectl /usr/local/bin/
}
```

### Configure o Servidor de API do Kubernetes

```
{
  sudo mkdir -p /var/lib/kubernetes/

  sudo mv ca.pem ca-key.pem kubernetes-key.pem kubernetes.pem \
    service-account-key.pem service-account.pem \
    encryption-config.yaml /var/lib/kubernetes/
}
```

O endereço de IP interno da instância será utilizado para anunciar o Servidor de API aos membros do cluster. Recupere o endereço de IP interno da instância computacional corrente:

```
INTERNAL_IP=$(curl -s -H "Metadata-Flavor: Google" \
  http://metadata.google.internal/computeMetadata/v1/instance/network-interfaces/0/ip)
```

Crie o arquivo _unit_ `kube-apiserver.service`  do systemd:

```
cat <<EOF | sudo tee /etc/systemd/system/kube-apiserver.service
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-apiserver \\
  --advertise-address=${INTERNAL_IP} \\
  --allow-privileged=true \\
  --apiserver-count=3 \\
  --audit-log-maxage=30 \\
  --audit-log-maxbackup=3 \\
  --audit-log-maxsize=100 \\
  --audit-log-path=/var/log/audit.log \\
  --authorization-mode=Node,RBAC \\
  --bind-address=0.0.0.0 \\
  --client-ca-file=/var/lib/kubernetes/ca.pem \\
  --enable-admission-plugins=Initializers,NamespaceLifecycle,NodeRestriction,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota \\
  --enable-swagger-ui=true \\
  --etcd-cafile=/var/lib/kubernetes/ca.pem \\
  --etcd-certfile=/var/lib/kubernetes/kubernetes.pem \\
  --etcd-keyfile=/var/lib/kubernetes/kubernetes-key.pem \\
  --etcd-servers=https://10.240.0.10:2379,https://10.240.0.11:2379,https://10.240.0.12:2379 \\
  --event-ttl=1h \\
  --experimental-encryption-provider-config=/var/lib/kubernetes/encryption-config.yaml \\
  --kubelet-certificate-authority=/var/lib/kubernetes/ca.pem \\
  --kubelet-client-certificate=/var/lib/kubernetes/kubernetes.pem \\
  --kubelet-client-key=/var/lib/kubernetes/kubernetes-key.pem \\
  --kubelet-https=true \\
  --runtime-config=api/all \\
  --service-account-key-file=/var/lib/kubernetes/service-account.pem \\
  --service-cluster-ip-range=10.32.0.0/24 \\
  --service-node-port-range=30000-32767 \\
  --tls-cert-file=/var/lib/kubernetes/kubernetes.pem \\
  --tls-private-key-file=/var/lib/kubernetes/kubernetes-key.pem \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### Configure o Gerenciador de Controladora do Kubernetes

Mova o kubeconfig do `kube-controller-manager` para o seu lugar:

```
sudo mv kube-controller-manager.kubeconfig /var/lib/kubernetes/
```

Crie o arquivo _unit_ `kube-controller-manager.service` do systemd:

```
cat <<EOF | sudo tee /etc/systemd/system/kube-controller-manager.service
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/GoogleCloudPlatform/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-controller-manager \\
  --address=0.0.0.0 \\
  --cluster-cidr=10.200.0.0/16 \\
  --cluster-name=kubernetes \\
  --cluster-signing-cert-file=/var/lib/kubernetes/ca.pem \\
  --cluster-signing-key-file=/var/lib/kubernetes/ca-key.pem \\
  --kubeconfig=/var/lib/kubernetes/kube-controller-manager.kubeconfig \\
  --leader-elect=true \\
  --root-ca-file=/var/lib/kubernetes/ca.pem \\
  --service-account-private-key-file=/var/lib/kubernetes/service-account-key.pem \\
  --service-cluster-ip-range=10.32.0.0/24 \\
  --use-service-account-credentials=true \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### Configure o Agendador do Kubernetes

Mova o kubeconfig do `kube-scheduler` para o seu lugar:

```
sudo mv kube-scheduler.kubeconfig /var/lib/kubernetes/
```

Crie o arquivo de configuração `kube-scheduler.yaml`:

```
cat <<EOF | sudo tee /etc/kubernetes/config/kube-scheduler.yaml
apiVersion: kubescheduler.config.k8s.io/v1alpha1
kind: KubeSchedulerConfiguration
clientConnection:
  kubeconfig: "/var/lib/kubernetes/kube-scheduler.kubeconfig"
leaderElection:
  leaderElect: true
EOF
```

Crie o arquivo _unit_ `kube-scheduler.service` do systemd:

```
cat <<EOF | sudo tee /etc/systemd/system/kube-scheduler.service
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-scheduler \\
  --config=/etc/kubernetes/config/kube-scheduler.yaml \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### Inicie os serviços da controladora

```
{
  sudo systemctl daemon-reload
  sudo systemctl enable kube-apiserver kube-controller-manager kube-scheduler
  sudo systemctl start kube-apiserver kube-controller-manager kube-scheduler
}
```

> Aguarde cerca de 10 segundos até o Servidor de API do Kubernetes inicializar completamente.

### Habilite os healthchecks HTTP

Usaremos um [Google Network Load Balancer](https://cloud.google.com/compute/docs/load-balancing/network) para distriuir tráfego entre os três servidores de API e permitir cada um deles ser terminação TLS e validar os certificados de cliente. O network load balancer somente suporte a health check HTTP, isso significa que o endpoint HTTPS exposto pela servidor de API não pode ser usado. Como medida paleativa, confiramos um nginx webserver nos nós de controller para servir de proxy http2http. Nessa sessão configuraremos o nginx para fazer proxy do http/80 para o `https://127.0.0.1:6443/healthz`



> O ednpoin de API `/healthz` Não requer autenticação por padrão

Instale um webserver básico para tratar os health checks:


```
sudo apt-get install -y nginx
```


Crie uma config de proxy para o api local do node

```
cat  <<EOF | sudo tee /etc/nginx/sites-available/kubernetes.default.svc.cluster.local
server {
  listen      80;
  server_name kubernetes.default.svc.cluster.local;

  location /healthz {
     proxy_pass                    https://127.0.0.1:6443/healthz;
     proxy_ssl_trusted_certificate /var/lib/kubernetes/ca.pem;
  }
}
EOF
```

Habilite a configuraão no nginx

```
{
  sudo ln -s /etc/nginx/sites-available/kubernetes.default.svc.cluster.local /etc/nginx/sites-enabled/
  sudo systemctl restart nginx
  sudo systemctl enable nginx
}
```

### Verificação

```
kubectl get componentstatuses --kubeconfig admin.kubeconfig
```

```
NAME                 STATUS    MESSAGE              ERROR
controller-manager   Healthy   ok
scheduler            Healthy   ok
etcd-2               Healthy   {"health": "true"}
etcd-0               Healthy   {"health": "true"}
etcd-1               Healthy   {"health": "true"}
```

Teste o proxy HTTP para health check:

```
curl -H "Host: kubernetes.default.svc.cluster.local" -i http://127.0.0.1/healthz
```

```
HTTP/1.1 200 OK
Server: nginx/1.14.0 (Ubuntu)
Date: Sun, 30 Sep 2018 17:44:24 GMT
Content-Type: text/plain; charset=utf-8
Content-Length: 2
Connection: keep-alive

ok
```

> Lembre-se de executar os comandos acima em cada nó da controladora: `controller-0`, `controller-1` e `controller-2`.

## RBAC para Autorização do Kubelet

Nessa seção você irá configurar as permissões RBAC para permitir que o Servidor de API do Kubernetes acesse a API do Kubelet em cada nó _worker_. O acesso à API do Kubelet é necessário para recuperar métricas, logs e executar comandos em pods.

> Esse tutorial define a flag `--authorization-mode` do Kubelet para `Webhook`. O modo Webhook utiliza a API [SubjectAccessReview](https://kubernetes.io/docs/admin/authorization/#checking-api-access) para determinar autorização.

```
gcloud compute ssh controller-0
```

Crie a [_ClusterRole_ (Função no Cluster)](https://kubernetes.io/docs/admin/authorization/rbac/#role-and-clusterrole) `system:kube-apiserver-to-kubelet` com permissões de acesso à API do Kubelet e de realizar as tarefas mais comuns associadas à gestão de pods:

```
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:kube-apiserver-to-kubelet
rules:
  - apiGroups:
      - ""
    resources:
      - nodes/proxy
      - nodes/stats
      - nodes/log
      - nodes/spec
      - nodes/metrics
    verbs:
      - "*"
EOF
```

O Servidor de API do Kubernetes autentica com o Kubelet via usuário `kubernetes` utilizando o certificado cliente definido pela flag `--kubelet-client-certificate`.

Vincule a _ClusterRole_ `system:kube-apiserver-to-kubelet` ao usuário `kubernetes`:


```
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: system:kube-apiserver
  namespace: ""
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:kube-apiserver-to-kubelet
subjects:
  - apiGroup: rbac.authorization.k8s.io
    kind: User
    name: kubernetes
EOF
```

## O Balanceador de Carga de Frontend do Kubernetes

Nessa seção você provisionará um balanceador de carga externo para fazer frente ao Servidor de API do Kubernetes. O endereço de IP estático do `kubernetes-the-hard-way` será anexado a esse balanceador de carga.

> As instâncias computacionais criadas nesse tutorial não terão permissão para completar essa seção. Execute os seguintes comandos da mesma máquina utilizada para criar as instâncias computacionais.

Crie os recursos de rede do balanceador de carga externo:

### Provisionando o balanceador


```
{
  ENDERECO_PUBLICO_KUBERNETES=$(gcloud compute addresses describe kubernetes-the-hard-way \
    --region $(gcloud config get-value compute/region) \
    --format 'value(address)')

  gcloud compute http-health-checks create kubernetes \
    --description "Kubernetes Health Check" \
    --host "kubernetes.default.svc.cluster.local" \
    --request-path "/healthz"

  gcloud compute firewall-rules create kubernetes-the-hard-way-allow-health-check \
    --network kubernetes-the-hard-way \
    --source-ranges 209.85.152.0/22,209.85.204.0/22,35.191.0.0/16 \
    --allow tcp

  gcloud compute target-pools create kubernetes-target-pool \
    --http-health-check kubernetes

  zone=( a b c)
  for enum in 0 1 2 ; do
    gcloud compute target-pools add-instances kubernetes-target-pool \
    --instances controller-$enum  --instances-zone=southamerica-east1-${zone[$enum]}
  done

  gcloud compute forwarding-rules create kubernetes-forwarding-rule \
    --address ${ENDERECO_PUBLICO_KUBERNETES} \
    --ports 6443 \
    --region $(gcloud config get-value compute/region) \
    --target-pool kubernetes-target-pool
}
```

### Verificação

Recupere o endereço estático de IP do `kubernetes-the-hard-way`:

```
ENDERECO_PUBLICO_KUBERNETES=$(gcloud compute addresses describe kubernetes-the-hard-way \
  --region $(gcloud config get-value compute/region) \
  --format 'value(address)')
```

Faça uma requisição HTTP para conseguir as informações de versão do Kubernetes:

```
curl --cacert ca.pem https://${ENDERECO_PUBLICO_KUBERNETES}:6443/version
```

> saída

```
{
  "major": "1",
  "minor": "12",
  "gitVersion": "v1.12.0",
  "gitCommit": "0ed33881dc4355495f623c6f22e7dd0b7632b7c0",
  "gitTreeState": "clean",
  "buildDate": "2018-09-27T16:55:41Z",
  "goVersion": "go1.10.4",
  "compiler": "gc",
  "platform": "linux/amd64"
}
```

Próximo: [Subindo os Nós Worker do Kubernetes](09-subindo-workers-kubernetes.md)

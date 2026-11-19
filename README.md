# Laboratório Final de Red Hat OpenShift — 8 horas

**Curso:** 200DEV · **Carga horária:** 8 horas (480 min de atividade, com 1 hora de almoço fora da carga)
**Formato:** prático, individual, em cluster compartilhado · **Versão de referência do cluster:** OpenShift 4.22 (conforme o módulo 9)

---

## 1. Visão geral

### 1.1 Objetivo

Ao final do laboratório, o aluno deverá ser capaz de:

1. Autenticar-se no cluster, usar a CLI `oc` e navegar pelo console.
2. Explicar a arquitetura do cluster (nós, plano de controle, Cluster Operators e Operators).
3. Criar, inspecionar, escalar e diagnosticar cargas de trabalho (pods e Deployments).
4. Aplicar isolamento e segurança com projetos, RBAC, ServiceAccounts e SCC.
5. Gerar imagens dentro do cluster com BuildConfig, S2I e ImageStreams.
6. Expor aplicações com Services e Routes e usar armazenamento persistente.
7. Diagnosticar o cluster com eventos, métricas, logs e `oc adm`.
8. Construir um pipeline de CI com OpenShift Pipelines (Tekton).
9. Entregar com GitOps (OpenShift GitOps / Argo CD), promover e reverter versões.
10. Resolver incidentes em um cenário de plantão.

### 1.2 Escopo

**Dentro do escopo (somente OpenShift):** o conteúdo dos módulos 05 (Fundamentos da Administração do Red Hat OpenShift) e 09 (DevOps e CI/CD no Red Hat OpenShift), nas partes que tratam do OpenShift e de seus componentes.

**Fora do escopo deste laboratório:**

- Virtualização de máquinas virtuais no OpenShift (módulo 06) e automação com Ansible (módulo 07).
- Linux, RHEL, redes e Podman (módulos 01 a 04), exceto quando aparecem diretamente como parte de um recurso do OpenShift.
- Cultura e práticas de DevOps (módulo 08).
- Supply chain e produtos adicionais de segurança (assinatura de imagens, SBOM, RHACS). Aparecem no módulo 9, mas são produtos complementares ao OpenShift, não o núcleo do curso.

### 1.3 Pré-requisitos do aluno

| Item | Detalhe |
|---|---|
| Acesso ao cluster | Usuário e token ou SSO fornecidos pelo instrutor. Permissão de `view` no próprio projeto e `cluster-reader` para leitura de recursos de cluster. |
| CLI `oc` | Versão compatível com o cluster. Baixe em **Help → Command Line Tools** no console. |
| CLI `tkn` | Tekton CLI, usada no Bloco 8. |
| CLI `argocd` | Argo CD CLI, usada no Bloco 9. |
| Git e conta no GitHub | Para fazer fork dos repositórios `lab-openshift-app` e `lab-openshift-config`. |
| Navegador | Acesso ao console web do cluster. |
| Editor de texto | Para criar os arquivos YAML dos blocos 8 e 9. |

### 1.4 Convenções usadas no documento

- `<usuario>`: seu identificador na turma (exemplo: `aluno01`). Ele aparece em nomes de projetos e de recursos. Substitua em todos os comandos.
- `<REGISTRY>`: endereço do registry de imagens indicado pelo instrutor.
- `<SEU-USUARIO-GITHUB>`: seu usuário no GitHub.
- `<usuario>-dev` e `<usuario>-prod`: projetos de desenvolvimento e produção do aluno.
- `<usuario>-incidentes`: projeto criado pelo instrutor para o desafio final.
- Os comandos estão escritos para **PowerShell** (Windows). Em outros shells, as aspas e os escapes dos comandos `jsonpath` mudam.
- O bloco **(instrutor)** indica um comando que exige permissão de administrador ou que é demonstrado pelo instrutor por demorar ou exigir cluster-admin.

---

## 2. Cronograma

| Horário | Duração | Atividade |
|---|---:|---|
| 08:00–08:30 | 30 min | Abertura e checagem do ambiente |
| 08:30–09:15 | 45 min | Bloco 1: Acesso, CLI `oc` e console |
| 09:15–09:30 | 15 min | Pausa |
| 09:30–10:10 | 40 min | Bloco 2: Arquitetura do cluster, nós e Operators |
| 10:10–10:50 | 40 min | Bloco 3: Cargas de trabalho e diagnóstico de pods |
| 10:50–11:30 | 40 min | Bloco 4: Segurança e isolamento |
| 11:30–12:30 | 60 min | Almoço (fora da carga horária) |
| 12:30–13:10 | 40 min | Bloco 5: Imagens, builds e ImageStreams |
| 13:10–13:25 | 15 min | Pausa |
| 13:25–13:50 | 25 min | Bloco 6: Services, Routes, armazenamento e templates |
| 13:50–14:15 | 25 min | Bloco 7: Observabilidade e diagnóstico do cluster |
| 14:15–15:15 | 60 min | Bloco 8: CI com OpenShift Pipelines (Tekton) |
| 15:15–16:15 | 60 min | Bloco 9: GitOps e estratégias de deploy |
| 16:15–17:00 | 45 min | Desafio final: plantão de incidentes e encerramento |

**Total de atividade:** 480 min (8 horas).

---

## 3. Abertura e checagem do ambiente (08:00–08:30)

**Objetivo:** garantir que todos conseguem acessar o cluster e que as ferramentas estão instaladas.

### 3.1 Checagem das ferramentas

```powershell
oc version
tkn version
argocd version --client
git --version
```

**Resultado esperado:** as três CLIs respondem com versão. `oc version` mostra versão do cliente e do servidor.

### 3.2 Login e identidade

```powershell
oc login --token=<TOKEN> --server=<API_URL>
oc whoami
```

**Resultado esperado:** `oc whoami` mostra o seu usuário. Se o cluster usar SSO corporativo, use `oc login --web` no lugar do token.

### 3.3 Repositórios

Confirme que você fez fork dos dois repositórios:

- `lab-openshift-app`: código da aplicação (um `index.html` e um `Dockerfile` que parte de `registry.access.redhat.com/ubi9/httpd-24`).
- `lab-openshift-config`: manifestos com `base/` e `overlays/dev` e `overlays/prod` (Kustomize).

Se o fork ainda não foi feito, faça agora, antes do Bloco 8.

---

## 4. Bloco 1 — Acesso, CLI `oc` e console (08:30–09:15)

**Objetivo:** autenticar-se, descobrir recursos pela CLI e navegar pelas perspectivas do console.

### Conceitos-chave

- `oc` inclui todos os comandos do `kubectl` e acrescenta recursos próprios do OpenShift, como **Projects**, **Routes** e **ImageStreams**.
- Login por **token** (para scripts e automações) ou por **`oc login --web`** (SSO corporativo). No console, o comando **Copy login command** gera um login com token temporário.
- Um **projeto** é um namespace do Kubernetes com recursos, cotas e permissões próprias.
- O console tem duas perspectivas usadas neste laboratório: **Administrator** e **Developer**.

### Exercício 1.1 — Criar seu projeto

```powershell
oc new-project <usuario>-dev --display-name="Lab <usuario> dev"
oc project
oc get projects
```

**Resultado esperado:** o projeto `<usuario>-dev` está ativo (`oc project` mostra o nome correto).

### Exercício 1.2 — Descobrir a API

```powershell
oc api-resources --api-group=apps
oc explain deployment.spec.strategy
oc explain route.spec.tls
```

**Resultado esperado:** você consegue listar os recursos do grupo `apps` e descrever os campos `strategy` e `tls`. Anote o significado do campo `type` de `strategy`.

### Exercício 1.3 — Versões e leitura de cluster

```powershell
oc version
oc get clusterversion
```

**Resultado esperado:** versão do cliente e do servidor. O comando `oc get clusterversion` exige permissão de leitura de cluster. Se negado, anote e siga; o instrutor mostra a saída.

### Exercício 1.4 — Console

No console web:

1. Selecione o projeto `<usuario>-dev` e troque para a perspectiva **Developer**. Abra a visualização **Topology**.
2. Em **Help → Command Line Tools**, localize o download do `oc` compatível com a versão do cluster.
3. Em **Home → API Explorer**, pesquise o recurso `Route`. Anote a versão de API listada.

**Resultado esperado:** você localiza as três telas sem ajuda do instrutor.

### Perguntas de verificação (Bloco 1)

1. Por que o `oc` é descrito como um superset do `kubectl`?
2. Em que situação você usaria `oc login --web` em vez de `oc login --token`?
3. Qual é a função de um projeto no OpenShift?

---

## 5. Bloco 2 — Arquitetura do cluster, nós e Operators (09:30–10:10)

**Objetivo:** identificar os componentes do cluster, o papel dos Cluster Operators e a relação entre nós, máquinas e Operators.

### Conceitos-chave

- **Plano de controle** coordena o cluster (agendamento, controle de políticas, reação a eventos). Os dados do cluster ficam no **etcd**.
- **Nós de computação (workers)** executam os pods das aplicações.
- **Node** é o objeto do Kubernetes que representa um nó. **Machine** é o recurso do OpenShift que descreve a infraestrutura física ou virtual, em nuvem ou no datacenter.
- **Cluster Version Operator (CVO)** controla as atualizações de versão do cluster.
- **Machine Config Operator (MCO)** gerencia o sistema operacional dos nós (RHEL CoreOS) por meio de **MachineConfig**. Alterações nos nós são feitas de forma ordenada: drenagem de cargas, aplicação e reinício.
- Cada componente central aparece como **ClusterOperator**, com as colunas **AVAILABLE**, **PROGRESSING** e **DEGRADED**.
- **Operators** estendem o cluster e são instalados pelo **OperatorHub**. A situação de um Operator aparece em um **ClusterServiceVersion (CSV)**.

### Exercício 2.1 — Nós e seus papéis

```powershell
oc get nodes -o wide
oc get nodes -L node-role.kubernetes.io/control-plane -L node-role.kubernetes.io/worker
oc describe node <NOME-DO-NO>
```

**Resultado esperado:** você identifica o papel de cada nó. No `describe`, localize a seção **Conditions** (Ready, MemoryPressure, DiskPressure, PIDPressure) e compare **Capacity** com **Allocatable**.

### Exercício 2.2 — Cluster Operators (leitura)

```powershell
oc get clusteroperators
```

**Resultado esperado:** todos os operadores com `AVAILABLE=True`, `PROGRESSING=False` e `DEGRADED=False`. Se algum estiver degradado, anote o nome e informe o instrutor.

### Exercício 2.3 — MachineConfigPools (leitura)

```powershell
oc get machineconfigpools
oc get mcp
```

**Resultado esperado:** os pools `master` e `worker` aparecem com `UPDATED=True`. A alteração de um MachineConfig passa por esses pools.

### Exercício 2.4 — Operators instalados

Este laboratório usa os Operators **OpenShift Pipelines** e **OpenShift GitOps**, instalados pelo instrutor.

```powershell
oc get csv -A
oc get csv -A | Select-String -Pattern "pipelines|gitops"
```

**Resultado esperado:** os CSVs de OpenShift Pipelines e OpenShift GitOps aparecem com a fase **Succeeded**. Se não aparecerem, avise o instrutor antes do Bloco 8.

### Exercício 2.5 — OperatorHub no console

No console, abra o **OperatorHub** e pesquise por "OpenShift Pipelines". Veja a página do operador e a lista de **Installed Operators** do projeto. **Não instale nada:** o operador já está instalado.

**Resultado esperado:** você identifica o operador instalado e o estado dele.

### Perguntas de verificação (Bloco 2)

1. Qual é a diferença entre um **Node** e uma **Machine**?
2. O que significa um ClusterOperator com `DEGRADED=True`?
3. Por que alterar um MachineConfig faz o MCO drenar cargas antes de reiniciar um nó?

---

## 6. Bloco 3 — Cargas de trabalho e diagnóstico de pods (10:10–10:50)

**Objetivo:** criar pods e Deployments, escalar, testar a autorrecuperação e diagnosticar os estados `ImagePullBackOff` e `CrashLoopBackOff`.

### Conceitos-chave

- O **pod** é a menor unidade gerenciável. Os containers de um pod compartilham rede e volumes.
- Um **Deployment** declara o número desejado de réplicas. Se um pod cai, o controlador cria outro (**autorrecuperação**).
- `oc run` cria um pod avulso. A opção `--restart` define a política: `Always`, `OnFailure` ou `Never`.
- `ImagePullBackOff`: o cluster não consegue baixar a imagem (nome, tag ou permissão).
- `CrashLoopBackOff`: o container inicia e falha repetidamente.

### Exercício 3.1 — Pod avulso

```powershell
oc run hello --image=registry.access.redhat.com/ubi9/ubi-minimal --restart=Never --command -- sh -c "echo ola-openshift; sleep 600"
oc get pod hello
oc logs hello
oc exec hello -- hostname
oc rsh hello
```

Dentro do `oc rsh`, digite `exit` para sair.

**Resultado esperado:** `oc logs` mostra `ola-openshift`. O `oc exec` retorna o nome do pod.

### Exercício 3.2 — Deployment, réplicas e autorrecuperação

```powershell
oc create deployment web --image=registry.access.redhat.com/ubi9/httpd-24:latest
oc scale deployment/web --replicas=3
oc get pods -l app=web -o wide
```

Escolha um dos pods listados e apague-o:

```powershell
oc delete pod <NOME-DE-UM-POD-WEB>
oc get pods -l app=web -w
```

Pressione `Ctrl+C` para sair do `-w`.

**Resultado esperado:** após a remoção, um pod novo aparece com outro nome e o número de réplicas volta a 3.

### Exercício 3.3 — Diagnóstico de `ImagePullBackOff`

```powershell
oc create deployment erro --image=registry.access.redhat.com/ubi9/imagem-que-nao-existe:1
oc get pods -l app=erro
oc get events --sort-by=.lastTimestamp
```

**Resultado esperado:** o pod entra em `ErrImagePull` e depois em `ImagePullBackOff`. Nos eventos, a mensagem indica a falha ao baixar a imagem. Depois da análise, remova o recurso:

```powershell
oc delete deployment erro
```

### Exercício 3.4 — Diagnóstico de `CrashLoopBackOff`

```powershell
oc run crash --image=registry.access.redhat.com/ubi9/ubi-minimal --restart=Always --command -- sh -c "exit 1"
oc get pod crash -w
```

Quando aparecer `CrashLoopBackOff`, pressione `Ctrl+C` e execute:

```powershell
oc logs crash --previous
oc describe pod crash
```

**Resultado esperado:** em `describe`, a seção **Last State** indica o código de saída e o número de reinícios aumenta. Depois, remova o pod:

```powershell
oc delete pod crash
```

### Exercício 3.5 — Acesso local com port-forward

Em um terminal, mantenha ativo:

```powershell
oc port-forward pod/<NOME-DE-UM-POD-WEB> 8080:8080
```

Em outro terminal:

```powershell
Invoke-WebRequest -UseBasicParsing http://localhost:8080 | Select-Object StatusCode
```

**Resultado esperado:** `StatusCode` igual a `200`. Encerre o port-forward com `Ctrl+C`.

### Limpeza do bloco

```powershell
oc delete pod hello
oc delete deployment web
```

### Perguntas de verificação (Bloco 3)

1. Por que um Deployment não é a mesma coisa que um pod?
2. Você apagou um pod do Deployment `web` e outro apareceu. Qual objeto e qual mecanismo garantem isso?
3. Qual é a diferença entre `ImagePullBackOff` e `CrashLoopBackOff`? Qual é o primeiro comando de diagnóstico para cada um?

---

## 7. Bloco 4 — Segurança e isolamento (10:50–11:30)

**Objetivo:** entender identidades, RBAC, ServiceAccounts, faixas de UID e SCC, e aplicar o princípio do menor privilégio.

### Conceitos-chave

- **Usuários comuns** são as pessoas, autenticadas no provedor de identidade da organização.
- **ServiceAccounts** são identidades para aplicações, pods e pipelines. Não representam pessoas.
- **Usuários de sistema**, como `system:admin`, têm controle amplo sobre o cluster.
- **RBAC** vincula papéis (`view`, `edit`, `admin`, `cluster-admin`) a usuários ou grupos dentro de um projeto.
- Cada projeto recebe uma **faixa de UIDs** (anotação `openshift.io/sa.scc.uid-range`). Os containers não rodam como root: o OpenShift injeta um UID da faixa do projeto.
- **Security Context Constraints (SCC)** limitam os privilégios que um pod pode usar (capabilities, SELinux, tipos de volume). A SCC padrão é restritiva.
- Evite `anyuid` e `privileged` sem justificativa. Se for necessário, use uma ServiceAccount dedicada.

### Exercício 4.1 — RBAC

```powershell
oc get rolebindings -n <usuario>-dev
oc auth can-i create deployments -n <usuario>-dev
oc auth can-i create namespaces
oc auth can-i get secrets -n openshift-gitops
```

**Resultado esperado:** `yes` para a primeira consulta, `no` para a segunda. Na terceira, o resultado depende da configuração da turma; anote-o.

**Opcional, em dupla:** conceda `view` ao seu colega e depois revogue:

```powershell
oc adm policy add-role-to-user view <USUARIO-DO-COLEGA> -n <usuario>-dev
oc get rolebindings -n <usuario>-dev
oc adm policy remove-role-from-user view <USUARIO-DO-COLEGA> -n <usuario>-dev
```

### Exercício 4.2 — ServiceAccount dedicada

```powershell
oc create serviceaccount app-sa
oc create deployment seguro --image=registry.access.redhat.com/ubi9/httpd-24:latest
oc set serviceaccount deployment/seguro app-sa
oc get deployment seguro -o jsonpath='{.spec.template.spec.serviceAccountName}'
```

**Resultado esperado:** a última linha imprime `app-sa`. O Deployment passa a usar a identidade `app-sa`, e não a identidade padrão do projeto.

### Exercício 4.3 — UIDs e SCC

```powershell
oc get namespace <usuario>-dev -o jsonpath='{.metadata.annotations.openshift\.io/sa\.scc\.uid-range}'
oc get scc
oc get pods -l app=seguro -o jsonpath='{.items[0].metadata.annotations.openshift\.io/scc}'
oc exec deploy/seguro -- id -u
```

**Resultado esperado:**

- A anotação do projeto mostra uma faixa de UIDs, por exemplo `1000680000/10000`.
- O pod está associado a uma SCC restritiva (por exemplo, `restricted-v2`).
- O `id -u` retorna um UID que pertence à faixa do projeto, e não `0`.

### Exercício 4.4 — Segredos

```powershell
oc create secret generic credencial-lab --from-literal=usuario=lab --from-literal=senha=nao-use-em-producao
oc get secret credencial-lab -o yaml
```

No YAML, os valores aparecem em **base64**. Decodifique o campo `senha` no PowerShell:

```powershell
[System.Text.Encoding]::UTF8.GetString([Convert]::FromBase64String("<VALOR-BASE64-DO-CAMPO-SENHA>"))
```

**Resultado esperado:** você entende que base64 não é cifragem. Por isso o RBAC restringe a leitura de Secrets. Remova o segredo de teste ao final:

```powershell
oc delete secret credencial-lab
```

### Limpeza do bloco

Mantenha o Deployment `seguro` até o fim do Bloco 5. Ele não é usado depois, então pode ser removido no fim do Bloco 5 com `oc delete deployment seguro`.

### Perguntas de verificação (Bloco 4)

1. Qual é a diferença entre um usuário e uma ServiceAccount?
2. O que o papel `view` permite e o que o papel `edit` acrescenta?
3. Por que o seu pod roda com um UID da faixa do projeto, e não com `0`?
4. Em que situação o uso de `anyuid` pode ser justificado? Como você limitaria o impacto?

---

## 8. Bloco 5 — Imagens, builds e ImageStreams (12:30–13:10)

**Objetivo:** gerar a imagem da aplicação dentro do cluster com `oc new-app`, acompanhar o build, entender ImageStreams e gatilhos.

### Conceitos-chave

- Imagens são compostas por camadas somente leitura. Uma **tag** (como `latest`) é um alias que pode mudar. Um **digest** (`sha256:...`) identifica o conteúdo exato.
- Um **BuildConfig** define fonte, estratégia, imagem de saída e gatilhos. Cada execução gera um objeto **Build**, com logs e status.
- **Source-to-Image (S2I)** injeta o código em uma imagem *builder* e gera a imagem final, sem Dockerfile.
- A **estratégia Docker** usa o `Dockerfile` ou `Containerfile` do repositório, com controle total das camadas.
- Um **ImageStream** é uma abstração sobre imagens e tags. Quando uma tag muda, ela pode disparar um novo rollout (gatilho **ImageChange**).
- `oc new-app` cria BuildConfig, ImageStream, Deployment e Service a partir de um repositório Git, em um único comando.
- A estratégia **JenkinsPipeline** do BuildConfig está descontinuada. Para CI, use OpenShift Pipelines (Bloco 8).
- **Builds for Red Hat OpenShift** (Shipwright) usam os objetos Build e BuildRun e a CLI `shp`.

### Exercício 5.1 — Criar a aplicação a partir do Git

```powershell
oc new-app https://github.com/sclorg/nodejs-ex.git --name=nodeapp -n <usuario>-dev
```

**Resultado esperado:** a saída lista os recursos criados (BuildConfig, ImageStream, Deployment, Service).

### Exercício 5.2 — Acompanhar o build

```powershell
oc get bc,builds,is -n <usuario>-dev
oc logs -f bc/nodeapp -n <usuario>-dev
oc get builds -n <usuario>-dev
```

**Resultado esperado:** o build `nodeapp-1` termina com status `Complete`. A ImageStream `nodeapp` tem a tag `latest`.

### Exercício 5.3 — Rebuild e gatilhos

```powershell
oc start-build nodeapp -n <usuario>-dev --follow
oc describe bc nodeapp -n <usuario>-dev
```

No `describe`, localize a seção **Triggers** (ConfigChange e ImageChange).

**Resultado esperado:** um novo build (`nodeapp-2`) termina com `Complete`. Você consegue explicar qual gatilho dispara o rollout depois que a imagem nova é publicada.

### Exercício 5.4 — Tag versus digest

```powershell
oc get is nodeapp -n <usuario>-dev -o jsonpath='{.status.tags[0].items[0].image}'
oc get deployment nodeapp -n <usuario>-dev -o jsonpath='{.spec.template.spec.containers[0].image}'
```

**Resultado esperado:** a primeira linha mostra um digest `sha256:...`. A segunda mostra a referência ao registry interno. Escreva em uma frase por que o digest é mais reprodutível que a tag `latest`.

### Exercício 5.5 — Shipwright (demonstração do instrutor)

O instrutor demonstra os comandos da CLI `shp`:

```powershell
shp build create --help
shp build run --help
```

Compare o par **Build** e **BuildRun** do Shipwright com o par **BuildConfig** e **Build** do OpenShift. Responda: quando a organização deve preferir um ou outro?

### Limpeza do bloco

Remova o Deployment de teste e mantenha o `nodeapp` para o Bloco 6:

```powershell
oc delete deployment seguro -n <usuario>-dev
```

### Perguntas de verificação (Bloco 5)

1. Quando você escolheria S2I em vez da estratégia Docker?
2. Por que a tag `latest` é menos previsível que um digest?
3. O que faz um gatilho ImageChange?
4. Por que a estratégia JenkinsPipeline não deve ser usada em um projeto novo?

---

## 9. Bloco 6 — Services, Routes, armazenamento e templates (13:25–13:50)

**Objetivo:** expor a aplicação para fora do cluster, anexar armazenamento persistente e ler um template parametrizado.

### Conceitos-chave

- **Service** cria um endereço interno estável para os pods, com balanceamento entre eles.
- **Route** expõe um Service para fora do cluster, com hostname e TLS (terminação *edge*, *passthrough* ou *reencrypt*).
- **PersistentVolumeClaim (PVC)** é o pedido de armazenamento feito pela aplicação. A **StorageClass** provisiona o volume físico. Sem StorageClass compatível, o PVC fica `Pending`.
- **Template** é um manifesto com parâmetros. O comando `oc process` gera os objetos a partir dele.

### Exercício 6.1 — Route HTTP

```powershell
oc get svc nodeapp -n <usuario>-dev
oc expose service nodeapp -n <usuario>-dev
oc get route nodeapp -n <usuario>-dev -o jsonpath='{.spec.host}'
```

Use o host retornado na URL:

```powershell
Invoke-WebRequest -UseBasicParsing http://<HOST-DA-ROUTE> | Select-Object StatusCode
```

**Resultado esperado:** `StatusCode` igual a `200` e a página da aplicação.

### Exercício 6.2 — Route com TLS (edge)

```powershell
oc create route edge nodeapp-tls --service=nodeapp --insecure-policy=Redirect -n <usuario>-dev
oc get route nodeapp-tls -n <usuario>-dev
```

**Resultado esperado:** a route aparece com a terminação `edge`. Ao acessar por HTTPS, o navegador pode mostrar um alerta de certificado, porque o certificado padrão do cluster pode ser emitido por uma CA interna.

### Exercício 6.3 — Armazenamento persistente

Anexe um volume ao Deployment (isso cria o PVC):

```powershell
oc set volume deployment/nodeapp --add --name=dados -t pvc --claim-name=dados-pvc --claim-size=1Gi --mount-path=/opt/app-root/data -n <usuario>-dev
oc get pvc dados-pvc -n <usuario>-dev
oc get storageclass
```

**Resultado esperado:** o PVC `dados-pvc` fica `Bound`. Se ficar `Pending`, use `oc describe pvc dados-pvc` e veja a seção **Events**. A causa mais comum é a falta de StorageClass padrão (veja o Anexo C).

### Exercício 6.4 — Templates

```powershell
oc get templates -n openshift
oc process -n openshift mysql-persistent --parameters
```

**Resultado esperado:** a lista de parâmetros do template (como nome de usuário, senha e capacidade do volume). **Não aplique** o template: o objetivo é só ler os parâmetros.

### Perguntas de verificação (Bloco 6)

1. Qual é a diferença entre um Service e uma Route?
2. Qual é a relação entre PV, PVC e StorageClass?
3. O que é um template e quando ele é útil?

---

## 10. Bloco 7 — Observabilidade e diagnóstico do cluster (13:50–14:15)

**Objetivo:** usar métricas, eventos, alertas e as ferramentas `oc adm` para diagnosticar o cluster e a aplicação.

### Conceitos-chave

- O monitoramento do OpenShift coleta métricas com **Prometheus**. **ServiceMonitor** e **PodMonitor** definem quais alvos são monitorados.
- Alertas aparecem em **Observe → Alerting**, organizados por severidade (Warning, Critical).
- Eventos têm tipo **Normal** ou **Warning**, e podem ser filtrados por tipo e por recurso.
- `oc adm top` mostra o consumo de CPU e memória de nós e pods.
- `oc adm node-logs` mostra logs de serviços do nó, como `crio` e `kubelet`.
- `oc adm inspect` coleta dados de um namespace ou operador. `oc adm must-gather` coleta o diagnóstico completo para o suporte da Red Hat.
- `oc debug node` abre uma sessão de diagnóstico conectada ao nó.

### Exercício 7.1 — Consumo de recursos

```powershell
oc adm top pods -n <usuario>-dev
oc adm top nodes
```

**Resultado esperado:** uma tabela com CPU e memória por pod e por nó. Anote o pod que mais consome memória no seu projeto.

### Exercício 7.2 — Eventos

```powershell
oc get events -n <usuario>-dev --sort-by=.lastTimestamp
oc get events -n <usuario>-dev --field-selector type=Warning
```

**Resultado esperado:** você lista os eventos `Warning` do seu projeto. Se a lista estiver vazia, gere um evento com `oc run erro-teste --image=registry.access.redhat.com/ubi9/imagem-que-nao-existe:1`, observe o Warning e remova o pod depois com `oc delete pod erro-teste`.

### Exercício 7.3 — Console: alertas e métricas

No console, na perspectiva **Administrator**:

1. Abra **Observe → Alerting** e liste os alertas ativos. Abra um e leia a descrição.
2. Abra **Observe → Metrics** e execute a consulta:

```
kube_pod_info{namespace="<usuario>-dev"}
```

**Resultado esperado:** o gráfico ou a tabela mostram os pods do seu projeto.

### Exercício 7.4 — Logs, inspeção e must-gather (instrutor, com acompanhamento)

```powershell
oc adm node-logs <NOME-DO-NO> -u crio --tail=20
oc adm inspect ns/<usuario>-dev --dest-dir=.\inspect
oc adm must-gather --dest-dir=.\must-gather
```

O `must-gather` pode levar vários minutos. O instrutor pode executá-lo antes da aula e mostrar o resultado. Consulte `oc adm node-logs --help` se as opções mudarem na versão do cluster.

**Resultado esperado:** os logs do `crio` aparecem; a pasta `inspect` contém os recursos do namespace; a pasta `must-gather` contém os recursos e logs coletados.

### Exercício 7.5 — Sessão de diagnóstico no nó (instrutor)

```powershell
oc debug node/<NOME-DO-NO>
```

No shell do nó, execute:

```
chroot /host
crictl ps
exit
```

**Resultado esperado:** a lista de containers em execução no nó. Saia do shell com `exit`.

### Perguntas de verificação (Bloco 7)

1. Quando você usaria `oc adm top` e quando usaria **Observe → Metrics**?
2. Para que serve o `must-gather` e por que o resultado é anexado a um chamado de suporte?
3. Qual é a diferença entre um evento `Normal` e um evento `Warning`?

---

## 11. Bloco 8 — CI com OpenShift Pipelines (Tekton) (14:15–15:15)

**Objetivo:** criar um pipeline de CI que clona o código, constrói a imagem no cluster, publica no registry e registra o digest como resultado.

### Conceitos-chave

- **Step**: um container que executa um comando. **Task**: sequência de steps que roda no mesmo pod, compartilhando um workspace.
- **Pipeline**: grafo de tasks, com ordem definida por `runAfter`.
- **TaskRun** e **PipelineRun** são as execuções concretas de uma Task e de um Pipeline.
- **Params** (entradas) e **Results** (saídas tipadas) conectam as tasks. Exemplo: o digest gerado pelo build é usado pelo passo seguinte.
- **Workspaces** são volumes compartilhados entre as tasks (PVC, Secret ou ConfigMap).
- **Resolvers** buscam tasks no cluster, no Git ou em bundles OCI. Eles substituem as antigas ClusterTasks.
- **TektonConfig** é o ponto único de configuração do Operator OpenShift Pipelines.
- **Triggers** (EventListener, TriggerBinding, TriggerTemplate, Interceptors) disparam PipelineRuns a partir de webhooks. **Pipelines as Code (PAC)** mantém a definição do pipeline em `.tekton/`, no próprio repositório.
- **Princípio:** o CI constrói e publica a imagem. Ele não faz deploy direto em produção. O deploy é feito pelo CD (Bloco 9).

### Exercício 8.1 — Verificar o ambiente

```powershell
tkn version
oc get csv -A | Select-String -Pattern "pipelines"
oc get pods -n openshift-pipelines
oc get sa pipeline -n <usuario>-dev
```

**Resultado esperado:** o CSV aparece como `Succeeded` e os pods do namespace `openshift-pipelines` estão `Running`. A ServiceAccount `pipeline` existe no seu projeto. Se ela não existir, crie-a com `oc create serviceaccount pipeline -n <usuario>-dev` e avise o instrutor.

### Exercício 8.2 — Projeto de produção e credencial do registry

Crie o projeto de produção:

```powershell
oc new-project <usuario>-prod --display-name="Lab <usuario> prod"
```

Crie a credencial do registry e vincule-a à ServiceAccount `pipeline`:

```powershell
oc create secret docker-registry registry-cred --docker-server=<REGISTRY> --docker-username=<USUARIO-DO-REGISTRY> --docker-password=<TOKEN-DO-REGISTRY> -n <usuario>-dev
oc secrets link pipeline registry-cred --for=pull,mount -n <usuario>-dev
oc get sa pipeline -n <usuario>-dev -o yaml
```

**Resultado esperado:** o YAML da ServiceAccount lista `registry-cred` entre os secrets vinculados. A credencial fica na ServiceAccount, e não nos steps do pipeline.

**Repositórios privados:** se o fork do seu repositório for privado, crie um Secret do tipo `kubernetes.io/basic-auth` com a anotação `tekton.dev/git-0: https://github.com` e vincule-o à ServiceAccount `pipeline`.

### Exercício 8.3 — Criar o PVC e o pipeline

Crie o arquivo `pvc.yaml` com o conteúdo abaixo:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: source-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

Crie o arquivo `pipeline.yaml` com o conteúdo abaixo:

```yaml
apiVersion: tekton.dev/v1
kind: Pipeline
metadata:
  name: app-pipeline
spec:
  description: Clona o repositório da aplicação, constrói a imagem e publica no registry.
  params:
    - name: git-url
      type: string
    - name: image
      type: string
  workspaces:
    - name: source
  results:
    - name: image-digest
      description: Digest da imagem publicada
      value: $(tasks.build.results.IMAGE_DIGEST)
  tasks:
    - name: clone
      taskRef:
        resolver: cluster
        params:
          - name: kind
            value: task
          - name: name
            value: git-clone
          - name: namespace
            value: openshift-pipelines
      params:
        - name: url
          value: $(params.git-url)
      workspaces:
        - name: output
          workspace: source
    - name: build
      runAfter:
        - clone
      taskRef:
        resolver: cluster
        params:
          - name: kind
            value: task
          - name: name
            value: buildah
          - name: namespace
            value: openshift-pipelines
      params:
        - name: IMAGE
          value: $(params.image)
      workspaces:
        - name: source
          workspace: source
```

Aplique os arquivos e inspecione o pipeline:

```powershell
oc apply -f pvc.yaml -n <usuario>-dev
oc apply -f pipeline.yaml -n <usuario>-dev
tkn pipeline describe app-pipeline -n <usuario>-dev
```

**Resultado esperado:** o `describe` mostra as tasks `clone` e `build`, com `build` depois de `clone`. O instrutor confirma antes da aula os nomes das tasks disponíveis em `openshift-pipelines` com `tkn task list -n openshift-pipelines`.

### Exercício 8.4 — Executar o pipeline

```powershell
tkn pipeline start app-pipeline -n <usuario>-dev -p git-url=https://github.com/<SEU-USUARIO-GITHUB>/lab-openshift-app.git -p image=<REGISTRY>/<usuario>/lab-app:v1 -w name=source,claimName=source-pvc --showlog
```

**Resultado esperado:** as tasks `clone` e `build` terminam com `Succeeded`. A imagem `lab-app:v1` é publicada no registry.

### Exercício 8.5 — Analisar a execução e o digest

```powershell
tkn pr list -n <usuario>-dev
tkn pr describe --last -n <usuario>-dev
tkn pr logs --last -f -n <usuario>-dev
```

**Resultado esperado:** a seção **Results** do PipelineRun mostra `image-digest` com um valor `sha256:...`. **Anote esse digest**: ele será usado no Bloco 9.

### Exercício 8.6 — Console

Na perspectiva **Developer**, abra **Pipelines** no projeto `<usuario>-dev`, selecione `app-pipeline` e abra a aba de execuções. Veja os logs de cada task.

**Resultado esperado:** o console mostra as mesmas execuções e o mesmo resultado que a CLI.

### Exercício 8.7 — Caminho do evento (discussão)

O instrutor mostra um EventListener, um TriggerBinding, um TriggerTemplate e um arquivo em `.tekton/` (Pipelines as Code). Em grupo, descreva o caminho desde um `git push` até a criação de um PipelineRun.

### Perguntas de verificação (Bloco 8)

1. Qual é a diferença entre uma Task e um Pipeline? E entre TaskRun e PipelineRun?
2. Como o resultado de uma task é usado por outra task?
3. Por que as credenciais do registry ficam vinculadas à ServiceAccount, e não aos steps?
4. Por que o CI não deve fazer deploy direto em produção?

---

## 12. Bloco 9 — GitOps e estratégias de deploy (15:15–16:15)

**Objetivo:** entregar a aplicação com OpenShift GitOps (Argo CD), observar a reconciliação, promover uma versão por pull request e reverter pelo Git.

### Conceitos-chave

- No modelo **GitOps**, o estado desejado está em manifestos YAML no Git. O cluster **puxa** as mudanças. O CI não precisa de credenciais de produção.
- **Application** liga um caminho do repositório (YAML, Kustomize ou Helm) a um cluster e namespace de destino.
- **AppProject** limita repositórios, clusters e tipos de recurso permitidos. **ApplicationSet** gera Applications em lote.
- **Sync policies:** `automated` com `prune` (remove o que saiu do Git) e `selfHeal` (reverte mudanças manuais). Quando o cluster diverge do Git, a Application fica **OutOfSync**.
- Para um namespace ser gerenciado pela instância padrão, ele precisa do rótulo `argocd.argoproj.io/managed-by=openshift-gitops`.
- **Estratégias de deploy:** `RollingUpdate` (com `maxSurge` e `maxUnavailable`), `Recreate`, **Blue-Green** (troca de backend na Route) e **Canary** (`alternateBackends` com pesos na Route). O Argo Rollouts adiciona análise automática de métricas.
- **Rollback:** a forma recomendada é reverter o commit no Git (`git revert`). `oc rollout undo` é uma reversão manual que conflita com o GitOps.

### Exercício 9.1 — Verificar o OpenShift GitOps

```powershell
oc get csv -A | Select-String -Pattern "gitops"
oc get pods -n openshift-gitops
oc get route openshift-gitops-server -n openshift-gitops -o jsonpath='{.spec.host}'
argocd login <HOST-DO-ARGOCD> --sso
argocd version
```

**Resultado esperado:** o CSV aparece como `Succeeded`. O login no Argo CD funciona pelo SSO (abre o navegador).

### Exercício 9.2 — Liberar os namespaces para o Argo CD

```powershell
oc label namespace <usuario>-dev argocd.argoproj.io/managed-by=openshift-gitops
oc label namespace <usuario>-prod argocd.argoproj.io/managed-by=openshift-gitops
```

**Resultado esperado:** os dois namespaces aparecem com o rótulo (`oc get namespace <usuario>-dev --show-labels`).

### Exercício 9.3 — Preparar o repositório de configuração

No seu fork de `lab-openshift-config`, edite `overlays/dev/kustomization.yaml` para apontar para a imagem gerada no Bloco 8:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../base
images:
  - name: lab-app
    newName: <REGISTRY>/<usuario>/lab-app
    newTag: v1
```

O `overlays/prod/kustomization.yaml` tem a mesma estrutura, com a tag `v1`. Faça commit e push.

**Resultado esperado:** os dois overlays renderizam manifestos sem erro. Confira com `oc kustomize overlays/dev` (dentro da pasta do repositório).

### Exercício 9.4 — Criar a Application de desenvolvimento

Crie o arquivo `application-dev.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: app-dev
  namespace: openshift-gitops
spec:
  project: default
  source:
    repoURL: https://github.com/<SEU-USUARIO-GITHUB>/lab-openshift-config.git
    targetRevision: main
    path: overlays/dev
  destination:
    server: https://kubernetes.default.svc
    namespace: <usuario>-dev
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

Aplique e sincronize:

```powershell
oc apply -f application-dev.yaml
argocd app get app-dev
argocd app sync app-dev
argocd app get app-dev
oc get deploy,svc,route -n <usuario>-dev
```

**Resultado esperado:** `Sync Status` igual a `Synced` e `Health Status` igual a `Healthy`. Os recursos do repositório existem no seu namespace de desenvolvimento.

### Exercício 9.5 — Simular drift

```powershell
oc scale deployment/lab-app --replicas=5 -n <usuario>-dev
oc get deploy lab-app -n <usuario>-dev -w
```

Pressione `Ctrl+C` quando o número de réplicas voltar.

**Resultado esperado:** o `selfHeal` do Argo CD devolve o Deployment ao número de réplicas definido no Git. Explique em uma frase por que isso acontece.

### Exercício 9.6 — Nova versão pela pipeline (simulação do passo de CI)

1. No seu fork de `lab-openshift-app`, altere o `index.html` para mostrar o texto `versão 2`. Faça commit e push.
2. Execute o pipeline com a nova tag:

```powershell
tkn pipeline start app-pipeline -n <usuario>-dev -p git-url=https://github.com/<SEU-USUARIO-GITHUB>/lab-openshift-app.git -p image=<REGISTRY>/<usuario>/lab-app:v2 -w name=source,claimName=source-pvc --showlog
tkn pr describe --last -n <usuario>-dev
```

3. Anote o digest `sha256:...` do resultado (`image-digest`).
4. No seu fork de `lab-openshift-config`, altere `overlays/dev/kustomization.yaml` para `newTag: v2`, ou para o digest da imagem. Faça commit e push. Em um fluxo completo, esse passo seria automatizado pelo CI. Aqui, você faz manualmente.
5. Sincronize e verifique:

```powershell
argocd app diff app-dev
argocd app sync app-dev
oc get deploy lab-app -n <usuario>-dev -o jsonpath='{.spec.template.spec.containers[0].image}'
```

**Resultado esperado:** a imagem do Deployment aponta para a versão 2. Acesse a Route de desenvolvimento e confirme o texto `versão 2`.

### Exercício 9.7 — Promoção para produção por pull request

1. Crie o arquivo `application-prod.yaml` a partir de `application-dev.yaml`, trocando `name` para `app-prod`, `path` para `overlays/prod` e `namespace` para `<usuario>-prod`. Aplique e sincronize:

```powershell
oc apply -f application-prod.yaml
argocd app sync app-prod
```

2. No seu fork de `lab-openshift-config`, crie uma branch `promocao-v2`. Altere `overlays/prod/kustomization.yaml` para `newTag: v2` (ou para o mesmo digest do dev). Faça commit, push e abra um **pull request** do `promocao-v2` para o `main`.
3. Faça o merge do pull request. Depois sincronize:

```powershell
argocd app sync app-prod
argocd app get app-prod
oc get pods -n <usuario>-prod
```

**Resultado esperado:** `app-prod` fica `Synced` e `Healthy`, e os pods de produção usam a versão 2. Em produção, muitas equipes desligam o `automated` para exigir aprovação manual. Discuta essa escolha com o instrutor.

### Exercício 9.8 — Rollback pelo Git

No seu fork de `lab-openshift-config`, reverta o commit de promoção:

```powershell
git log --oneline -5
git revert <SHA-DO-COMMIT-DE-PROMOCAO>
git push
argocd app sync app-prod
argocd app history app-prod
```

**Resultado esperado:** o histórico do Argo CD mostra a nova sincronização. A Route de produção volta a mostrar a versão 1.

### Exercício 9.9 — Reversão manual e conflito com o GitOps

Com a aplicação de desenvolvimento na versão 2, execute um rollback manual:

```powershell
oc rollout history deployment/lab-app -n <usuario>-dev
oc rollout undo deployment/lab-app -n <usuario>-dev
oc rollout status deployment/lab-app -n <usuario>-dev
argocd app get app-dev
```

**Resultado esperado:** o `rollout undo` volta a versão anterior, mas o `selfHeal` do Argo CD reverte a mudança, porque o Git ainda aponta para a versão 2. Explique por que o caminho correto é o Git.

### Exercício 9.10 — Estratégias de deploy (leitura)

Leia o trecho abaixo, que mostra uma Route com dois backends e pesos (canary com 10% do tráfego para a nova versão):

```yaml
spec:
  to:
    kind: Service
    name: lab-app
    weight: 90
  alternateBackends:
    - kind: Service
      name: lab-app-v2
      weight: 10
```

Inspecione a estratégia atual do Deployment:

```powershell
oc get deployment lab-app -n <usuario>-dev -o jsonpath='{.spec.strategy}'
```

**Resultado esperado:** você identifica o tipo da estratégia e os valores de `maxSurge` e `maxUnavailable`, quando presentes. Responda: qual estratégia exige duas versões ao mesmo tempo? Qual delas derruba todos os pods antes de subir a nova versão?

### Perguntas de verificação (Bloco 9)

1. Por que a Application fica `OutOfSync` depois de um `oc scale`?
2. Qual é a diferença entre `prune` e `selfHeal`?
3. Por que o rollback recomendado é `git revert`, e não `oc rollout undo`?
4. Quando você escolheria `Recreate` em vez de `RollingUpdate`?
5. Como você configuraria um canary de 10% em uma Route?

---

## 13. Desafio final — Plantão de incidentes (16:15–17:00)

**Objetivo:** resolver quatro incidentes em um projeto de incidentes, usando os conceitos dos blocos 1 a 9. Depois, fazer o fechamento do laboratório.

### 13.1 Regras

- O trabalho é **individual**. Use a CLI `oc` e o console.
- Os incidentes estão no projeto `<usuario>-incidentes`, criado pelo instrutor. Esse projeto **não** é gerenciado pelo Argo CD, então as correções feitas à mão permanecem.
- **Não apague o projeto** nem os recursos fora dos incidentes.
- Cada chamado tem um critério de aceite. Você só considera o chamado resolvido quando o critério é atendido.
- Tempo sugerido: 7 minutos por chamado (28 min), mais 2 minutos de checagem final.

### 13.2 Chamados

**CH-01 — "O app `inc-app` não sobe"**

- **Sintoma relatado:** o Deployment `inc-app` não tem nenhum pod disponível.
- **Critério de aceite:** o pod de `inc-app` está `Running` e `Ready`.
- **Pista:** `oc get pods -l app=inc-app` e `oc get events`.

**CH-02 — "A URL do `inc-svc` retorna 503"**

- **Sintoma relatado:** a Route `inc-svc` responde com erro 503 ("Application is not available").
- **Critério de aceite:** `Invoke-WebRequest` na URL da Route retorna `StatusCode` `200`.
- **Pista:** verifique se o Service tem endpoints (`oc get endpoints inc-svc`) e compare o seletor do Service com as labels dos pods (`oc get pods --show-labels`).

**CH-03 — "O PVC `inc-pvc` está pendente"**

- **Sintoma relatado:** o PVC `inc-pvc` não sai do estado `Pending`.
- **Critério de aceite:** `oc get pvc inc-pvc -n <usuario>-incidentes` mostra `Bound`.
- **Pista:** `oc describe pvc inc-pvc` e `oc get storageclass`. A correção pode exigir recriar o PVC com uma StorageClass válida.

**CH-04 — "O `inc-worker` não cria todas as réplicas"**

- **Sintoma relatado:** o Deployment `inc-worker` pede 6 réplicas e tem menos pods em execução. Há mensagens de erro sobre cota.
- **Critério de aceite:** o `inc-worker` tem os pods em execução sem nenhum evento de `exceeded quota`, e o número de pods do projeto está dentro da cota.
- **Pista:** `oc get resourcequota -n <usuario>-incidentes` e `oc describe resourcequota`. Você não pode aumentar a cota: ajuste o número de réplicas para caber nela.

### 13.3 Checagem final e fechamento (16:45–17:00)

Marque cada item:

- [ ] Login e projeto criados (Bloco 1)
- [ ] Pods e Deployment diagnosticados, com autorrecuperação observada (Bloco 3)
- [ ] RBAC, ServiceAccount e UID de execução verificados (Bloco 4)
- [ ] Build com S2I e ImageStream acompanhado (Bloco 5)
- [ ] Route e PVC criados (Bloco 6)
- [ ] Eventos e métricas consultados (Bloco 7)
- [ ] Pipeline executado com o digest registrado (Bloco 8)
- [ ] `app-dev` e `app-prod` em `Synced` e `Healthy`, com promoção e rollback pelo Git (Bloco 9)
- [ ] CH-01 a CH-04 resolvidos

**Fechamento:** cada aluno conta, em até dois minutos, uma falha que resolveu e qual comando ou recurso foi decisivo. O instrutor encerra com a lista de perguntas da turma.

---

## Anexo A — Gabarito das perguntas de verificação

**Bloco 1**

1. O `oc` inclui todos os comandos do `kubectl` e acrescenta recursos do OpenShift (Projects, Routes, ImageStreams).
2. Use `--web` quando há SSO corporativo: ele abre o navegador e injeta o token automaticamente. Use `--token` em scripts e automações.
3. O projeto isola recursos, cotas, permissões e configurações entre equipes e ambientes.

**Bloco 2**

1. O Node é o objeto do Kubernetes que representa o nó. A Machine é o recurso do OpenShift que descreve a infraestrutura (física, virtual ou em nuvem).
2. O operador está em estado não saudável e precisa de investigação. Use `oc describe clusteroperator <nome>` e os logs do operador.
3. Porque mudanças no sistema operacional dos nós exigem reinício. O MCO drena as cargas, aplica a mudança e reinicia os nós de forma ordenada.

**Bloco 3**

1. O Deployment declara o estado desejado (réplicas, template) e gerencia os pods. O pod é a instância em execução.
2. O controlador do Deployment (por meio do ReplicaSet), que reconcilia o número de pods com as réplicas desejadas.
3. `ImagePullBackOff`: a imagem não é baixada (nome, tag ou permissão). Comece por `oc describe pod` e `oc get events`. `CrashLoopBackOff`: o container falha ao iniciar repetidamente. Comece por `oc logs --previous` e `oc describe pod`.

**Bloco 4**

1. O usuário representa uma pessoa. A ServiceAccount representa uma aplicação, um pod ou um pipeline.
2. `view` permite ler os recursos do projeto. `edit` permite criar e alterar a maioria dos recursos, mas não gerencia permissões.
3. O OpenShift injeta um UID da faixa do projeto e ignora a instrução de rodar como root. A SCC restritiva faz isso por padrão.
4. Quando a aplicação precisa realmente de privilégio, como um legado que escreve como root. A justificativa deve ser registrada, e a ServiceAccount deve ser dedicada e isolada.

**Bloco 5**

1. Quando a aplicação segue um builder suportado e você não quer manter um Dockerfile. A estratégia Docker é preferida quando você precisa de controle total das camadas.
2. A tag `latest` é um alias que pode apontar para outro conteúdo a qualquer momento. O digest identifica o conteúdo exato.
3. Quando a imagem muda na tag observada, o gatilho dispara um novo rollout do Deployment.
4. A estratégia está descontinuada. Para CI, use OpenShift Pipelines (Tekton).

**Bloco 6**

1. O Service dá um endereço interno estável e balanceia entre os pods. A Route expõe o Service para fora do cluster, com hostname e TLS.
2. O PV é o volume provisionado. O PVC é o pedido feito pela aplicação. A StorageClass provisiona o PV para atender o PVC.
3. Um template é um manifesto com parâmetros reaproveitável. O `oc process` gera os objetos a partir dele.

**Bloco 7**

1. `oc adm top` dá uma leitura rápida pela CLI. **Observe → Metrics** permite consultas PromQL e gráficos ao longo do tempo.
2. O must-gather coleta recursos, logs e métricas para o suporte da Red Hat analisar. O pacote é anexado ao chamado.
3. O evento `Normal` é informativo. O `Warning` indica algo que precisa de investigação.

**Bloco 8**

1. A Task é uma sequência de steps no mesmo pod. O Pipeline é um grafo de tasks. TaskRun e PipelineRun são as execuções concretas.
2. Por meio de results: a task produz um resultado (por exemplo, `IMAGE_DIGEST`), que é referenciado como `$(tasks.<nome>.results.<resultado>)` pela task seguinte ou pelo pipeline.
3. A ServiceAccount é o ponto de controle das credenciais. Os steps não recebem segredos em parâmetros nem nos logs. Isso segue o princípio do menor privilégio.
4. Porque o deploy por GitOps puxa as mudanças do Git. Assim o CI não precisa de credenciais de produção, reduz a superfície de ataque e a mudança fica registrada no Git.

**Bloco 9**

1. Porque o estado do cluster divergiu do Git (drift). O Argo CD compara o estado do cluster com o manifesto e marca a Application como `OutOfSync`.
2. O `prune` remove recursos que saíram do Git. O `selfHeal` reverte mudanças feitas diretamente no cluster.
3. Porque o Git é a fonte da verdade. A reversão manual conflita com o `selfHeal` e não fica registrada no histórico do Git.
4. Quando a aplicação não tolera duas versões ao mesmo tempo. O `Recreate` derruba todos os pods antes de subir a nova versão.
5. Na Route, o Service principal recebe `weight: 90` e o `alternateBackends` recebe `weight: 10` para o Service da nova versão.

---

## Anexo B — Preparação do instrutor

> Este anexo não deve ser entregue aos alunos.

### B.1 Pré-requisitos do cluster

- Cluster OpenShift 4.x (versão de referência 4.22).
- Operators **OpenShift Pipelines** e **OpenShift GitOps** instalados e com CSV em `Succeeded`.
- Uma **StorageClass padrão** disponível (`oc get storageclass`). Sem ela, os PVCs ficam `Pending` (Bloco 6 e CH-03).
- ImageStreams de builder disponíveis no namespace `openshift`, como `nodejs`, para o `oc new-app` do Bloco 5. Confirme com `oc get is -n openshift`.
- Acesso a um registry de imagens (interno ou externo) com credencial por aluno.
- Projetos por aluno: `<usuario>-dev` e `<usuario>-prod`, criados pelo aluno nos blocos 1 e 8.

### B.2 Repositórios de modelo

- **`lab-openshift-app`**: contém um `index.html` com o texto `versão 1` e um `Dockerfile` que parte de `registry.access.redhat.com/ubi9/httpd-24` e copia o `index.html` para `/var/www/html/`. A aplicação escuta na porta 8080.
- **`lab-openshift-config`**: contém `base/` com Deployment `lab-app` (container `lab-app`, porta 8080, label `app=lab-app`), Service `lab-app` e Route `lab-app`, além de `overlays/dev/` e `overlays/prod/` com `kustomization.yaml`.

### B.3 Incidentes do Desafio Final

Execute para cada aluno, no projeto `<usuario>-incidentes` (crie com `oc new-project <usuario>-incidentes`). Não deixe o projeto gerenciado pelo Argo CD.

**CH-01:**

```powershell
oc create deployment inc-app --image=registry.access.redhat.com/ubi9/httpd-24:versao-que-nao-existe -n <usuario>-incidentes
```

Solução de referência: `oc set image deployment/inc-app httpd-24=registry.access.redhat.com/ubi9/httpd-24:latest` (o nome do container é `httpd-24`; o aluno confirma com `oc get deploy inc-app -o jsonpath=...`).

**CH-02:**

```powershell
oc create deployment inc-web --image=registry.access.redhat.com/ubi9/httpd-24:latest -n <usuario>-incidentes
oc create service clusterip inc-svc --tcp=8080:8080 -n <usuario>-incidentes
oc expose service inc-svc -n <usuario>-incidentes
```

O Service `inc-svc` nasce com seletor `app=inc-svc`, e não há pods com essa label. Solução de referência: `oc set selector svc/inc-svc app=inc-web -n <usuario>-incidentes`.

**CH-03:** crie o PVC com uma StorageClass inexistente:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: inc-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: classe-que-nao-existe
  resources:
    requests:
      storage: 1Gi
```

Aplique com `oc apply -f inc-pvc.yaml -n <usuario>-incidentes`. Solução de referência: apague o PVC e recrie com uma StorageClass de `oc get storageclass`.

**CH-04:**

```powershell
oc create quota limite-pods --hard=pods=5 -n <usuario>-incidentes
oc create deployment inc-worker --image=registry.access.redhat.com/ubi9/ubi-minimal -n <usuario>-incidentes -- sleep infinity
oc scale deployment/inc-worker --replicas=6 -n <usuario>-incidentes
```

Com `inc-app` e `inc-web` ocupando 2 pods, o `inc-worker` pode ter no máximo 3 pods. Solução de referência: `oc scale deployment/inc-worker --replicas=3`.

**Ordem:** crie os recursos de CH-01 a CH-04 antes da aula, para verificar o comportamento de cada um. Depois, valide que a solução de referência funciona.

### B.4 Riscos e tempos

- `oc adm must-gather` pode levar vários minutos. Execute antes da aula e mostre o resultado.
- O `oc debug node` depende de permissão de cluster-admin. Mantenha a demonstração com o instrutor.
- Alinhe o nome do registry e os tokens de cada aluno antes do Bloco 8.
- Verifique os comandos deste documento em um cluster de teste antes da aula, especialmente `oc set serviceaccount`, `oc set selector` e as opções de `oc adm node-logs`.

---

## Anexo C — Solução de problemas comuns

| Sintoma | Causa provável | Ação |
|---|---|---|
| `oc login` retorna `Unauthorized` | Token expirado | Gere um novo token pelo console (**Copy login command**) |
| `oc new-app` falha com ImageStream não encontrada | Builder ausente no namespace `openshift` | Avise o instrutor (ver B.1) |
| PVC fica `Pending` | Sem StorageClass padrão, ou classe inexistente | `oc get storageclass` e recrie o PVC com uma classe válida |
| Route retorna 503 | Service sem endpoints (seletor incorreto ou pods não prontos) | `oc get endpoints <svc>` e `oc get pods --show-labels` |
| Pipeline falha no `buildah` por permissão | ServiceAccount `pipeline` sem a SCC esperada | Avise o instrutor; não altere SCC sozinho |
| `tkn` não encontra o recurso | Namespace errado | Use `-n <usuario>-dev` em todos os comandos `tkn` |
| Pipeline não acessa o registry | Credencial não vinculada à ServiceAccount `pipeline` | Refaça o `oc secrets link` do Bloco 8 |
| Application fica `OutOfSync` permanente | Campo mutável no cluster, ou falta de permissão no namespace | `argocd app diff`; use `ignoreDifferences` para campo mutável; confira o rótulo `managed-by` |
| Pods não são criados (`exceeded quota`) | Cota do projeto atingida | `oc describe resourcequota` e ajuste as réplicas |
| `argocd login` falha | Host incorreto ou SSO não configurado | Confira a Route `openshift-gitops-server` e use `--sso` |
| `Invoke-WebRequest` falha com erro de certificado | Certificado padrão do cluster emitido por CA interna | Use HTTP na Route de teste, ou aceite o certificado no navegador |

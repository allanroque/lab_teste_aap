# Documentação de Instalação AAP 2.7 - Containerizado HA Mínimo (2 Nós Híbridos + Banco Externo)

## Descrição do Ambiente

- **Sistema Operacional**: RHEL 9.6
- **CPU**: 8 vCPUs por nó AAP / 4 vCPUs no nó de banco
- **Memória**: 32GB por nó AAP / 16GB no nó de banco
- **Armazenamento**: 200GB por nó AAP / 100GB no nó de banco
- **Versão do AAP**: 2.7 Containerizado
- **Tipo de Instalação**: HA mínimo - 2 nós híbridos (todos os serviços replicados) + 1 nó de banco de dados dedicado
- **Banco de dados**: PostgreSQL **instalado e gerenciado pelo instalador do AAP** no nó dedicado (grupo `[database]`)

> **Importante**: esta é uma topologia de **laboratório** para simular alta disponibilidade com o menor número possível de máquinas. As topologias testadas e suportadas pela Red Hat estão em [Container topologies](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.7/html/tested_deployment_models/container-topologies). Para um Enterprise suportado são necessários no mínimo 3 nós no control plane (por conta do Redis em modo cluster) e um load balancer externo.

### O que é "híbrido" nesta topologia

Cada um dos dois nós executa os componentes do control plane ao mesmo tempo e também executa jobs (o Controller containerizado é sempre um nó híbrido, ou seja, control + execution no mesmo host):

Gateway | Controller | EDA | Ansible Lightspeed | Receptor | Performance Co-Pilot

Três serviços ficam em **nó único** (`aapha01`), por opção de projeto e não por limitação do instalador: **Automation Hub**, **Automation Metrics** e **Ansible MCP Server**. O detalhamento está em [Por que o Automation Hub fica em nó único](#por-que-o-automation-hub-fica-em-nó-único).

O PostgreSQL fica **fora** dos dois nós, no terceiro servidor, para que a perda de um nó do control plane não derrube o banco.

## Estrutura de Rede e Serviços

### Servidores e Funções

| Hostname | IP | Função | Descrição |
|----------|-----|--------|-----------|
| **aap.aroque.com.br** | 192.168.100.20 | VIP / Load Balancer | Nome público do AAP - balanceia 443 entre os dois nós híbridos |
| **aapha01.aroque.com.br** | 192.168.100.21 | Nó híbrido 1 | Gateway, Controller, EDA, Lightspeed, Receptor + Hub, Metrics e MCP (nó único) |
| **aapha02.aroque.com.br** | 192.168.100.22 | Nó híbrido 2 | Gateway, Controller, EDA, Lightspeed, Receptor |
| **aapdb01.aroque.com.br** | 192.168.100.25 | Database | PostgreSQL 15 implantado pelo instalador do AAP |

> **Nota sobre o Load Balancer**: o `gateway_main_url` aponta para o VIP. Em laboratório sem LB, é possível apontar `gateway_main_url` para um dos nós (ex.: `https://aapha01.aroque.com.br`), mas nesse caso não há failover da camada web - apenas do control plane. O LB deve balancear **TCP/443** com persistência de sessão (source IP) e health check em `/api/gateway/v1/status/`.

### Arquivo `/etc/hosts` dos Servidores

Todos os servidores devem ter o seguinte conteúdo no arquivo `/etc/hosts`:

```
127.0.0.1 localhost
::1 localhost
192.168.100.20 aap aap.aroque.com.br
192.168.100.21 aapha01 aapha01.aroque.com.br
192.168.100.22 aapha02 aapha02.aroque.com.br
192.168.100.25 aapdb01 aapdb01.aroque.com.br
```

## Por que o Automation Hub fica em nó único

O Automation Hub **pode** ser replicado nos dois nós, mas isso obriga a um storage compartilhado entre eles (NFS, S3 ou Azure Blob) para os artefatos de collections e imagens de container. Com `hub_storage_backend=file` em dois nós sem storage compartilhado, cada nó enxerga um conjunto diferente de artefatos - o upload feito em um nó simplesmente não existe no outro.

Como o objetivo aqui é simular HA com o **menor número possível de máquinas**, mantemos o Hub em `aapha01` com storage local e evitamos um servidor NFS adicional (que, aliás, seria mais um ponto único de falha). O mesmo raciocínio vale para o **Automation Metrics** e o **Ansible MCP Server**: são serviços de apoio, sem impacto na execução de automações se ficarem indisponíveis.

### O que isso significa na prática

A indisponibilidade de `aapha01` afeta apenas a distribuição de conteúdo (download de collections e execution environments a partir do Hub local). O **Gateway, o Controller e o EDA continuam ativos em `aapha02`** e as automações continuam executando normalmente, desde que os execution environments já estejam em cache no nó.

### Usando as capacidades do AAP na nuvem

Boa parte do que o Hub local entrega já está disponível como serviço gerenciado na Red Hat Hybrid Cloud Console, sem custo adicional para quem tem subscription do AAP - e é justamente por isso que um Hub local em nó único costuma ser suficiente:

| Capacidade | Serviço na nuvem | Endereço |
|------------|------------------|----------|
| Collections certificadas e validadas | Automation Hub (cloud) | https://console.redhat.com/ansible/automation-hub |
| Execution environments certificados | Registry Red Hat | `registry.redhat.io` |
| Métricas de uso, jobs e nós gerenciados | Automation Analytics / Automation Dashboard | https://console.redhat.com/ansible/automation-analytics |
| Catálogo de conteúdo e curadoria | Ansible automation hub | https://console.redhat.com/ansible |

Padrões de uso possíveis:

- **Hub local apenas como cache/proxy**: o Hub em `aapha01` sincroniza do Automation Hub da nuvem e serve de espelho interno. Se o nó cair, é possível apontar os projetos temporariamente para a nuvem.
- **Sem Hub local**: consumir collections e execution environments direto de `console.redhat.com` e `registry.redhat.io`, removendo o grupo `[automationhub]` do inventory. Exige saída para a internet a partir dos nós.
- **Analytics na nuvem**: com o `metrics_utility` habilitado e `METRICS_UTILITY_SHIP_TARGET` apontando para `controller_db` ou para a Red Hat, os relatórios de consumo sobem para o Automation Analytics, sem depender do Metrics local.

> **Se você precisar do Hub nos dois nós**, adicione `aapha02.aroque.com.br` ao grupo `[automationhub]` e configure um storage compartilhado:
>
> ```ini
> hub_storage_backend=file
> hub_shared_data_path=192.168.100.26:/exports/hub
> hub_shared_data_mount_opts=rw,sync,hard
> ```
>
> Alternativamente, use `hub_storage_backend=s3` ou `azure` com as respectivas variáveis de credencial e bucket/container.

## Configuração do Sistema Operacional

### Pré-requisitos Gerais

Todos os servidores devem ter:

- RHEL 9.6 instalado e **registrado no RHSM**
- SELinux configurado (`Enforcing`)
- Portas de firewall liberadas conforme documentação do AAP
- Usuário `ansible` criado com acesso sudo
- Acesso SSH habilitado (chave SSH do nó instalador distribuída para todos)

### Registro dos Servidores no RHSM

Em **todos** os servidores (dois nós híbridos e banco):

```bash
# Registrar a máquina na subscription Red Hat
sudo subscription-manager register --username XXXXXXXXXX --password 'redhat*99'

# Anexar/validar a subscription
sudo subscription-manager attach --auto
sudo subscription-manager status

# Habilitar apenas os repositórios necessários
sudo subscription-manager repos --disable="*"
sudo subscription-manager repos --enable=rhel-9-for-x86_64-baseos-rpms \
                                 --enable=rhel-9-for-x86_64-appstream-rpms

# Atualizar o sistema
sudo dnf update -y
```

No nó onde a instalação será executada (`aapha01`), instalar o `ansible-core`:

```bash
sudo dnf install -y ansible-core wget git rsync
ansible --version
```

### Configuração de Usuário e Acesso

Em **todos** os servidores:

```bash
# Criar usuário ansible
useradd ansible

# Definir senha (substituir pela senha desejada)
echo 'ansible:redhat*99' | chpasswd

# Configurar sudo
echo 'ansible ALL=(ALL) ALL' > /etc/sudoers.d/ansible && chmod 440 /etc/sudoers.d/ansible
```

> **Nota**: a instalação containerizada roda como usuário **não-root** (`ansible`). O `become` é usado apenas para as tarefas de preparação do host (pacotes, limites de kernel, subuid/subgid). Se preferir sudo sem senha, use `ansible ALL=(ALL) NOPASSWD: ALL` e remova o `--ask-become-pass` do comando de instalação.

Distribuir a chave SSH a partir do nó instalador (`aapha01`):

```bash
# Como usuário ansible em aapha01
ssh-keygen -t rsa -b 4096 -N '' -f ~/.ssh/id_rsa

ssh-copy-id ansible@aapha01.aroque.com.br
ssh-copy-id ansible@aapha02.aroque.com.br
ssh-copy-id ansible@aapdb01.aroque.com.br
```

## Banco de Dados

Nesta topologia **não criamos os bancos previamente**. O servidor `aapdb01.aroque.com.br` está declarado no grupo `[database]` do inventory e o próprio instalador do AAP:

1. Instala o PostgreSQL containerizado no nó;
2. Cria os usuários e os bancos de cada componente;
3. Aplica as extensões necessárias (`hstore`, `uuid-ossp` para o Hub);
4. Executa as migrações de schema.

O único requisito é informar as credenciais desejadas no inventory - o instalador se encarrega do resto.

### Mapeamento de Bancos de Dados

| Database | Componente | Usuário |
|----------|------------|---------|
| `gateway` | Automation Gateway | `gateway` |
| `awx` | Automation Controller | `awx` |
| `pulp` | Automation Hub | `pulp` |
| `eda` | Event-Driven Ansible | `eda` |
| `eda_event_persistence` | EDA - persistência de eventos | `eda_event_persistence` |
| `metrics_service` | Automation Metrics / Dashboard | `metrics_service` |
| `lightspeed` | Ansible Lightspeed | `lightspeed` |

> **Observação**: o usuário `ms_awx_readonly` também é criado pelo instalador - é o acesso somente-leitura que o Automation Metrics usa para ler o banco do Controller.

### Portas de Firewall

| Origem | Destino | Porta | Uso |
|--------|---------|-------|-----|
| Clientes / LB | aapha01, aapha02 | 443/tcp | Interface web e API do Gateway |
| aapha01, aapha02 | aapdb01 | 5432/tcp | PostgreSQL |
| aapha01 <-> aapha02 | - | 27199/tcp | Mesh Receptor entre os nós |
| aapha01 <-> aapha02 | - | 6379/tcp, 16379/tcp | Redis |
| Rede de monitoramento | aapha01, aapha02 | 44321/tcp, 44322/tcp | Performance Co-Pilot |

Liberação nos nós híbridos:

```bash
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --permanent --add-port=27199/tcp
sudo firewall-cmd --permanent --add-port=6379/tcp --add-port=16379/tcp
sudo firewall-cmd --permanent --add-port=44321/tcp --add-port=44322/tcp
sudo firewall-cmd --reload
sudo firewall-cmd --list-all
```

## Download do Instalador

Baixar o **Ansible Automation Platform 2.7 Containerized Setup** no [Red Hat Customer Portal](https://access.redhat.com/downloads/content/480) e extrair no nó instalador:

```bash
# Como usuário ansible em aapha01
tar -xzvf ansible-automation-platform-containerized-setup-2.7-x.tar.gz
cd ansible-automation-platform-containerized-setup-2.7-x
ls -l
```

## Arquivo de Inventory

Este é o arquivo de inventory para a topologia HA mínima com dois nós híbridos e banco de dados dedicado implantado pelo instalador:

inventory-ha

```ini
# =============================================================================
# AAP 2.7 - Containerizado HA Minimo
# 2 nos hibridos (todos os servicos) + 1 no de PostgreSQL dedicado
#
# Todos os componentes disponiveis no installer estao habilitados:
#   Gateway | Controller | Automation Hub | EDA | Automation Metrics (dashboard/
#   analytics) | Ansible Lightspeed (assistant) | Ansible MCP Server |
#   PostgreSQL | Redis | Receptor | Performance Co-Pilot | metrics-utility
#
# Docs:
# https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.7/html/containerized_installation
# =============================================================================

# AAP Gateway
# -----------------------------------------------------
[automationgateway]
aapha01.aroque.com.br ansible_host=192.168.100.21
aapha02.aroque.com.br ansible_host=192.168.100.22

# AAP Controller
# -----------------------------------------------------
[automationcontroller]
aapha01.aroque.com.br ansible_host=192.168.100.21
aapha02.aroque.com.br ansible_host=192.168.100.22

# AAP Automation Hub
# -----------------------------------------------------
# Nó único por opção de projeto: replicar o Hub exigiria storage compartilhado
# (NFS/S3/Azure) entre os dois nós. Ver a secao "Por que o Automation Hub fica
# em no unico" na documentacao.
[automationhub]
aapha01.aroque.com.br ansible_host=192.168.100.21

# AAP EDA Controller (Event-Driven Ansible)
# -----------------------------------------------------
[automationeda]
aapha01.aroque.com.br ansible_host=192.168.100.21
aapha02.aroque.com.br ansible_host=192.168.100.22

# AAP Automation Metrics Service (dashboard / analytics)
# -----------------------------------------------------
[automationmetrics]
aapha01.aroque.com.br ansible_host=192.168.100.21

# Ansible Lightspeed (assistant)
# -----------------------------------------------------
[ansiblelightspeed]
aapha01.aroque.com.br ansible_host=192.168.100.21
aapha02.aroque.com.br ansible_host=192.168.100.22

# Ansible MCP Server
# -----------------------------------------------------
[ansiblemcp]
aapha01.aroque.com.br ansible_host=192.168.100.21

# AAP database - o instalador cria o PostgreSQL, usuarios e bancos
# -----------------------------------------------------
[database]
aapdb01.aroque.com.br ansible_host=192.168.100.25

# Execution nodes: nao usados aqui (os dois controllers sao nos hibridos).
# Para adicionar execution/hop nodes remotos depois:
# [execution_nodes]
# exec1.aroque.com.br
# hop1.aroque.com.br receptor_type='hop'

[all:vars]
# =============================================================================
# Common
# =============================================================================
# https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.7/html/containerized_installation/appendix-inventory-files-vars#ref-general-inventory-variables
ansible_user=ansible
ansible_become=true
ansible_become_method=sudo

# Credenciais do registry (registry.redhat.io) - OBRIGATORIO PREENCHER
registry_username=XXXXXXXXXX
registry_password='redhat*99'
registry_url=registry.redhat.io
registry_ns_aap=ansible-automation-platform-27
registry_tls_verify=true

# Redis: com apenas 2 nos nao ha quorum para o modo cluster (minimo 3 nos),
# entao forcamos o modo standalone. Em producao com 3+ nos, remova esta linha
# para que o installer use redis_mode=cluster automaticamente.
redis_mode=standalone

# Performance Co-Pilot (monitoramento do control plane) - portas 44321/44322
setup_monitoring=true

# Tuning de kernel/limits do host para concorrencia
tune_host_limits=true

# Timeout HTTP do usuario final
client_request_timeout=30

# Automation Dashboard (novo no 2.7): a coleta/UI vem DESABILITADA por default.
# A flag e propagada para gateway/controller/metrics e cria as tabelas de
# dashboard_reports no banco metrics_service durante a migracao.
feature_flags={"FEATURE_DASHBOARD_COLLECTION_ENABLED": True}

# =============================================================================
# PostgreSQL (instalado pelo installer em aapdb01)
# =============================================================================
# https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.7/html/containerized_installation/appendix-inventory-files-vars#ref-database-variables
postgresql_admin_username=postgres
postgresql_admin_password='redhat*99'
postgresql_port=5432
postgresql_max_connections=1024
postgresql_password_encryption=scram-sha-256
postgresql_keep_databases=false

# =============================================================================
# AAP Gateway  -> https://aap.aroque.com.br (VIP do load balancer)
# =============================================================================
# https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.7/html/containerized_installation/appendix-inventory-files-vars#ref-gateway-variables
gateway_admin_user=admin
gateway_admin_password='redhat*99'
gateway_main_url=https://aap.aroque.com.br
gateway_pg_host=aapdb01.aroque.com.br
gateway_pg_database=gateway
gateway_pg_username=gateway
gateway_pg_password='redhat*99'
gateway_pg_port=5432
# Senha fixa do Redis: sem ela o instalador gera uma senha aleatoria a cada
# execucao, o que recria o container do gateway e pode falhar o start via systemd
gateway_redis_password=XXXXXXXXXX
gateway_uwsgi_processes=8
gateway_grpc_server_processes=5

# =============================================================================
# AAP Controller
# =============================================================================
# https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.7/html/containerized_installation/appendix-inventory-files-vars#ref-controller-variables
controller_admin_user=admin
controller_admin_password='redhat*99'
controller_pg_host=aapdb01.aroque.com.br
controller_pg_database=awx
controller_pg_username=awx
controller_pg_password='redhat*99'
controller_pg_port=5432
# Percentual de memoria do host reservado para capacidade de jobs
controller_percent_memory_capacity=0.5
controller_uwsgi_processes=8
controller_event_workers=8
controller_create_preload_data=true

# metrics-utility (coleta/relatorio de consumo para Automation Analytics).
metrics_utility_enabled=true
metrics_utility_cronjob_gather_schedule='*-*-* *:00:00'
metrics_utility_cronjob_report_schedule='*-*-02 00:00:00'
metrics_utility_extra_settings=[{"setting": "METRICS_UTILITY_SHIP_TARGET", "value": "directory"}, {"setting": "METRICS_UTILITY_SHIP_PATH", "value": "/var/lib/awx/metrics_utility"}, {"setting": "METRICS_UTILITY_REPORT_TYPE", "value": "CCSPv2"}, {"setting": "METRICS_UTILITY_PRICE_PER_NODE", "value": 11.55}, {"setting": "METRICS_UTILITY_REPORT_SKU", "value": "MCT3752MO"}, {"setting": "METRICS_UTILITY_REPORT_SKU_DESCRIPTION", "value": "Red Hat Ansible Automation Platform, Full Support (1 Managed Node, Dedicated, Monthly)"}, {"setting": "METRICS_UTILITY_REPORT_H1_HEADING", "value": "CCSP Reporting aroque.com.br: ANSIBLE Consumption"}, {"setting": "METRICS_UTILITY_REPORT_COMPANY_NAME", "value": "aroque.com.br"}, {"setting": "METRICS_UTILITY_REPORT_EMAIL", "value": "allanrafaelroque@gmail.com"}, {"setting": "METRICS_UTILITY_REPORT_RHN_LOGIN", "value": "allanrafaelroque"}, {"setting": "METRICS_UTILITY_REPORT_COMPANY_BUSINESS_LEADER", "value": "Allan Roque"}, {"setting": "METRICS_UTILITY_REPORT_COMPANY_PROCUREMENT_LEADER", "value": "Allan Roque"}]

# Subscription manifest (habilita a licenca automaticamente no install).
# Baixe em https://access.redhat.com/management/subscription_allocations
# controller_license_file=/home/ansible/manifest.zip

# Postinstall opcional (cria projects/JTs/credentials via as-code)
# controller_postinstall=true
# controller_postinstall_dir=/home/ansible/postinstall
# controller_postinstall_repo_url=https://github.com/<org>/<repo>.git

# =============================================================================
# AAP Automation Hub
# =============================================================================
# https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.7/html/containerized_installation/appendix-inventory-files-vars#ref-hub-variables
hub_admin_password='redhat*99'
hub_pg_host=aapdb01.aroque.com.br
hub_pg_database=pulp
hub_pg_username=pulp
hub_pg_password='redhat*99'
hub_pg_port=5432
hub_workers=2
hub_api_workers=4
hub_seed_collections=false

# Storage local: valido porque o Hub roda em um unico no. Se adicionar
# aapha02 ao grupo [automationhub], descomente o storage compartilhado:
hub_storage_backend=file
# hub_shared_data_path=192.168.100.26:/exports/hub
# hub_shared_data_mount_opts=rw,sync,hard

# Assinatura de collections e de containers (chave GPG gerada em
# /home/ansible/aap-signing/aap-signing-key.asc, passphrase = redhat*99)
hub_collection_signing=true
hub_collection_auto_sign=true
hub_collection_signing_key=/home/ansible/aap-signing/aap-signing-key.asc
hub_collection_signing_pass='redhat*99'
hub_collection_signing_service=ansible-default
hub_container_signing=true
hub_container_signing_key=/home/ansible/aap-signing/aap-signing-key.asc
hub_container_signing_pass='redhat*99'
hub_container_signing_service=container-default

# =============================================================================
# AAP EDA Controller (Event-Driven Ansible)
# =============================================================================
# https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.7/html/containerized_installation/appendix-inventory-files-vars#event-driven-ansible-controller
eda_admin_password='redhat*99'
eda_pg_host=aapdb01.aroque.com.br
eda_pg_database=eda
eda_pg_username=eda
eda_pg_password='redhat*99'
eda_pg_port=5432
eda_type=hybrid
eda_workers=2
eda_activation_workers=2
eda_gunicorn_workers=4

# Event streams (webhooks externos via gateway, com mTLS)
eda_event_stream_mtls=true
eda_event_stream_pg_username=eda_event_stream
eda_event_stream_pg_password='redhat*99'

# Persistencia de eventos (banco dedicado para historico de eventos)
eda_event_persistence_deploy_db=true
eda_event_persistence_pg_database=eda_event_persistence
eda_event_persistence_pg_username=eda_event_persistence
eda_event_persistence_pg_password='redhat*99'

# =============================================================================
# AAP Automation Metrics Service (dashboard / analytics)
# =============================================================================
# https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.7/html/containerized_installation/appendix-inventory-files-vars
automationmetrics_pg_host=aapdb01.aroque.com.br
automationmetrics_pg_database=metrics_service
automationmetrics_pg_username=metrics_service
automationmetrics_pg_password='redhat*99'
automationmetrics_pg_port=5432
# Acesso somente-leitura ao banco do Controller (usuario criado pelo installer)
automationmetrics_controller_db=awx
automationmetrics_controller_pg_username=ms_awx_readonly
automationmetrics_controller_read_pg_host=aapdb01.aroque.com.br
automationmetrics_controller_read_pg_password='redhat*99'
automationmetrics_controller_read_pg_port=5432
automationmetrics_gunicorn_workers=4
automationmetrics_dispatcherd_workers=2
automationmetrics_skip_install=false

# =============================================================================
# Ansible Lightspeed (assistant)
# =============================================================================
# https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.7/html/containerized_installation/appendix-inventory-files-vars#ref-lightspeed-variables
lightspeed_admin_user=admin
lightspeed_admin_password='redhat*99'
lightspeed_pg_host=aapdb01.aroque.com.br
lightspeed_pg_database=lightspeed
lightspeed_pg_username=lightspeed
lightspeed_pg_password='redhat*99'
lightspeed_pg_port=5432
lightspeed_uwsgi_processes=4

# --- Chatbot (requer um endpoint de modelo: RHOAI / OpenAI / Azure) ----------
# Sem estas 3 variaveis o Lightspeed sobe SEM o chatbot. Descomente e preencha
# quando tiver o endpoint do modelo (preflight exige url + api_key + model_id).
# lightspeed_chatbot_default_provider=rhoai
# lightspeed_chatbot_model_url=https://XXXXXXXXXX/v1
# lightspeed_chatbot_model_id=XXXXXXXXXX
# lightspeed_chatbot_model_api_key=XXXXXXXXXX
# lightspeed_chatbot_model_verify_ssl=true
#
# Ferramentas MCP do chatbot (exigem lightspeed_chatbot_model_url definido)
# lightspeed_mcp_controller_enabled=true
# lightspeed_mcp_lightspeed_enabled=true
#
# BYOK - base de conhecimento propria (carregue a imagem no podman antes)
# lightspeed_chatbot_byok_image=quay.io/<org>/byok-rag-content:latest
# lightspeed_chatbot_byok_score_multiplier=1.5

# =============================================================================
# Ansible MCP Server
# =============================================================================
# https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.7/html/containerized_installation/appendix-inventory-files-vars
mcp_public_base_url=https://aap.aroque.com.br
mcp_allow_write_operations=true
mcp_ignore_certificate_errors=false

# =============================================================================
# Receptor (mesh entre os dois nos hibridos)
# =============================================================================
# https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.7/html/containerized_installation/appendix-inventory-files-vars#ref-receptor-variables
receptor_port=27199
receptor_protocol=tcp
receptor_log_level=info
```

### Explicação das Variáveis do Inventory

#### Variáveis Comuns

- `ansible_user` / `ansible_become`: acesso SSH e escalonamento de privilégio para preparar os hosts
- `registry_username` / `registry_password`: credenciais do `registry.redhat.io`
- `registry_ns_aap`: namespace das imagens da versão 2.7 (`ansible-automation-platform-27`)
- `redis_mode`: **`standalone`** porque 2 nós não formam quorum de cluster Redis (mínimo 3)
- `setup_monitoring`: instala o Performance Co-Pilot para métricas do control plane
- `tune_host_limits`: ajusta limites de kernel/ulimits do host para concorrência
- `feature_flags`: habilita a coleta do novo **Automation Dashboard** do 2.7 (vem desabilitada por padrão)

#### Variáveis do PostgreSQL

- `postgresql_admin_username` / `postgresql_admin_password`: credenciais do superusuário que o instalador usa para criar bancos e roles
- `postgresql_max_connections`: somatório das conexões de todos os componentes dos dois nós - com muitos serviços replicados, 1024 é o piso recomendado
- `postgresql_password_encryption`: `scram-sha-256` (padrão do AAP 2.7)
- `postgresql_keep_databases`: se `false`, o uninstall remove os bancos

#### Variáveis do Automation Gateway

- `gateway_main_url`: **URL pública** - aponta para o VIP do load balancer, é o que aparece nos links da UI
- `gateway_pg_*`: conexão do Gateway com o banco externo
- `gateway_redis_password`: senha fixa do Redis (substitua `XXXXXXXXXX` por uma string aleatoria, ex.: `openssl rand -hex 24`) - sem ela o instalador gera uma nova a cada execução, recriando o container do gateway
- `gateway_uwsgi_processes` / `gateway_grpc_server_processes`: workers web e gRPC por nó

#### Variáveis do Automation Controller

- `controller_percent_memory_capacity`: percentual de memória do host reservado para capacidade de jobs (0.5 = 50%)
- `controller_uwsgi_processes` / `controller_event_workers`: workers de API e de processamento de eventos de job
- `controller_create_preload_data`: cria a organização e o inventário de demonstração
- `metrics_utility_*`: coleta e geração de relatórios de consumo (CCSPv2) para o Automation Analytics
- `controller_license_file`: aplica o manifest de subscription automaticamente durante a instalação

#### Variáveis do Automation Hub

- `hub_storage_backend=file`: armazenamento local de artefatos, válido porque o Hub roda em nó único. Com 2 nós de Hub, `hub_shared_data_path` (NFS) ou um backend `s3`/`azure` passa a ser obrigatório
- `hub_collection_signing` / `hub_container_signing`: assinatura GPG de collections e de imagens de container
- `hub_seed_collections`: se `true`, popula o Hub com as collections certificadas (demora bastante)

#### Variáveis do Event-Driven Ansible

- `eda_type=hybrid`: o nó EDA roda API e worker de ativação no mesmo host
- `eda_event_stream_mtls`: habilita event streams (webhooks externos) com autenticação mútua via Gateway
- `eda_event_persistence_deploy_db`: cria o banco dedicado ao histórico de eventos

#### Variáveis do Automation Metrics

- `automationmetrics_controller_pg_username=ms_awx_readonly`: usuário somente-leitura criado pelo instalador para o Metrics ler o banco do Controller
- `automationmetrics_gunicorn_workers` / `automationmetrics_dispatcherd_workers`: workers de API e de processamento assíncrono

#### Variáveis do Lightspeed e MCP

- `lightspeed_chatbot_model_url` / `_model_id` / `_model_api_key`: os três são obrigatórios para subir o chatbot - sem eles o Lightspeed sobe apenas com o assistente
- `mcp_public_base_url`: URL pública usada pelo MCP Server para montar os endpoints
- `mcp_allow_write_operations`: permite que clientes MCP executem operações de escrita no AAP

## Geração da Chave de Assinatura (GPG)

Necessária para `hub_collection_signing` e `hub_container_signing`. Executar no nó instalador:

```bash
mkdir -p /home/ansible/aap-signing && cd /home/ansible/aap-signing

cat > gpg-params <<'EOF'
%echo Gerando chave de assinatura AAP
Key-Type: RSA
Key-Length: 4096
Name-Real: AAP Signing Key
Name-Email: allanrafaelroque@gmail.com
Expire-Date: 0
Passphrase: redhat*99
%commit
%echo Pronto
EOF

gpg --batch --gen-key gpg-params
gpg --list-secret-keys --keyid-format=long
gpg --armor --export-secret-keys allanrafaelroque@gmail.com > aap-signing-key.asc
chmod 600 aap-signing-key.asc
```

## Executar a Instalação

A partir do diretório do instalador, no nó `aapha01`:

```bash
ansible-playbook -i inventory-ha ansible.containerized_installer.install -vv --ask-become-pass
```

O parâmetro `-vv` habilita modo verbose para acompanhar o progresso da instalação em detalhes. Se o sudo estiver como `NOPASSWD`, remova o `--ask-become-pass`.

> **Tempo estimado**: 60 a 90 minutos com todos os componentes habilitados nos dois nós.

## Testes de Validação

### Teste de Acesso SSH entre Servidores

```bash
ansible all -i inventory-ha -m ping
```

### Verificar os Containers em Cada Nó

```bash
# Como usuário ansible, em cada nó híbrido
podman ps --format "table {{.Names}}\t{{.Status}}"
systemctl --user list-units 'automation-*' --no-pager
```

### Teste de Conexão com o Banco de Dados

```bash
# A partir de qualquer nó híbrido
psql -h aapdb01.aroque.com.br -U awx -d awx -c "SELECT version();"
psql -h aapdb01.aroque.com.br -U gateway -d gateway -c "\l"
```

Listar todos os bancos criados pelo instalador:

```bash
# No nó aapdb01
sudo -u postgres psql -c "\l" | grep -E 'gateway|awx|pulp|eda|metrics_service|lightspeed'
```

### Validar o Status do Gateway e do Mesh

```bash
# Status geral da plataforma
curl -sk https://aap.aroque.com.br/api/gateway/v1/status/ | python3 -m json.tool

# Estado do mesh Receptor entre os dois nós
podman exec -it receptor receptorctl status
```

### Validar o Failover (simulação de HA)

```bash
# Derrubar os serviços do nó 2
ssh ansible@aapha02.aroque.com.br 'systemctl --user stop automation-*'

# A UI deve continuar respondendo pelo VIP, servida apenas pelo nó 1
curl -sk https://aap.aroque.com.br/api/gateway/v1/status/

# Restabelecer
ssh ansible@aapha02.aroque.com.br 'systemctl --user start automation-*'
```

### Acessos da Plataforma

| Serviço | URL | Usuário |
|---------|-----|---------|
| Gateway / Plataforma | https://aap.aroque.com.br | `admin` |
| Automation Controller | https://aap.aroque.com.br/execution/ | `admin` |
| Automation Hub | https://aap.aroque.com.br/hub/ | `admin` (servido por `aapha01`) |
| Event-Driven Ansible | https://aap.aroque.com.br/eda/ | `admin` |
| Automation Dashboard | https://aap.aroque.com.br/analytics/ | `admin` |
| Ansible MCP Server | https://aap.aroque.com.br/mcp/ | token do Gateway |

## Troubleshooting

### O preflight falha por falta de recursos

O instalador do 2.7 valida CPU e memória por componente. Com todos os serviços nos dois nós, respeite o mínimo de 8 vCPU / 32GB por nó. Para ignorar temporariamente em laboratório:

```bash
ansible-playbook -i inventory-ha ansible.containerized_installer.install -e ignore_preflight_errors=true -vv
```

### Containers do Gateway reiniciando após reinstalação

Ocorre quando `gateway_redis_password` não está fixa no inventory - cada execução gera uma nova senha e invalida o container já registrado no systemd. Mantenha a variável definida.

### Automation Hub indisponível após queda do aapha01

Comportamento esperado: o Hub roda em nó único. O Gateway e o Controller continuam respondendo por `aapha02`, mas o download de collections e execution environments a partir do Hub local falha. Enquanto o nó não volta, aponte os projetos e credentials de registry para a nuvem (`console.redhat.com/ansible/automation-hub` e `registry.redhat.io`).

Se adicionou o Hub nos dois nós e cada um mostra collections diferentes, o `hub_shared_data_path` não foi montado. Validar em ambos:

```bash
mount | grep exports/hub
```

### Coletar logs

```bash
ansible-playbook -i inventory-ha ansible.containerized_installer.collect_logs
```

### Desinstalar

```bash
ansible-playbook -i inventory-ha ansible.containerized_installer.uninstall
```

> Com `postgresql_keep_databases=false`, o uninstall **remove os bancos de dados**. Para preservá-los, altere para `true` antes de desinstalar.

## Referências

### Documentação Oficial Red Hat - AAP 2.7

| Documento | Descrição | Link |
|-----------|-----------|------|
| **Portal da documentação 2.7** | Índice de todos os guias da versão | https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.7 |
| **Planning your installation** | Requisitos de sistema, topologias e decisões de arquitetura antes de instalar | https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.7/html/planning_your_installation |
| **Containerized installation** | Guia principal da instalação containerizada (o método usado nesta documentação) | https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.7/html/containerized_installation |
| **Appendix: Inventory file variables** | Referência completa de **todas** as variáveis do inventory, por componente | https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.7/html/containerized_installation/appendix-inventory-files-vars |
| **Tested deployment models - Container topologies** | Topologias testadas e suportadas pela Red Hat, com o hardware homologado | https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.7/html/tested_deployment_models/container-topologies |
| **Release notes 2.7** | Novidades da versão, recursos descontinuados e problemas conhecidos | https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.7/html/release_notes |
| **RPM installation** | Método alternativo de instalação, via RPM | https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.7/html/rpm_installation |

### Referências por Componente do Inventory

| Componente | Seção do Appendix |
|------------|-------------------|
| Variáveis gerais | https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.7/html/containerized_installation/appendix-inventory-files-vars#ref-general-inventory-variables |
| PostgreSQL | https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.7/html/containerized_installation/appendix-inventory-files-vars#ref-database-variables |
| Automation Gateway | https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.7/html/containerized_installation/appendix-inventory-files-vars#ref-gateway-variables |
| Automation Controller | https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.7/html/containerized_installation/appendix-inventory-files-vars#ref-controller-variables |
| Automation Hub | https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.7/html/containerized_installation/appendix-inventory-files-vars#ref-hub-variables |
| Event-Driven Ansible | https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.7/html/containerized_installation/appendix-inventory-files-vars#event-driven-ansible-controller |
| Ansible Lightspeed | https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.7/html/containerized_installation/appendix-inventory-files-vars#ref-lightspeed-variables |
| Receptor | https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.7/html/containerized_installation/appendix-inventory-files-vars#ref-receptor-variables |

> **Nota**: as âncoras (`#ref-...`) podem mudar entre releases da documentação. Se um link direto não abrir na seção esperada, use o Appendix completo e navegue pelo índice lateral.

### Portal do Cliente e Console

| Recurso | Descrição | Link |
|---------|-----------|------|
| **Download do AAP 2.7 Containerized Setup** | Pacote do instalador | https://access.redhat.com/downloads/content/480 |
| **Subscription allocations** | Geração do manifest de subscription (`controller_license_file`) | https://access.redhat.com/management/subscription_allocations |
| **Ansible Automation Hub (cloud)** | Collections certificadas e validadas | https://console.redhat.com/ansible/automation-hub |
| **Automation Analytics** | Métricas de uso, jobs e nós gerenciados | https://console.redhat.com/ansible/automation-analytics |
| **Red Hat Ecosystem Catalog** | Imagens de execution environments certificadas | https://catalog.redhat.com/software/containers/search |
| **Knowledgebase / Suporte** | Artigos, soluções e abertura de casos | https://access.redhat.com/support |

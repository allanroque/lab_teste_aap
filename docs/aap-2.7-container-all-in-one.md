# Documentação de Instalação AAP 2.7 - Containerizado All-in-One (Growth)

## Descrição do Ambiente

- **Sistema Operacional**: RHEL 9.6
- **CPU**: 6 vCPUs
- **Memória**: 17GB
- **Armazenamento**: 150GB
- **Versão do AAP**: 2.7 Containerizado
- **Tipo de Instalação**: All-in-One (topologia *Growth*) - todos os serviços, incluindo o PostgreSQL, em um único nó
- **Banco de dados**: PostgreSQL **instalado e gerenciado pelo instalador do AAP** no próprio nó

> **Importante**: a topologia *Growth* é indicada para laboratórios, provas de conceito e ambientes pequenos. Não há redundância - a perda do nó derruba toda a plataforma. Para ambientes com necessidade de HA, consulte a [Instalação AAP 2.7 HA Mínimo](aap-2.7-container-ha-minimo-2-nodes-db-externo.md).

### Componentes Habilitados

Todos os componentes disponíveis no instalador 2.7 estão habilitados neste ambiente:

Gateway | Controller | Automation Hub | EDA | Automation Metrics (dashboard/analytics) | Ansible Lightspeed (assistant) | Ansible MCP Server | PostgreSQL | Redis | Receptor | Performance Co-Pilot | metrics-utility

## Estrutura de Rede e Serviços

### Servidores e Funções

| Hostname | IP | Função | Descrição |
|----------|-----|--------|-----------|
| **aap01.aroque.com.br** | 192.168.100.11 | All-in-One | Gateway, Controller, Hub, EDA, Metrics, Lightspeed, MCP, PostgreSQL, Redis, Receptor |

### Arquivo `/etc/hosts` do Servidor

```
127.0.0.1 localhost
::1 localhost
192.168.100.11 aap01 aap01.aroque.com.br
```

> **Nota**: como a instalação usa `ansible_connection=local`, o nome `aap01.aroque.com.br` precisa resolver para o próprio host - daí a entrada no `/etc/hosts`.

## Configuração do Sistema Operacional

### Pré-requisitos Gerais

O servidor deve ter:

- RHEL 9.6 instalado e **registrado no RHSM**
- SELinux configurado (`Enforcing`)
- Portas de firewall liberadas conforme documentação do AAP
- Usuário `ansible` criado com acesso sudo
- Acesso SSH habilitado

### Registro do Servidor no RHSM

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

Instalar o `ansible-core` e utilitários:

```bash
sudo dnf install -y ansible-core wget git rsync
ansible --version
```

### Configuração de Usuário e Acesso

```bash
# Criar usuário ansible
useradd ansible

# Definir senha (substituir pela senha desejada)
echo 'ansible:redhat*99' | chpasswd

# Configurar sudo
echo 'ansible ALL=(ALL) ALL' > /etc/sudoers.d/ansible && chmod 440 /etc/sudoers.d/ansible
```

> **Nota**: a instalação containerizada roda como usuário **não-root** (`ansible`), usando Podman rootless. O `become` é usado apenas para as tarefas de preparação do host (pacotes, limites de kernel, subuid/subgid). Toda a instalação é executada logada como `ansible`.

### Portas de Firewall

| Origem | Porta | Uso |
|--------|-------|-----|
| Clientes | 443/tcp | Interface web e API do Gateway |
| Execution/hop nodes remotos | 27199/tcp | Mesh Receptor |
| Rede de monitoramento | 44321/tcp, 44322/tcp | Performance Co-Pilot |

```bash
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --permanent --add-port=27199/tcp
sudo firewall-cmd --permanent --add-port=44321/tcp --add-port=44322/tcp
sudo firewall-cmd --reload
sudo firewall-cmd --list-all
```

> O PostgreSQL (5432) e o Redis (6379) não precisam de liberação externa - o tráfego é local ao host.

## Banco de Dados

Nesta topologia **não criamos os bancos previamente**. O próprio host está declarado no grupo `[database]` do inventory e o instalador do AAP:

1. Instala o PostgreSQL containerizado no nó;
2. Cria os usuários e os bancos de cada componente;
3. Aplica as extensões necessárias (`hstore`, `uuid-ossp` para o Hub);
4. Executa as migrações de schema.

O único requisito é informar as credenciais desejadas no inventory - o instalador se encarrega do resto.

### Mapeamento de Bancos de Dados

Os nomes de database/usuário seguem o padrão da documentação Red Hat:

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

## Download do Instalador

Baixar o **Ansible Automation Platform 2.7 Containerized Setup** no [Red Hat Customer Portal](https://access.redhat.com/downloads/content/480) e extrair no host:

```bash
# Como usuário ansible
tar -xzvf ansible-automation-platform-containerized-setup-2.7-x.tar.gz
cd ansible-automation-platform-containerized-setup-2.7-x
ls -l
```

## Arquivo de Inventory

Este é o arquivo de inventory para a topologia all-in-one com todos os componentes habilitados:

inventory-growth

```ini
# =============================================================================
# AAP 2.7 - Containerized "Growth" (All-in-One) topology
# Host: aap01.aroque.com.br (192.168.100.11) - RHEL 9.6 / 6 vCPU / 17 GB RAM
#
# Todos os componentes disponiveis no installer estao habilitados:
#   Gateway | Controller | Automation Hub | EDA | Automation Metrics (dashboard/
#   analytics) | Ansible Lightspeed (assistant) | Ansible MCP Server |
#   PostgreSQL | Redis | Receptor | Performance Co-Pilot | metrics-utility
#
# Nomes de database/usuario seguem o padrao da documentacao Red Hat:
#   gateway/gateway | awx/awx | pulp/pulp | eda/eda |
#   metrics_service/metrics_service | lightspeed/lightspeed
#
# Docs:
# https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.7/html/containerized_installation
# =============================================================================

# AAP Gateway
# -----------------------------------------------------
[automationgateway]
aap01.aroque.com.br

# AAP Controller
# -----------------------------------------------------
[automationcontroller]
aap01.aroque.com.br

# AAP Automation Hub
# -----------------------------------------------------
[automationhub]
aap01.aroque.com.br

# AAP EDA Controller (Event-Driven Ansible)
# -----------------------------------------------------
[automationeda]
aap01.aroque.com.br

# AAP Automation Metrics Service (dashboard / analytics)
# -----------------------------------------------------
[automationmetrics]
aap01.aroque.com.br

# Ansible Lightspeed (assistant)
# -----------------------------------------------------
[ansiblelightspeed]
aap01.aroque.com.br

# Ansible MCP Server
# -----------------------------------------------------
[ansiblemcp]
aap01.aroque.com.br

# AAP database - o instalador cria o PostgreSQL, usuarios e bancos
# -----------------------------------------------------
[database]
aap01.aroque.com.br

# Execution nodes: nao usados no all-in-one (o controller e um no hibrido).
# Para adicionar execution/hop nodes remotos depois:
# [execution_nodes]
# exec1.aroque.com.br
# hop1.aroque.com.br receptor_type='hop'

[all:vars]
# =============================================================================
# Common
# =============================================================================
# https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.7/html/containerized_installation/appendix-inventory-files-vars#ref-general-inventory-variables
ansible_connection=local

# Credenciais do registry (registry.redhat.io) - OBRIGATORIO PREENCHER
registry_username=XXXXXXXXXX
registry_password='redhat*99'
registry_url=registry.redhat.io
registry_ns_aap=ansible-automation-platform-27
registry_tls_verify=true

# Redis em modo standalone (obrigatorio para topologia all-in-one)
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
# PostgreSQL (interno)
# =============================================================================
# https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.7/html/containerized_installation/appendix-inventory-files-vars#ref-database-variables
postgresql_admin_username=postgres
postgresql_admin_password='redhat*99'
postgresql_port=5432
postgresql_max_connections=1024
postgresql_password_encryption=scram-sha-256
postgresql_keep_databases=false

# =============================================================================
# AAP Gateway  -> https://aap01.aroque.com.br
# =============================================================================
# https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.7/html/containerized_installation/appendix-inventory-files-vars#ref-gateway-variables
gateway_admin_user=admin
gateway_admin_password='redhat*99'
gateway_main_url=https://aap01.aroque.com.br
gateway_pg_host=aap01.aroque.com.br
gateway_pg_database=gateway
gateway_pg_username=gateway
gateway_pg_password='redhat*99'
gateway_pg_port=5432
# Senha fixa do Redis: sem ela o instalador gera uma senha aleatoria a cada
# execucao, o que recria o container do gateway e pode falhar o start via systemd
gateway_redis_password=XXXXXXXXXX
# All-in-one: limita workers (default seria 2*CPU+1 = 13 por servico)
gateway_uwsgi_processes=4
gateway_grpc_server_processes=3

# =============================================================================
# AAP Controller
# =============================================================================
# https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.7/html/containerized_installation/appendix-inventory-files-vars#ref-controller-variables
controller_admin_user=admin
controller_admin_password='redhat*99'
controller_pg_host=aap01.aroque.com.br
controller_pg_database=awx
controller_pg_username=awx
controller_pg_password='redhat*99'
controller_pg_port=5432
# Percentual de memoria do host reservado para capacidade de jobs
controller_percent_memory_capacity=0.4
controller_uwsgi_processes=4
controller_event_workers=4
controller_create_preload_data=true

# metrics-utility (coleta/relatorio de consumo para Automation Analytics).
# METRICS_UTILITY_PRICE_PER_NODE, REPORT_COMPANY_NAME, REPORT_EMAIL, REPORT_SKU
# e REPORT_TYPE sao obrigatorios quando metrics_utility_enabled=true.
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
hub_pg_host=aap01.aroque.com.br
hub_pg_database=pulp
hub_pg_username=pulp
hub_pg_password='redhat*99'
hub_pg_port=5432
hub_storage_backend=file
hub_workers=2
hub_api_workers=4
hub_seed_collections=false

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
eda_pg_host=aap01.aroque.com.br
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
automationmetrics_pg_host=aap01.aroque.com.br
automationmetrics_pg_database=metrics_service
automationmetrics_pg_username=metrics_service
automationmetrics_pg_password='redhat*99'
automationmetrics_pg_port=5432
# Acesso somente-leitura ao banco do Controller (usuario criado pelo installer)
automationmetrics_controller_db=awx
automationmetrics_controller_pg_username=ms_awx_readonly
automationmetrics_controller_read_pg_host=aap01.aroque.com.br
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
lightspeed_pg_host=aap01.aroque.com.br
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
# BYOK - base de conhecimento propria (carregue a imagem no podman antes)
# lightspeed_chatbot_byok_image=quay.io/<org>/byok-rag-content:latest
# lightspeed_chatbot_byok_score_multiplier=1.5

# --- Chatbot via provider OpenAI (exemplo em uso neste laboratorio) ----------
lightspeed_chatbot_default_provider=openai
lightspeed_chatbot_model_url=https://api.openai.com/v1
lightspeed_chatbot_model_id=gpt-4o-mini
lightspeed_chatbot_model_api_key=XXXXXXXXXX
lightspeed_chatbot_model_extra_settings={}

# Ferramentas MCP do chatbot (exigem lightspeed_chatbot_model_url definido)
lightspeed_mcp_controller_enabled=true
lightspeed_mcp_lightspeed_enabled=true

# =============================================================================
# Ansible MCP Server
# =============================================================================
# https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.7/html/containerized_installation/appendix-inventory-files-vars
mcp_public_base_url=https://aap01.aroque.com.br
mcp_allow_write_operations=true
mcp_ignore_certificate_errors=false

# =============================================================================
# Receptor
# =============================================================================
# https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.7/html/containerized_installation/appendix-inventory-files-vars#ref-receptor-variables
receptor_port=27199
receptor_protocol=tcp
receptor_log_level=info
```

### Explicação das Variáveis do Inventory

#### Variáveis Comuns

- `ansible_connection=local`: a instalação é executada no próprio host, sem SSH
- `registry_username` / `registry_password`: credenciais do `registry.redhat.io`
- `registry_ns_aap`: namespace das imagens da versão 2.7 (`ansible-automation-platform-27`)
- `redis_mode=standalone`: **obrigatório** no all-in-one - o modo cluster exige no mínimo 3 nós
- `setup_monitoring`: instala o Performance Co-Pilot para métricas do control plane
- `tune_host_limits`: ajusta limites de kernel/ulimits do host - importante quando todos os serviços dividem o mesmo host
- `client_request_timeout`: timeout HTTP do usuário final
- `feature_flags`: habilita a coleta do novo **Automation Dashboard** do 2.7 (vem desabilitada por padrão) e cria as tabelas `dashboard_reports` no banco `metrics_service` durante a migração

#### Variáveis do PostgreSQL

- `postgresql_admin_username` / `postgresql_admin_password`: credenciais do superusuário que o instalador usa para criar bancos e roles
- `postgresql_max_connections=1024`: com todos os componentes no mesmo host, o somatório de conexões é alto - 1024 é o piso recomendado
- `postgresql_password_encryption`: `scram-sha-256` (padrão do AAP 2.7)
- `postgresql_keep_databases`: se `false`, o uninstall remove os bancos

#### Variáveis do Automation Gateway

- `gateway_main_url`: URL pública da plataforma - é o que aparece nos links da UI
- `gateway_pg_*`: conexão do Gateway com o banco
- `gateway_redis_password`: senha fixa do Redis (substitua `XXXXXXXXXX` por uma string aleatoria, ex.: `openssl rand -hex 24`) - sem ela o instalador gera uma nova a cada execução, recriando o container do gateway e podendo falhar o start via systemd
- `gateway_uwsgi_processes` / `gateway_grpc_server_processes`: **limitados manualmente** - o default seria `2*CPU+1` (13 processos) por serviço, o que estouraria a memória do host

#### Variáveis do Automation Controller

- `controller_percent_memory_capacity=0.4`: apenas 40% da memória é reservada para capacidade de jobs, porque os demais serviços dividem o mesmo host
- `controller_uwsgi_processes` / `controller_event_workers`: workers de API e de processamento de eventos de job, também reduzidos
- `controller_create_preload_data`: cria a organização e o inventário de demonstração
- `metrics_utility_*`: coleta e geração de relatórios de consumo (CCSPv2) para o Automation Analytics. `METRICS_UTILITY_PRICE_PER_NODE`, `REPORT_COMPANY_NAME`, `REPORT_EMAIL`, `REPORT_SKU` e `REPORT_TYPE` são obrigatórios quando `metrics_utility_enabled=true`
- `controller_license_file`: aplica o manifest de subscription automaticamente durante a instalação
- `controller_postinstall*`: opcional - cria projects, job templates e credentials via *configuration as code*

#### Variáveis do Automation Hub

- `hub_storage_backend=file`: armazenamento local de artefatos (suficiente em nó único)
- `hub_workers` / `hub_api_workers`: workers de conteúdo e de API
- `hub_collection_signing` / `hub_container_signing`: assinatura GPG de collections e de imagens de container
- `hub_seed_collections=false`: não popula o Hub com as collections certificadas (o seed demora bastante e consome disco)

#### Variáveis do Event-Driven Ansible

- `eda_type=hybrid`: API e worker de ativação no mesmo host
- `eda_event_stream_mtls`: habilita event streams (webhooks externos) com autenticação mútua via Gateway
- `eda_event_persistence_deploy_db`: cria o banco dedicado ao histórico de eventos

#### Variáveis do Automation Metrics

- `automationmetrics_controller_pg_username=ms_awx_readonly`: usuário somente-leitura criado pelo instalador para o Metrics ler o banco do Controller
- `automationmetrics_gunicorn_workers` / `automationmetrics_dispatcherd_workers`: workers de API e de processamento assíncrono
- `automationmetrics_skip_install=false`: instala o serviço (se `true`, apenas prepara o banco)

#### Variáveis do Lightspeed e MCP

- `lightspeed_chatbot_model_url` / `_model_id` / `_model_api_key`: os três são obrigatórios para subir o chatbot - sem eles o Lightspeed sobe apenas com o assistente. O preflight valida a presença dos três
- `lightspeed_chatbot_default_provider`: `rhoai`, `openai` ou `azure`
- `lightspeed_mcp_controller_enabled` / `lightspeed_mcp_lightspeed_enabled`: ferramentas MCP do chatbot - exigem `lightspeed_chatbot_model_url` definido
- `mcp_public_base_url`: URL pública usada pelo MCP Server para montar os endpoints
- `mcp_allow_write_operations`: permite que clientes MCP executem operações de escrita no AAP

> **Atenção com segredos**: as chaves de API (`lightspeed_chatbot_model_api_key`), senhas do registry e senhas de banco estão em texto plano no inventory. **Não versione este arquivo com valores reais** - use `ansible-vault encrypt` ou mantenha o inventory fora do repositório.

## Geração da Chave de Assinatura (GPG)

Necessária para `hub_collection_signing` e `hub_container_signing`:

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

A partir do diretório do instalador, logado como `ansible`:

```bash
ansible-playbook -i inventory-growth ansible.containerized_installer.install -vv --ask-become-pass
```

O parâmetro `-vv` habilita modo verbose para acompanhar o progresso da instalação em detalhes. Se o sudo estiver como `NOPASSWD`, remova o `--ask-become-pass`.

> **Tempo estimado**: 45 a 75 minutos com todos os componentes habilitados.

## Testes de Validação

### Verificar os Containers

```bash
podman ps --format "table {{.Names}}\t{{.Status}}"
systemctl --user list-units 'automation-*' --no-pager
```

Serviços esperados: `automation-gateway`, `automation-controller-*`, `automation-hub-*`, `automation-eda-*`, `automation-metrics-*`, `lightspeed-*`, `ansible-mcp`, `postgresql`, `redis`, `receptor`.

### Validar os Bancos Criados pelo Instalador

```bash
podman exec -it postgresql psql -U postgres -c "\l" | grep -E 'gateway|awx|pulp|eda|metrics_service|lightspeed'
```

Validar as extensões do Hub:

```bash
podman exec -it postgresql psql -U postgres -d pulp -c "\dx"
```

### Validar o Status do Gateway

```bash
curl -sk https://aap01.aroque.com.br/api/gateway/v1/status/ | python3 -m json.tool
```

### Validar o Receptor

```bash
podman exec -it receptor receptorctl status
```

### Validar o Performance Co-Pilot

```bash
sudo systemctl status pmcd pmlogger
pcp
```

### Validar o metrics-utility

```bash
systemctl --user list-timers 'metrics-utility*' --no-pager
ls -l /var/lib/awx/metrics_utility
```

### Acessos da Plataforma

| Serviço | URL | Usuário |
|---------|-----|---------|
| Gateway / Plataforma | https://aap01.aroque.com.br | `admin` |
| Automation Controller | https://aap01.aroque.com.br/execution/ | `admin` |
| Automation Hub | https://aap01.aroque.com.br/hub/ | `admin` |
| Event-Driven Ansible | https://aap01.aroque.com.br/eda/ | `admin` |
| Automation Dashboard | https://aap01.aroque.com.br/analytics/ | `admin` |
| Ansible Lightspeed | https://aap01.aroque.com.br/lightspeed/ | `admin` |
| Ansible MCP Server | https://aap01.aroque.com.br/mcp/ | token do Gateway |

## Troubleshooting

### O preflight falha por falta de recursos

O instalador do 2.7 valida CPU e memória por componente, e a soma de todos os componentes no all-in-one é agressiva para 6 vCPU / 17GB. Para ignorar em laboratório:

```bash
ansible-playbook -i inventory-growth ansible.containerized_installer.install -e ignore_preflight_errors=true -vv
```

### Host ficando sem memória / containers sendo mortos pelo OOM killer

Reduza os workers no inventory - são as variáveis que mais pesam:

```ini
gateway_uwsgi_processes=2
gateway_grpc_server_processes=2
controller_uwsgi_processes=2
controller_event_workers=2
controller_percent_memory_capacity=0.3
automationmetrics_gunicorn_workers=2
lightspeed_uwsgi_processes=2
```

### Containers do Gateway reiniciando após reinstalação

Ocorre quando `gateway_redis_password` não está fixa no inventory - cada execução gera uma nova senha e invalida o container já registrado no systemd. Mantenha a variável definida.

### Lightspeed sobe sem o chatbot

O chatbot só é implantado quando `lightspeed_chatbot_model_url`, `lightspeed_chatbot_model_id` e `lightspeed_chatbot_model_api_key` estão as três definidas. Validar:

```bash
podman ps | grep lightspeed
podman logs lightspeed-chatbot
```

### Coletar logs

```bash
ansible-playbook -i inventory-growth ansible.containerized_installer.collect_logs
```

### Desinstalar

```bash
ansible-playbook -i inventory-growth ansible.containerized_installer.uninstall
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

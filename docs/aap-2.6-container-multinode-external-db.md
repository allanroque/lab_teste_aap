# Documentação de Instalação AAP 2.6 - Containerizado Multi-Node com Banco de Dados Externo

## Descrição do Ambiente

- **Sistema Operacional**: RHEL 9
- **CPU**: 4 vCPUs por servidor
- **Memória**: 16GB por servidor
- **Armazenamento**: 80GB por servidor
- **Versão do AAP**: 2.6 Containerizado
- **Tipo de Instalação**: Multi-Node com Banco de Dados Externo

## Estrutura de Rede e Serviços

### Servidores e Funções

| Hostname | IP | Função | Descrição |
|----------|-----|--------|-----------|
| **labac01.aroque.com.br** | 192.168.100.51 | Controller e Gateway | Nó 1 do cluster de Automation Controller e Automation Gateway |
| **labac02.aroque.com.br** | 192.168.100.52 | Controller e Gateway | Nó 2 do cluster de Automation Controller e Automation Gateway |
| **labac03.aroque.com.br** | 192.168.100.53 | Controller e Gateway | Nó 3 do cluster de Automation Controller e Automation Gateway |
| **labdb01.aroque.com.br** | 192.168.100.55 | Database Externo | Servidor PostgreSQL 15 com suporte ICU para todos os bancos do AAP |
| **labhub01.aroque.com.br** | 192.168.100.56 | Private Hub | Servidor Automation Hub (Private Automation Hub) |
| **labeda01.aroque.com.br** | 192.168.100.57 | Event-Driven Ansible | Servidor Event-Driven Ansible Controller |

### Arquivo `/etc/hosts` dos Servidores

Todos os servidores devem ter o seguinte conteúdo no arquivo `/etc/hosts`:

```
127.0.0.1 localhost
::1 localhost
192.168.100.51 labac01 labac01.aroque.com.br
192.168.100.52 labac02 labac02.aroque.com.br
192.168.100.53 labac03 labac03.aroque.com.br
192.168.100.55 labdb01 labdb01.aroque.com.br
192.168.100.56 labhub01 labhub01.aroque.com.br
192.168.100.57 labeda01 labeda01.aroque.com.br
```

## Configuração do Banco de Dados Externo

### Visão Geral

Esta implementação utiliza um banco de dados PostgreSQL externo (customer provided) para todos os componentes do AAP. O banco de dados é gerenciado pelo cliente e não pelo instalador do AAP.

**Referência Oficial**: [Configuring an external (customer provided) PostgreSQL database](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.6/html-single/containerized_installation/index#setup-ext-db-with-admin-creds)

### Requisitos do Banco de Dados

- **Versão PostgreSQL**: 15, 16 ou 17
- **Suporte ICU**: Obrigatório para bancos externos (International Components for Unicode)
- **Encoding**: UTF8
- **Locale**: en_US.utf8
- **Backup e Restore**: Para PostgreSQL 16 ou 17, é necessário usar processos externos de backup/restore. O backup/restore automático do AAP embora funcione em diferentes versões do PostgreSQL esta neste momento testado e homologado para o PostgreSQL 15.

### Instalação e Configuração do PostgreSQL

> **Nota**: Geralmente a configuração do banco de dados é realizada por um DBA, mas aqui estão os passos mínimos necessários para o AAP funcionar com banco externo.

#### 1. Habilitar Módulo PostgreSQL 15

```bash
sudo dnf module reset postgresql -y
sudo dnf module enable postgresql:15 -y
```

#### 2. Instalar PostgreSQL e Dependências

```bash
sudo dnf install -y postgresql-server postgresql-contrib libicu
```

#### 3. Inicializar o PostgreSQL

```bash
sudo postgresql-setup --initdb
```

Saída esperada:
```
 * Initializing database in '/var/lib/pgsql/data'
 * Initialized, logs are in /var/lib/pgsql/initdb_postgresql.log
```

#### 4. Iniciar e Habilitar PostgreSQL

```bash
sudo systemctl enable postgresql
sudo systemctl start postgresql
sudo systemctl status postgresql
```

#### 5. Validar Configuração ICU e Encoding

Verificar se o cluster foi criado com ICU:

```bash
sudo -u postgres psql -c "SHOW lc_collate;"
```

> **OBS**: Pode ser que essa verificação falhe, não tem problema! O importante e ter o libicu implementado para viabilizar ações futuras.

Validar encoding e locale:

```bash
sudo -u postgres psql -c "SHOW server_encoding;"
sudo -u postgres psql -c "SHOW lc_collate;"
sudo -u postgres psql -c "SHOW lc_ctype;"
```

Valores esperados:
- `server_encoding`: UTF8
- `lc_collate`: en_US.utf8
- `lc_ctype`: en_US.utf8

#### 6. Configurar PostgreSQL (postgresql.conf)

Editar o arquivo de configuração:

```bash
sudo vi /var/lib/pgsql/data/postgresql.conf
```

Ajustes mínimos recomendados:

```ini
listen_addresses = '*'
max_connections = 500
shared_buffers = 1GB
work_mem = 16MB
maintenance_work_mem = 256MB
password_encryption = scram-sha-256
```

> **Importante**: Alguns parâmetros variam de acordo com CPU e memória. É recomendado validar e ajustar parâmetros com o time de DBA.

#### 7. Configurar Autenticação (pg_hba.conf)

Editar o arquivo de autenticação:

```bash
sudo vi /var/lib/pgsql/data/pg_hba.conf
```

**Opção 1 - Acesso Geral (menos seguro, apenas para testes):**

Adicionar ao final do arquivo:

```
host    all             all             0.0.0.0/0               md5
host    all             all             ::/0                    md5
```

**Opção 2 - Acesso Restrito por Componente (recomendado para produção):**

Limitar acesso apenas aos componentes do cluster para melhor segurança:

```
# Automation Controller (AWX)
host    awx     awx     192.168.100.51/32 scram-sha-256
host    awx     awx     192.168.100.52/32 scram-sha-256
host    awx     awx     192.168.100.53/32 scram-sha-256

# Automation Gateway
host    gateway gateway 192.168.100.51/32 scram-sha-256
host    gateway gateway 192.168.100.52/32 scram-sha-256
host    gateway gateway 192.168.100.53/32 scram-sha-256

# Automation Hub (Pulp)
host    pulp    pulp    192.168.100.56/32 scram-sha-256

# Event-Driven Ansible
host    eda     eda     192.168.100.57/32 scram-sha-256
```

Após editar, reiniciar o PostgreSQL:

```bash
sudo systemctl restart postgresql
```

#### 8. Configurar Firewall (se ativo)

```bash
sudo firewall-cmd --permanent --add-port=5432/tcp
sudo firewall-cmd --reload
sudo firewall-cmd --list-ports
```

#### 9. Criar Usuários e Bancos de Dados

Conectar ao PostgreSQL:

```bash
sudo -u postgres psql
```

**Criar usuário ADMIN (opcional, geralmente mantido pelo DBA):**

```sql
CREATE ROLE aap_admin WITH LOGIN SUPERUSER PASSWORD 'StrongAdminPass!';
```

**Criar roles para cada componente:**

```sql
CREATE ROLE awx     LOGIN PASSWORD 'AwxPass!';
CREATE ROLE pulp    LOGIN PASSWORD 'PulpPass!';
CREATE ROLE eda      LOGIN PASSWORD 'EdaPass!';
CREATE ROLE gateway  LOGIN PASSWORD 'GatewayPass!';
```

**Criar databases:**

```sql
CREATE DATABASE awx     OWNER awx     ENCODING 'UTF8';
CREATE DATABASE pulp    OWNER pulp    ENCODING 'UTF8';
CREATE DATABASE eda     OWNER eda     ENCODING 'UTF8';
CREATE DATABASE gateway OWNER gateway ENCODING 'UTF8';
```

**Habilitar extensões obrigatórias para o Automation Hub:**

```sql
\c pulp

CREATE EXTENSION IF NOT EXISTS hstore;
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

\dx
```

Sair do psql:

```sql
\q
```

#### 10. Validar Criação dos Bancos e Permissões

```bash
sudo -u postgres psql -c "SELECT d.datname, r.rolname AS owner FROM pg_database d JOIN pg_roles r ON d.datdba = r.oid WHERE d.datname IN ('awx','pulp','eda','gateway') ORDER BY d.datname;"
```

#### 11. Validar Versão e Porta do PostgreSQL

```bash
# Validar versão
postgres -V

# Validar porta
ss -lntu | grep 5432
```

### Mapeamento de Bancos de Dados

| Database | Componente | Usuário | Descrição |
|----------|------------|---------|-----------|
| `awx` | Automation Controller | `awx` | Banco de dados do Automation Controller |
| `pulp` | Automation Hub | `pulp` | Banco de dados do Private Automation Hub |
| `eda` | Event-Driven Ansible | `eda` | Banco de dados do Event-Driven Ansible Controller |
| `gateway` | Automation Gateway | `gateway` | Banco de dados do Automation Gateway |

## Configuração do Sistema Operacional

### Pré-requisitos Gerais

Todos os servidores devem ter:

- RHEL 9 instalado e registrado no RHSM
- SELinux configurado (Enforcing ou Permissive conforme política)
- Portas de firewall liberadas conforme documentação do AAP
- Usuário `ansible` criado com acesso sudo
- Acesso SSH habilitado

### Configuração de Usuário e Acesso

```bash
# Criar usuário ansible
useradd ansible

# Definir senha (substituir pela senha desejada)
echo 'ansible:*********' | chpasswd

# Configurar sudo
echo 'ansible ALL=(ALL) ALL' > /etc/sudoers.d/ansible && chmod 440 /etc/sudoers.d/ansible
```

## Arquivo de Inventory

Este é o arquivo de inventory-growth para instalação containerizada multi-node com banco de dados externo:

inventory-growth

```ini
# inventory-growth
# This is the AAP installer inventory file intended for the Container growth deployment topology.
# This inventory file expects to be run from the host where AAP will be installed.
# Please consult the Ansible Automation Platform product documentation about this topology's tested hardware configuration.
# https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.6/html/tested_deployment_models/container-topologies
#
# Please consult the docs if you're unsure what to add
# For all optional variables please consult the included README.md
# or the Ansible Automation Platform documentation:
# https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.6/html/containerized_installation

# This section is for your AAP Gateway host(s)
# -----------------------------------------------------
[automationgateway]
labac01.aroque.com.br  ansible_host=192.168.100.51
labac02.aroque.com.br  ansible_host=192.168.100.52
labac03.aroque.com.br  ansible_host=192.168.100.53

# This section is for your AAP Controller host(s)
# -----------------------------------------------------
[automationcontroller]
labac01.aroque.com.br  ansible_host=192.168.100.51
labac02.aroque.com.br  ansible_host=192.168.100.52
labac03.aroque.com.br  ansible_host=192.168.100.53

# This section is for your AAP Automation Hub host(s)
# -----------------------------------------------------
[automationhub]
labhub01.aroque.com.br ansible_host=192.168.100.56

# This section is for your AAP EDA Controller host(s)
# -----------------------------------------------------
[automationeda]
labeda01.aroque.com.br ansible_host=192.168.100.57


# This section is for your AAP Lightspeed host(s)
# -----------------------------------------------------
# [ansiblelightspeed]
# aap.example.org

# This section is for your Ansible MCP Server host(s)
# -----------------------------------------------------
# [ansiblemcp]
# aap.example.org

# This section is for the AAP database
# -----------------------------------------------------
#[database]
#labdb01.aroque.com.br

#referencia database
# awx > Automation Controller
# pulp > Automation Hub
# eda > Event-Driven Ansible
# gateway > Automation Gateway


[all:vars]
# Ansible
#ansible_connection=local

#CONF DE ACESSO - Se necessario
ansible_user=ansible
#ansible_ssh_pass=*********
#ansible_become=true
#ansible_become_method=sudo

# Common variables
# https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.6/html/containerized_installation/appendix-inventory-files-vars#ref-general-inventory-variables
# -----------------------------------------------------
#postgresql_admin_username=postgres
#postgresql_admin_password=

registry_username=*******
registry_password=*******

redis_mode=standalone

# AAP Gateway
# https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.6/html/containerized_installation/appendix-inventory-files-vars#ref-gateway-variables
# -----------------------------------------------------
gateway_admin_password=redhat*99
gateway_pg_host=labdb01.aroque.com.br
gateway_pg_database=gateway
gateway_pg_username=gateway
gateway_pg_password=GatewayPass!

# AAP Controller
# https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.6/html/containerized_installation/appendix-inventory-files-vars#ref-controller-variables
# -----------------------------------------------------
controller_admin_password=redhat*99
controller_pg_host=labdb01.aroque.com.br
controller_pg_database=awx
controller_pg_username=awx
controller_pg_password=AwxPass!
controller_percent_memory_capacity=0.5

# AAP Automation Hub
# https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.6/html/containerized_installation/appendix-inventory-files-vars#ref-hub-variables
# -----------------------------------------------------
hub_admin_password=redhat*99
hub_pg_host=labdb01.aroque.com.br
hub_pg_password=PulpPass!
hub_pg_database=pulp
hub_pg_username=pulp
hub_seed_collections=false

# AAP EDA Controller
# https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.6/html/containerized_installation/appendix-inventory-files-vars#event-driven-ansible-controller
# -----------------------------------------------------
eda_admin_password=redhat*99
eda_pg_host=labdb01.aroque.com.br
eda_pg_password=EdaPass!
eda_pg_database=eda
eda_pg_username=eda

# AAP Lightspeed
# https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.6/html/containerized_installation/appendix-inventory-files-vars#ref-lightspeed-variables
# -----------------------------------------------------
# lightspeed_admin_password=<set your own>
# lightspeed_pg_host=aap.example.org
# lightspeed_pg_password=<set your own>

# In case chabot is enabled, default provider is "rhoai"
# lightspeed_chatbot_model_url=<set your own>
# lightspeed_chatbot_model_api_key=<set your own>
# lightspeed_chatbot_model_id=<set your own>

# In case "azure" provider
# lightspeed_chatbot_default_provider = "azure"

# In case "openai" provider
# lightspeed_chatbot_default_provider = "openai"

# lightspeed_mcp_controller_enabled=true
# lightspeed_mcp_lightspeed_enabled=true
# lightspeed_wca_model_api_key=<set your own>
# lightspeed_wca_model_id=<set your own>
```

### Explicação das Variáveis do Inventory

#### Variáveis Comuns

- `registry_username` / `registry_password`: Credenciais para acesso ao Red Hat Container Registry
- `redis_mode`: Modo do Redis (standalone neste caso)

#### Variáveis do Automation Gateway

- `gateway_admin_password`: Senha do administrador do Gateway
- `gateway_pg_host`: Hostname do banco de dados externo
- `gateway_pg_database`: Nome do banco de dados (gateway)
- `gateway_pg_username`: Usuário do banco de dados
- `gateway_pg_password`: Senha do usuário do banco de dados

#### Variáveis do Automation Controller

- `controller_admin_password`: Senha do administrador do Controller
- `controller_pg_host`: Hostname do banco de dados externo
- `controller_pg_database`: Nome do banco de dados (awx)
- `controller_pg_username`: Usuário do banco de dados
- `controller_pg_password`: Senha do usuário do banco de dados
- `controller_percent_memory_capacity`: Percentual de memória para capacidade (0.5 = 50%)

#### Variáveis do Automation Hub

- `hub_admin_password`: Senha do administrador do Hub
- `hub_pg_host`: Hostname do banco de dados externo
- `hub_pg_database`: Nome do banco de dados (pulp)
- `hub_pg_username`: Usuário do banco de dados
- `hub_pg_password`: Senha do usuário do banco de dados
- `hub_seed_collections`: Se deve popular coleções iniciais (false = não)

#### Variáveis do Event-Driven Ansible

- `eda_admin_password`: Senha do administrador do EDA
- `eda_pg_host`: Hostname do banco de dados externo
- `eda_pg_database`: Nome do banco de dados (eda)
- `eda_pg_username`: Usuário do banco de dados
- `eda_pg_password`: Senha do usuário do banco de dados

## Executar a Instalação

Após configurar o banco de dados externo e o arquivo de inventory, execute a instalação:

```bash
ansible-playbook -i inventory-growth ansible.containerized_installer.install -vv
```

O parâmetro `-vv` habilita modo verbose para acompanhar o progresso da instalação em detalhes.

## Testes de Validação

### Teste de Conexão com o Banco de Dados

Antes de executar a instalação, valide a conectividade dos servidores AAP com o banco de dados:

```bash
# Do servidor labac01, testar conexão com o banco
psql -h labdb01.aroque.com.br -U awx -d awx -c "SELECT version();"
psql -h labdb01.aroque.com.br -U gateway -d gateway -c "SELECT version();"

# Do servidor labhub01
psql -h labdb01.aroque.com.br -U pulp -d pulp -c "SELECT version();"

# Do servidor labeda01
psql -h labdb01.aroque.com.br -U eda -d eda -c "SELECT version();"
```

### Teste de Acesso SSH entre Servidores

```bash
# Testar conectividade entre todos os servidores
ansible all -i inventory-growth -m ping
```

## Referências

- [Red Hat Ansible Automation Platform 2.6 - Containerized Installation Guide](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.6/html-single/containerized_installation/index)
- [Setting up an external database with PostgreSQL admin credentials](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.6/html-single/containerized_installation/index#setup-ext-db-with-admin-creds)
- [Containerized Inventory File Variables](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.6/html/containerized_installation/appendix-inventory-files-vars)

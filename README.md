# Documentação de Instalação do Ansible Automation Platform (AAP)

Este repositório contém documentações detalhadas de instalação de ambientes **Ansible Automation Platform (AAP)** de diferentes versões.

## Sobre o Repositório

Este repositório foi criado para centralizar e organizar as documentações de instalação do AAP em diferentes versões e métodos de instalação. Cada documentação contém informações específicas sobre:

- Configuração do ambiente
- Pré-requisitos do sistema
- Estrutura de rede
- Configurações de inventário
- Procedimentos de instalação
- Troubleshooting e resolução de problemas

## Documentações Disponíveis

### AAP 2.4

- **[Instalação AAP 2.4 via RPM](docs/aap-2.4-rpm-instalacao.md)** - Documentação completa de instalação do AAP 2.4 Bundle via RPM, incluindo configuração do ambiente, testes de acesso, troubleshooting e resolução de problemas comuns.

### AAP 2.6

- **[Instalação AAP 2.6 Containerizado Multi-Node com Banco Externo](docs/aap-2.6-container-multinode-external-db.md)** - Documentação completa de instalação do AAP 2.6 em arquitetura containerizada multi-node com banco de dados PostgreSQL externo, incluindo configuração detalhada do banco de dados, estrutura de rede, inventory e procedimentos de instalação.

### AAP 2.7

- **[Instalação AAP 2.7 Containerizado All-in-One (Growth)](docs/aap-2.7-container-all-in-one.md)** - Documentação completa de instalação do AAP 2.7 containerizado com todos os serviços em um único nó (topologia Growth), incluindo Gateway, Controller, Hub, EDA, Automation Metrics, Lightspeed, MCP Server e PostgreSQL implantado pelo próprio instalador.

- **[Instalação AAP 2.7 Containerizado HA Mínimo](docs/aap-2.7-container-ha-minimo-2-nodes-db-externo.md)** - Documentação completa de instalação do AAP 2.7 containerizado simulando alta disponibilidade com 2 nós híbridos (todos os serviços replicados) e um nó de banco de dados dedicado, com PostgreSQL implantado pelo próprio instalador.

## Contribuindo

Se você tiver documentações de outras versões do AAP ou métodos de instalação diferentes, sinta-se à vontade para contribuir adicionando novas documentações seguindo a estrutura existente.

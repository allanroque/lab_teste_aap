# Documentação do Ambiente de Teste e Troubleshooting

## Descrição do Ambiente

- **Sistema Operacional**: RHEL 8.9
- **CPU**: 2 vCPUs
- **Memória**: 8GB
- **Armazenamento**: 40GB (Montado em `/var/lib/awx`)
- **Versão do AAP**: 2.4 Bundle

## Estrutura de Rede

- **AC01**: 192.168.100.111
- **AC02**: 192.168.100.112
- **AC03**: 192.168.100.113
- **DB01**: 192.168.100.114
- **Gateway**: 192.168.100.1
- **DNS**: 8.8.8.8

## Configuração do Sistema Operacional

### Geral

- Registradas no RHSM
- SELinux: `Enforcing`
- Portas de firewall liberadas conforme documentação

### Usuário e Acesso

- Usuário `ansible` criado
  - `useradd ansible`
  - `echo 'ansible:redhat*99' | chpasswd`
- Acesso SSH habilitado para `ansible`

### Configuração do Sudo

- Usuário `ansible` configurado para sudo com senha
  - `echo 'ansible ALL=(ALL) ALL' > /etc/sudoers.d/ansible && chmod 440 /etc/sudoers.d/ansible`
- Acesso SSH como root direto bloqueado (`PermitRootLogin no`)
  - `sed -i 's/^PermitRootLogin yes/PermitRootLogin no/' /etc/ssh/sshd_config && systemctl restart sshd`

## Arquivo `/etc/hosts` dos Servidores

```
127.0.0.1 localhost
::1 localhost
192.168.100.111 ac01 ac01.aroque.com.br
192.168.100.112 ac02 ac02.aroque.com.br
192.168.100.113 ac03 ac03.aroque.com.br
192.168.100.114 db01 db01.aroque.com.br
```

## Testes de Acesso e Configurações

### Pré-requisitos

- **Ansible Core** instalado (`yum install ansible-core`)

### Teste de Conexão

- Necessário realizar teste de SSH do AC01 para todos os hosts para adicionar ao `known_hosts`

```bash
export ANSIBLE_HOST_KEY_CHECKING=True
ansible-playbook -i inventory teste_acesso.yml
```

### Playbook teste_acesso.yml

Este playbook realiza testes de ping e executa o comando id para verificar conexões e privilégios.

```yaml
---
- name: Teste de Conexão e Verificação de Usuários no Ansible
  hosts: all
  gather_facts: no
  tasks:
    - name: Verifica conectividade via ping
      ping:

    - name: Executa comando 'id' sem privilégios elevados
      command: id
      register: id_normal

    - name: Exibe resultado do comando 'id' sem privilégios elevados
      debug: var=id_normal.stdout

    - name: Executa comando 'id' com privilégios elevados (root)
      command: id
      become: true
      register: id_become

    - name: Exibe resultado do comando 'id' com privilégios elevados
      debug: var=id_become.stdout

    - name: Executa comando 'id' como usuário 'awx' com elevação
      command: id
      become: true
      become_user: awx
      register: id_become_user
      ignore_errors: true

    - name: Exibe resultado do comando 'id' como usuário 'awx'
      debug: var=id_become_user.stdout
```

### Executar a instalação

```bash
./setup
```


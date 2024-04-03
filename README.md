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

## Testes Extras

```bash
ansible-playbook -i inventory teste_acesso2.yml
```

teste_acesso2.yml
```yaml
---
- name: Teste de Conexão e Verificação de Usuários no Ansible
  hosts: all
  gather_facts: no

  tasks:
    - name: Verifica conectividade com todos os hosts do inventário via ping
      ping:
      # Esta tarefa garante que conseguimos alcançar todos os hosts listados no inventário.

    # TESTE 1: Execução do comando 'id' sem privilégios elevados
    - name: Executa o comando 'id' para o usuário padrão do Ansible (sem privilégios elevados)
      command: id
      register: id_normal
      # Executa o comando 'id' como o usuário sob o qual o Ansible opera por padrão.

    - name: Exibe o resultado do comando 'id' executado como usuário padrão do Ansible
      debug:
        var: id_normal.stdout
      # Exibe o UID, GID e grupos do usuário padrão do Ansible.

    # TESTE 2: Execução do comando 'id' com privilégios elevados (become: true)
    - name: Executa o comando 'id' com privilégios elevados (become=true) como root
      command: id
      become: true
      register: id_become
      # Utiliza a opção 'become: true' para executar o comando 'id' como root.

    - name: Exibe o resultado do comando 'id' executado com privilégios elevados
      debug:
        var: id_become.stdout
      # Exibe o UID, GID e grupos do usuário root, demonstrando que o comando foi executado com sucesso com elevação de privilégios.

    - name: Executa o comando 'id' com elevação para o usuário específico 'awx'
      command: id
      become: true
      become_user: awx
      register: id_become_user
      ignore_errors: true
      # Tenta executar o comando 'id' como o usuário 'awx' usando elevação de privilégios.

    - name: Exibe o resultado do comando 'id' executado como o usuário 'awx'
      debug:
        var: id_become_user.stdout
      # Exibe o resultado da execução do comando 'id' como o usuário 'awx', ou exibe um erro se o usuário 'awx' não existir ou não puder ser acessado.

    - name: Verifica as regras de sudo para o usuário atual do Ansible
      command: sudo -l
      register: sudo_para_usuario_padrao
      ignore_errors: true
      # Verifica e registra as regras de sudo disponíveis para o usuário sob o qual o Ansible está sendo executado.

    - name: Exibe as regras de sudo disponíveis para o usuário atual do Ansible
      debug:
        var: sudo_para_usuario_padrao.stdout
      # Exibe as regras de sudo para o usuário padrão de execução, mostrando os comandos que o usuário pode executar como sudo.

    - name: Verifica as regras de sudo para o usuário 'awx'
      command: sudo -l -U awx
      register: sudo_para_awx
      ignore_errors: true
      # Verifica e registra as regras de sudo disponíveis para o usuário 'awx'.

    - name: Exibe as regras de sudo disponíveis para o usuário 'awx'
      debug:
        var: sudo_para_awx.stdout
      # Exibe as regras de sudo para o usuário 'awx', mostrando os comandos que o usuário 'awx' pode executar como sudo.
```

## Ponto de Atenção

## Erro Durante a Execução de Rsync com Sudo

### Descrição do Erro

Durante a instalação, o playbook encontrou um erro ao tentar executar o comando `rsync` para copiar pacotes do bundle para o diretório de destino nos hosts. Os logs relevantes incluem mensagens de aviso de depreciação e um erro fatal indicando que "sudo: a terminal is required to read the password".

Este erro ocorre porque o comando `rsync` foi chamado com sudo, mas sem uma maneira de fornecer a senha de sudo de forma interativa ou através de um ajudante de senha (askpass helper).

Evidência:

```bash
[DEPRECATION WARNING]: The connection's stdin object is deprecated. Call 
display.prompt_until(msg) instead. This feature will be removed in version 
2.19. Deprecation warnings can be disabled by setting 
deprecation_warnings=False in ansible.cfg.
fatal: [ac02.aroque.com.br]: FAILED! => {"changed": false, "cmd": "sshpass -d4 /usr/bin/rsync --delay-updates -F --compress --delete-after --archive --rsh='/usr/bin/ssh -S none -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null' --rsync-path='sudo -u root rsync' --out-format='<<CHANGED>>%i %n%L' /home/ansible/ansible-automation-platform-setup-bundle-2.4-5-x86_64/./bundle/packages/el8/repos/ ansible@ac02.aroque.com.br:/var/lib/ansible-automation-platform-bundle", "msg": "Warning: Permanently added 'ac02.aroque.com.br,192.168.100.112' (ECDSA) to the list of known hosts.\r\nsudo: a terminal is required to read the password; either use the -S option to read from standard input or configure an askpass helper\nrsync: connection unexpectedly closed (0 bytes received so far) [sender]\nrsync error: error in rsync protocol data stream (code 12) at io.c(226) [sender=3.1.3]\n", "rc": 12}
........
```

### Possivel Solução

Para contornar este problema, uma task subsequente foi adicionada para realizar a operação necessária sem o uso de sudo ou configurando sudo para não requerer uma senha quando chamado a partir de rsync. Uma abordagem pode ser configurar o sudoers para permitir a execução do comando rsync por parte do usuário ansible sem solicitar senha.

Exemplo de entrada no sudoers (isso deve ser feito com cautela e considerando as implicações de segurança):

```bash
ansible ALL=(ALL) NOPASSWD: /usr/bin/rsync
```

Impacto na Instalação
É importante notar que esse erro não impactou a conclusão bem-sucedida da instalação.

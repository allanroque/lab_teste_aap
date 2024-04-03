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
´´´
127.0.0.1 localhost
::1 localhost
192.168.100.111 ac01 ac01.aroque.com.br
192.168.100.112 ac02 ac02.aroque.com.br
192.168.100.113 ac03 ac03.aroque.com.br
192.168.100.114 db01 db01.aroque.com.br
´´´

# Projeto_Vagrant_Ansible

projeto/
├── Vagrantfile
├── playbook.yml
├── update_system.yml
├── configure_hostname.yml
├── create_user.yml
├── configure_sudo.yml
├── configure_ssh.yml
├── configure_lvm.yml
├── configure_nfs.yml
├── monitor_access.yml


Aluno: Wiliam Terroso de Sousa Melo 
Professor: Pedro Carvalho
Disciplina: Administração de Sistemas Abertos

Relatório detalhado do projeto:

O código Vagrant fornecido configura uma máquina virtual (VM) usando a plataforma VirtualBox, com a instalação do Debian 12 e provisionamento automatizado via Ansible. Aqui está um resumo detalhado do que o código faz:

Definição da Caixa de VM:

A configuração define que a VM será baseada na caixa generic/debian12, que é uma imagem do Debian 12.
Configuração do Provider VirtualBox:

O provider utilizado é o VirtualBox, que define o nome da VM como p01_Wiliam.
A memória da VM é configurada para 1024 MB (1 GB).
Configuração de Rede:

A VM é configurada para usar uma rede privada com o endereço IP 192.168.57.10, o que significa que ela estará em uma rede interna, não acessível diretamente pela rede externa.
Configuração de Discos Adicionais:

São criados três discos virtuais de 10 GB cada (disk1.vdi, disk2.vdi, disk3.vdi), que são então anexados à VM através do controlador SATA.
Cada disco é atribuído a uma porta SATA diferente, permitindo que a VM tenha múltiplos discos.
Sincronização de Pastas:

O código configura uma pasta sincronizada, permitindo que a pasta local (onde o Vagrantfile está localizado) seja mapeada para a pasta /vagrant dentro da VM.
Isso facilita o compartilhamento de arquivos entre a máquina host e a VM.
Provisionamento com Ansible:

O código configura o provisionamento automático da VM usando o Ansible.
O playbook Ansible a ser executado é o playbook.yml localizado no diretório atual.
A opção ansible.become = true permite que o Ansible execute comandos com privilégios de superusuário na VM.
Resumo:
O código configura uma máquina virtual Debian 12 com 1 GB de memória, três discos virtuais de 10 GB cada e uma rede privada. Além disso, ele sincroniza a pasta do host com a VM e provisiona a VM automaticamente usando um playbook do Ansible.



Provisionamento Ansible

1. `playbook.yml`: Arquivo principal de orquestração que importa e executa todos os outros playbooks em sequência:
   - Atualização do sistema
   - Configuração do hostname
   - Criação de usuário
   - Configuração de permissões sudo
   - Configuração de SSH
   - Configuração de LVM (Logical Volume Management)
   - Configuração do servidor NFS
   - Configuração de monitoramento de acesso

2. `update_system.yml`: Atualiza o cache de pacotes do sistema e faz upgrade de todos os pacotes

3. `configure_hostname.yml`: Define o hostname como "p01-Wiliam"

4. `create_user.yml`: 
   - Cria um usuário chamado "wiliam"
   - Adiciona uma mensagem de aviso de acesso ao sistema (/etc/motd)

5. `configure_sudo.yml`:
   - Cria um grupo "ifpb"
   - Concede a este grupo acesso sudo sem senha

6. `configure_ssh.yml`:
   - Desabilita login do root
   - Desabilita autenticação por senha
   - Reinicia o serviço SSH

7. `configure_lvm.yml`:
   - Instala LVM2
   - Cria um grupo de volumes "dados" usando /dev/sdb, /dev/sdc, /dev/sdd
   - Cria um volume lógico de 15GB chamado "sistema"
   - Formata o volume como ext4
   - Configura montagem automática no /etc/fstab

8. `configure_nfs.yml`:
   - Instala o servidor NFS
   - Configura um diretório compartilhado em /dados/nfs
   - Define permissões de acesso de rede para a subrede 192.168.57.0/24

9. `monitor_access.yml`:
   - Cria um script para registrar detalhes de acesso do usuário
   - Registra carimbo de tempo, nome de usuário, TTY e informações do cliente em /dados/nfs/acessos


# Relatório Detalhado da Infraestrutura - Ordem de Execução

## 1. Vagrantfile
Este é o arquivo principal que define a infraestrutura virtual.
```ruby
Vagrant.configure("2") do |config|
  config.vm.box = "generic/debian12"
  config.vm.provider "virtualbox" do |vb|
    vb.name = "p01_Wiliam"
    vb.memory = 1024
  end
  
  # Configuração de rede bridge
  config.vm.network "public_network", bridge: "enp4s0"
  # Mantenho também a rede privada original
  config.vm.network "private_network", ip: "192.168.57.10"
  
  # Configuração de discos adicionais
  config.vm.provider "virtualbox" do |vb|
    vb.customize ["createhd", "--filename", "disk1.vdi", "--size", 10240]
    vb.customize ["createhd", "--filename", "disk2.vdi", "--size", 10240]
    vb.customize ["createhd", "--filename", "disk3.vdi", "--size", 10240]
    vb.customize ["storageattach", :id, "--storagectl", "SATA Controller", "--port", 1, "--device", 0, "--type", "hdd", "--medium", "disk1.vdi"]
    vb.customize ["storageattach", :id, "--storagectl", "SATA Controller", "--port", 2, "--device", 0, "--type", "hdd", "--medium", "disk2.vdi"]
    vb.customize ["storageattach", :id, "--storagectl", "SATA Controller", "--port", 3, "--device", 0, "--type", "hdd", "--medium", "disk3.vdi"]
  end
  
  config.vm.synced_folder ".", "/vagrant", create: true
  
  config.vm.provision "ansible" do |ansible|
    ansible.playbook = "playbook.yml"
    ansible.become = true
  end
end
```
Características principais:
- VM Debian 12 com 1GB RAM
- Rede bridge e privada
- 3 discos adicionais de 10GB cada
- Provisionamento via Ansible

## 2. playbook.yml
Orquestra a execução de todos os playbooks na ordem correta.
```yaml
- import_playbook: update_system.yml
- import_playbook: configure_hostname.yml
- import_playbook: create_user.yml
- import_playbook: configure_sudo.yml
- import_playbook: configure_ssh.yml
- import_playbook: configure_lvm.yml
- import_playbook: configure_nfs.yml
- import_playbook: monitor_access.yml
```
Ordem de execução planejada para garantir dependências corretas.

## 3. update_system.yml
Primeira etapa: atualização do sistema.
```yaml
- name: Atualizar o sistema operacional
  hosts: all
  become: yes
  tasks:
    - name: Atualizar cache e pacotes do sistema
      apt:
        update_cache: yes
        upgrade: safe
```
Garante que o sistema esteja atualizado antes das configurações.

## 4. configure_hostname.yml
Define o nome do host.
```yaml
- name: Configurar o hostname
  hosts: all
  tasks:
    - name: Alterar hostname
      hostname:
        name: "p01-Wiliam"
```
Estabelece a identidade do servidor na rede.

## 5. create_user.yml
Cria usuário e configura mensagem de boas-vindas.
```yaml
- name: Criar usuário e configurar mensagem de boas-vindas
  hosts: all
  tasks:
    - name: Criar usuário wiliam
      user:
        name: "wiliam"
        state: present
        shell: /bin/bash

    - name: Configurar mensagem de boas-vindas
      copy:
        dest: /etc/motd
        content: |
          Acesso restrito apenas à pessoas com autorização expressa
          Seu acesso está sendo monitorado !!!
```
Prepara o ambiente para acesso do usuário.

## 6. configure_sudo.yml
Configura permissões administrativas.
```yaml
- name: Configurar SUDO para o grupo ifpb
  hosts: all
  tasks:
    - name: Criar grupo ifpb
      group:
        name: "ifpb"
        state: present

    - name: Adicionar grupo ifpb ao sudoers
      copy:
        dest: /etc/sudoers.d/ifpb
        content: "%ifpb ALL=(ALL) NOPASSWD:ALL"
        mode: "0440"
```
Estabelece privilégios administrativos para o grupo.

## 7. configure_ssh.yml
Implementa configurações de segurança SSH.
```yaml
- name: Configurar SSH
  hosts: all
  tasks:
    - name: Gerar chave SSH no computador local
      openssh_keypair:
        path: "~/.ssh/id_rsa"
        type: rsa
        size: 4096
        state: present
        force: no
      delegate_to: localhost
      become: no

    - name: Criar diretório .ssh no servidor remoto
      file:
        path: ~/.ssh
        state: directory
        mode: '0700'

    - name: Copiar chave pública para o servidor remoto
      authorized_key:
        user: "{{ ansible_user }}"
        state: present
        key: "{{ lookup('file', '~/.ssh/id_rsa.pub') }}"

    - name: Bloquear login do root e autenticação por senha
      lineinfile:
        path: /etc/ssh/sshd_config
        regexp: "{{ item.regexp }}"
        line: "{{ item.line }}"
      with_items:
        - { regexp: "^PermitRootLogin", line: "PermitRootLogin no" }
        - { regexp: "^PasswordAuthentication", line: "PasswordAuthentication no" }

    - name: Reiniciar serviço SSH
      service:
        name: sshd
        state: restarted
```
Configura acesso seguro ao servidor.

## 8. configure_lvm.yml
Configura o gerenciamento de volumes lógicos.
```yaml
- name: Configurar LVM
  hosts: all
  become: yes
  tasks:
    - name: Instalar o pacote lvm2
      apt:
        name: lvm2
        state: present

    - name: Criar volume group "dados"
      command: vgcreate dados /dev/sdb /dev/sdc /dev/sdd

    - name: Criar logical volume "sistema"
      command: lvcreate -L 15G -n sistema dados

    - name: Formatar o volume em ext4
      filesystem:
        fstype: ext4
        dev: /dev/dados/sistema

    - name: Configurar montagem automática no fstab
      blockinfile:
        path: /etc/fstab
        marker: "# {mark} ANSIBLE MANAGED BLOCK"
        block: "/dev/dados/sistema /dados ext4 defaults 0 0"
```
Prepara o armazenamento do sistema.

## 9. configure_nfs.yml
Configura o servidor de arquivos.
```yaml
- name: Configurar servidor NFS
  hosts: all
  tasks:
    - name: Instalar NFS server
      apt:
        name: nfs-kernel-server
        state: present

    - name: Configurar diretório compartilhado
      blockinfile:
        path: /etc/exports
        marker: "# {mark} ANSIBLE MANAGED BLOCK"
        block: |
          /dados/nfs 192.168.57.0/24(rw,sync,no_root_squash,all_squash,anonuid=1000,anongid=1000)
```
Habilita compartilhamento de arquivos na rede.

## 10. monitor_access.yml
Implementa o sistema de monitoramento.
```yaml
- name: Monitoramento de acessos
  hosts: all
  tasks:
    - name: Criar script de monitoramento
      copy:
        dest: /etc/profile.d/track_access.sh
        content: |
          echo "$(date '+%Y-%m-%d %H:%M:%S'); $USER; $SSH_TTY; $SSH_CLIENT" >> /dados/nfs/acessos
        mode: "0755"
```
Estabelece o registro de acessos ao sistema.

## Fluxo de Execução

1. **Vagrant** inicia a VM e configura recursos básicos
2. **playbook.yml** orquestra a sequência de configuração
3. Sistema é atualizado (**update_system.yml**)
4. Identidade do servidor é estabelecida (**configure_hostname.yml**)
5. Usuário é criado (**create_user.yml**)
6. Permissões administrativas são configuradas (**configure_sudo.yml**)
7. Acesso SSH é securizado (**configure_ssh.yml**)
8. Storage é configurado (**configure_lvm.yml**)
9. Compartilhamento de arquivos é habilitado (**configure_nfs.yml**)
10. Sistema de monitoramento é implementado (**monitor_access.yml**)

Esta sequência garante que cada componente seja configurado após suas dependências estarem prontas, criando um ambiente completo e seguro.

O projeto é uma configuração abrangente de sistema usando Ansible, focando em atualizações do sistema, gerenciamento de usuários, fortalecimento de segurança, configuração de armazenamento e monitoramento de acesso.



# 🤖 Guia de Estudo: Ansible

> Automação de infraestrutura com Playbooks, inventários e IaC (Infrastructure as Code).

**Autor:** Ariel Closs Novais | **Licença:** MIT

---

## Índice

1. [O que é o Ansible?](#o-que-é-o-ansible)
2. [Pré-requisitos](#pré-requisitos)
3. [Inventário de Hosts](#inventário-de-hosts)
4. [Playbooks](#playbooks)
5. [Variáveis](#variáveis)
6. [Handlers](#handlers)
7. [Roles](#roles)
8. [Comandos Úteis](#comandos-úteis)
9. [Boas Práticas](#boas-práticas)

---

## O que é o Ansible?

O Ansible é uma ferramenta open-source de automação de TI que permite:

- **Provisionar** servidores e infraestrutura
- **Configurar** sistemas operacionais e aplicações
- **Implantar** aplicações de forma repetível
- **Orquestrar** fluxos complexos de múltiplas etapas

Diferente de outras ferramentas, o Ansible é **agentless** — não precisa de software instalado nos servidores gerenciados, apenas SSH e Python.

```
┌─────────────────────┐         SSH / Python
│  Máquina de Controle│ ──────────────────────► Nó Gerenciado 01
│  (Ansible instalado) │ ──────────────────────► Nó Gerenciado 02
└─────────────────────┘ ──────────────────────► Nó Gerenciado 03
```

---

## Pré-requisitos

### Máquina de controle (onde o Ansible roda)

| Requisito | Detalhe |
|-----------|---------|
| Sistema operacional | Linux ou macOS |
| Python | 3.8 ou superior |
| Ansible | Instalado via `pip` ou gerenciador de pacotes |
| Acesso SSH | Chave pública configurada nos nós |

```bash
# Instalação no Ubuntu
sudo apt update && sudo apt install ansible -y

# Verificar versão
ansible --version
```

### Nós gerenciados (servidores alvo)

| Requisito | Detalhe |
|-----------|---------|
| SSH | Instalado e rodando (`openssh-server`) |
| Python | 3.x instalado |
| Usuário | Com permissão sudo (para tarefas privilegiadas) |

```bash
# Instalar SSH no nó gerenciado (Ubuntu)
sudo apt install openssh-server -y

# Configurar chave SSH da máquina de controle
ssh-copy-id usuario@192.168.1.10
```

---

## Inventário de Hosts

O inventário define **quais servidores** o Ansible irá gerenciar e **como agrupá-los**.

### Formato INI (`hosts.ini`)

```ini
# ─── Grupo: Servidores Web ────────────────────────────────────────────────────
[webservers]
192.168.1.10
192.168.1.11
webserver01.example.com

# ─── Grupo: Banco de Dados ───────────────────────────────────────────────────
[dbservers]
db01.example.com
10.0.0.50

# ─── Grupo de grupos (meta-grupo) ────────────────────────────────────────────
[producao:children]
webservers
dbservers

# ─── Variáveis por grupo ─────────────────────────────────────────────────────
[webservers:vars]
ansible_user=ubuntu
ansible_python_interpreter=/usr/bin/python3
```

### Formato YAML (`hosts.yml`)

```yaml
all:
  children:
    webservers:
      hosts:
        192.168.1.10:
        192.168.1.11:
        webserver01.example.com:
    dbservers:
      hosts:
        db01.example.com:
        10.0.0.50:
          ansible_port: 2222  # porta SSH customizada
```

> **Dica:** O formato YAML é preferível para inventários complexos por ser mais legível e suportar hierarquias aninhadas facilmente.

### Verificar inventário

```bash
# Listar todos os hosts
ansible all -i hosts.ini --list-hosts

# Listar hosts de um grupo específico
ansible webservers -i hosts.ini --list-hosts
```

---

## Playbooks

Playbooks são arquivos YAML que descrevem **o estado desejado** dos seus servidores. São a espinha dorsal do Ansible.

### Estrutura de um Playbook

```yaml
---
# O traço triplo indica início de documento YAML

- name: Nome descritivo do play       # Identifica o que este bloco faz
  hosts: webservers                    # Grupo do inventário alvo
  become: yes                          # Equivalente ao sudo
  vars:                                # Variáveis locais do play
    pacote_web: nginx

  tasks:
    - name: Descrição da tarefa
      modulo_ansible:
        parametro: valor
```

### Exemplo 1 — Instalar e configurar Nginx

```yaml
# instalar_nginx.yml
---
- name: Instalação e configuração do Nginx
  hosts: webservers
  become: yes

  tasks:
    - name: Atualizar cache do apt
      apt:
        update_cache: yes
        cache_valid_time: 3600       # Evita atualizar se já foi feito na última hora
      when: ansible_os_family == "Debian"

    - name: Instalar o Nginx
      package:
        name: nginx
        state: present               # present = instalar | absent = remover | latest = atualizar

    - name: Garantir que o Nginx está rodando e habilitado
      service:
        name: nginx
        state: started
        enabled: yes                 # Inicia automaticamente no boot
```

### Exemplo 2 — Criar usuário e configurar diretório

```yaml
# configurar_usuario.yml
---
- name: Configuração de usuário de deploy
  hosts: all
  become: yes

  tasks:
    - name: Criar usuário "deploy"
      user:
        name: deploy
        shell: /bin/bash
        create_home: yes
        groups: sudo
        append: yes

    - name: Criar diretório da aplicação
      file:
        path: /opt/minha_app
        state: directory
        owner: deploy
        group: deploy
        mode: '0755'

    - name: Copiar arquivo de configuração
      copy:
        src: ./config/app.conf
        dest: /opt/minha_app/app.conf
        owner: deploy
        mode: '0644'
```

### Exemplo 3 — Playbook com múltiplos plays

```yaml
# deploy_completo.yml
---
- name: Preparar banco de dados
  hosts: dbservers
  become: yes
  tasks:
    - name: Instalar PostgreSQL
      package:
        name: postgresql
        state: present

- name: Preparar servidores web
  hosts: webservers
  become: yes
  tasks:
    - name: Instalar Nginx
      package:
        name: nginx
        state: present

    - name: Instalar dependências da aplicação
      pip:
        name:
          - gunicorn
          - flask
        state: present
```

### Executar um Playbook

```bash
# Execução básica
ansible-playbook -i hosts.ini instalar_nginx.yml

# Simular execução sem aplicar mudanças (dry-run)
ansible-playbook -i hosts.ini instalar_nginx.yml --check

# Mostrar diff das mudanças que seriam feitas
ansible-playbook -i hosts.ini instalar_nginx.yml --check --diff

# Executar com verbosidade (útil para debug)
ansible-playbook -i hosts.ini instalar_nginx.yml -v    # básico
ansible-playbook -i hosts.ini instalar_nginx.yml -vvv  # detalhado

# Limitar execução a um host específico
ansible-playbook -i hosts.ini instalar_nginx.yml --limit 192.168.1.10

# Pedir confirmação de senha sudo
ansible-playbook -i hosts.ini instalar_nginx.yml --ask-become-pass
```

---

## Variáveis

Variáveis permitem tornar seus playbooks **reutilizáveis e dinâmicos**.

### Definindo variáveis

```yaml
# No próprio playbook (vars)
---
- name: Deploy da aplicação
  hosts: webservers
  vars:
    app_name: minha_app
    app_port: 8080
    app_dir: /opt/{{ app_name }}

  tasks:
    - name: Criar diretório {{ app_name }}
      file:
        path: "{{ app_dir }}"
        state: directory
```

### Arquivo de variáveis externo (`vars/main.yml`)

```yaml
# vars/main.yml
app_name: minha_app
app_port: 8080
db_host: db01.example.com
db_name: producao
packages:
  - nginx
  - git
  - curl
```

```yaml
# Referenciando no playbook
- name: Deploy
  hosts: webservers
  vars_files:
    - vars/main.yml
  tasks:
    - name: Instalar pacotes
      package:
        name: "{{ item }}"
        state: present
      loop: "{{ packages }}"
```

### Variáveis de host e grupo (`host_vars/` e `group_vars/`)

```
inventario/
├── hosts.ini
├── group_vars/
│   ├── webservers.yml      # Variáveis para todos os webservers
│   └── dbservers.yml       # Variáveis para todos os dbservers
└── host_vars/
    └── 192.168.1.10.yml    # Variáveis específicas para este host
```

---

## Handlers

Handlers são tarefas especiais executadas **apenas quando notificadas** por outra tarefa. Muito usados para reiniciar serviços somente quando há mudança de configuração.

```yaml
---
- name: Configurar Nginx
  hosts: webservers
  become: yes

  tasks:
    - name: Copiar configuração do Nginx
      copy:
        src: ./nginx.conf
        dest: /etc/nginx/nginx.conf
      notify: Reiniciar Nginx         # Notifica o handler se houver mudança

    - name: Habilitar site
      file:
        src: /etc/nginx/sites-available/meusite
        dest: /etc/nginx/sites-enabled/meusite
        state: link
      notify: Reiniciar Nginx         # Mesmo handler, só roda uma vez ao final

  handlers:
    - name: Reiniciar Nginx
      service:
        name: nginx
        state: restarted
```

> **Importante:** Mesmo que o handler seja notificado várias vezes, ele só executa **uma vez**, ao final do play.

---

## Roles

Roles organizam playbooks complexos em **estrutura modular e reutilizável**. São a forma recomendada de organizar automações de médio e grande porte.

### Estrutura de uma Role

```
roles/
└── nginx/
    ├── tasks/
    │   └── main.yml        # Tarefas principais
    ├── handlers/
    │   └── main.yml        # Handlers da role
    ├── templates/
    │   └── nginx.conf.j2   # Templates Jinja2
    ├── files/
    │   └── index.html      # Arquivos estáticos
    ├── vars/
    │   └── main.yml        # Variáveis fixas
    ├── defaults/
    │   └── main.yml        # Variáveis com valores padrão
    └── meta/
        └── main.yml        # Metadados e dependências
```

### Criando uma role

```bash
# Criar estrutura automaticamente
ansible-galaxy init roles/nginx
```

### Usando roles no playbook

```yaml
---
- name: Configurar servidores web
  hosts: webservers
  become: yes
  roles:
    - nginx
    - { role: firewall, quando: configurar_firewall | bool }
```

---

## Comandos Úteis

### Conectividade e diagnóstico

```bash
# Testar conectividade com todos os hosts
ansible all -i hosts.ini -m ping

# Testar grupo específico
ansible webservers -i hosts.ini -m ping

# Coletar informações do sistema (facts)
ansible all -i hosts.ini -m setup

# Coletar apenas informações de rede
ansible all -i hosts.ini -m setup -a "filter=ansible_interfaces"
```

### Executar comandos ad-hoc

```bash
# Executar comando shell nos hosts
ansible all -i hosts.ini -m shell -a "uptime"

# Verificar espaço em disco
ansible all -i hosts.ini -m shell -a "df -h"

# Instalar pacote rapidamente
ansible webservers -i hosts.ini -m apt -a "name=curl state=present" --become

# Reiniciar serviço
ansible webservers -i hosts.ini -m service -a "name=nginx state=restarted" --become
```

### Validação de playbooks

```bash
# Verificar sintaxe YAML
ansible-playbook -i hosts.ini meu_playbook.yml --syntax-check

# Listar todas as tarefas do playbook (sem executar)
ansible-playbook -i hosts.ini meu_playbook.yml --list-tasks

# Listar hosts que seriam afetados
ansible-playbook -i hosts.ini meu_playbook.yml --list-hosts
```

---

## Boas Práticas

### Organização de projeto

```
meu_projeto/
├── ansible.cfg             # Configurações padrão do Ansible
├── hosts.ini               # Inventário
├── site.yml                # Playbook principal (ponto de entrada)
├── roles/                  # Roles reutilizáveis
│   ├── nginx/
│   └── postgresql/
├── group_vars/             # Variáveis por grupo
├── host_vars/              # Variáveis por host
└── playbooks/              # Playbooks específicos
    ├── deploy.yml
    └── backup.yml
```

### Arquivo `ansible.cfg`

```ini
[defaults]
inventory       = hosts.ini
remote_user     = ubuntu
private_key_file = ~/.ssh/id_rsa
host_key_checking = False

[privilege_escalation]
become          = True
become_method   = sudo
become_user     = root
```

### Dicas essenciais

| Prática | Por quê |
|---------|---------|
| Use `--check` antes de aplicar | Previne erros em produção |
| Nomeie todas as tarefas | Facilita leitura dos logs |
| Prefira módulos Ansible a comandos shell | Módulos são idempotentes |
| Use `no_log: yes` para senhas | Evita expor dados sensíveis |
| Versione seu inventário no Git | Rastreabilidade e rollback |
| Use `ansible-vault` para segredos | Criptografa variáveis sensíveis |

### Idempotência

O Ansible foi projetado para ser **idempotente**: rodar o mesmo playbook múltiplas vezes deve produzir o mesmo resultado, sem efeitos colaterais.

```yaml
# ✅ Correto — idempotente: o módulo verifica se já existe
- name: Instalar Nginx
  package:
    name: nginx
    state: present

# ⚠️  Cuidado — pode não ser idempotente dependendo do comando
- name: Instalar Nginx via shell
  shell: apt install nginx
```

### Criptografar segredos com Ansible Vault

```bash
# Criptografar arquivo de variáveis
ansible-vault encrypt vars/secrets.yml

# Editar arquivo criptografado
ansible-vault edit vars/secrets.yml

# Executar playbook com vault
ansible-playbook -i hosts.ini site.yml --ask-vault-pass
```

---

## Referências

- [Documentação oficial do Ansible](https://docs.ansible.com)
- [Ansible Galaxy — Roles da comunidade](https://galaxy.ansible.com)
- [Repositório no GitHub](https://github.com/ansible/ansible)

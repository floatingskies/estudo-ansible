Guia de Estudo: Ansible
Este repositório serve como um guia de estudo pessoal para automação utilizando Ansible, focado na criação de Playbooks, gestão de inventários e infraestrutura como código (IaC).

Pré-requisitos
Para que o Ansible funcione corretamente em um ambiente Ubuntu 24.04, a seguinte configuração é necessária:

Máquina de Controle (Host):
Ansible instalado.
Python instalado.
Nós Gerenciados (Targets):
SSH instalado e rodando.
Python instalado (necessário para execução dos módulos Ansible).
Acesso via chave SSH configurado (recomendado) ou senha.
📁 Inventário de Hosts (hosts.ini)
O arquivo de inventário define os hosts que serão gerenciados. O Ansible suporta endereçamento IP (Classes A, B, C) e nomes de domínio (DNS).

Crie um arquivo chamado hosts.ini:

ini

[webservers]
192.168.1.10
192.168.1.11
webserver01.example.com

[dbservers]
db01.example.com
10.0.0.50
Nota: O agrupamento entre colchetes [nome_do_grupo] permite segmentar tarefas específicas para cada tipo de servidor.

Playbooks
Os Playbooks são arquivos escritos em YAML (Yet Another Markup Language) que descrevem as automações. O Ansible segue os padrões de indentação rigorosos do YAML.

Exemplo: instalar_pacotes.yml
Este playbook atualiza o cache do APT e instala o Nginx nos servidores do grupo webservers.
- name: Instalação e Configuração de Pacotes
  hosts: webservers
  become: yes  # Equivalente ao sudo
  tasks:
    - name: Atualizar cache do apt (Debian/Ubuntu)
      apt:
        update_cache: yes
      when: ansible_os_family == "Debian"

    - name: Instalar o Nginx
      package:
        name: nginx
        state: present
Detalhes do Playbook:

---: Indica o início de um arquivo YAML.
hosts: webservers: Define que as tarefas rodarão apenas no grupo definido no hosts.ini.
become: yes: Eleva os privilégios para root (sudo).
when: Condicionais para garantir que o comando execute apenas no sistema operacional correto.
Executando o Playbook
Para rodar a automação, utilize o comando ansible-playbook apontando para o inventário e o arquivo YAML:


ansible-playbook -i hosts.ini instalar_pacotes.yml
Comandos Úteis
Verificar conectividade (Ping):
bash

ansible all -i hosts.ini -m ping
Checar sintaxe do Playbook:
bash

ansible-playbook -i hosts.ini instalar_pacotes.yml --syntax-check
Autor: Ariel Closs Novais
Licença: MIT

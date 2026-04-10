# estudo-ansible
Um guia de estudo para mim em ansible/yaml/playbook
# Requirementos para funcionar o Ansible nos computadores com Ubuntu 24.04
-  SSH (todos)
- Python (todos)
- Ansible (Na maquina host automatadora)



## A seguir a criação de hosts.ini
``` 
[webservers]
x.x.x.x
x.x.x.x
x.x.x.x

[dbservers]
://exemplo.com
```

### Há suporte a Classe A, B, e C de endereço Internet Protocol

Alem de seguir o padrão .yaml da Red Hat, pois literalmente foi criado pela Red Hat o Ansible, seguindo o padrão

```—
- name: Instalacao de pacotes
  hosts: webservers
  become: yes
  tasks:
    - name: Atualizar cache do apt (Debian/Ubuntu)
      apt:
        update_cache: yes
      when: ansible_os_family == "Debian"

    - name: Instalar o Nginx
      package:
        name: nginx
        state: present

```

E assim executamos o playbook com o seguinte comando
```ansible-playbook -i hosts.ini instalar_pacotes.yml```



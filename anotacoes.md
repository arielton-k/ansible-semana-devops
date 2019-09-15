# ansible-semana-devops

Anotações do video sobre Ansible da semana DevOps do canal LinuxTips.

## Instalação

```
sudo apt install -y software-properties-common
sudo apt-add-repository --yes --update ppa:ansible/ansible
sudo apt update
sudo apt install -y ansible
```

Versão do Ansible

```
ansible --version
```

Criando uma pasta

```
mkdir ansible
cd ansible

cat << EOF > hosts
[local]
localhost ansible_connection=local ansible_python_interpreter=python gather_facts=false

[semanadevops]
EOF
```

Rodando o ansible add-hoc

```
ansible -i hosts all -m ping
```

Criando um playbook

```
cat << EOF > main.yml
- hosts: local
  roles:
  - create
EOF
```

Criando a pasta

```
mkdir -p roles
cd roles
```

Criando a estrutura do projeto

```
ansible-galaxy init create
cd create
```

```
cat << EOF > tasks/provisioning.yml
- name: Criando o Security Group
  local_action:
    module: ec2_group
    name: "{{ security_group }}"
    description: Security Group Giropops
    region: "{{ region }}"
    rules:
    - proto: tcp
      from_port: 22
      to_port: 22
      cidr_ip: 0.0.0.0/0
    rules_egress:
    - proto: all
      cidr_ip: 0.0.0.0/0
    register: basic_firewall

- name: Criando a instancia EC2
  local_action: ec2
    group={{ security_group }}
    instance_type={{ instance_type }}
    image={{ image }}
    wait=true
    region={{ region }}
    keypair={{ keypair }}
    count={{ count }}
  register: ec2

- name: Adicionando a instancia ao inventario temp
  add_host: name={{ item.public_ip }} groups=giropops-new
  with_items: "{{ ec2.instances }}"

- name: Adicionando a instancia criada no arquivo hosts
  local_action: lineinfile
    dest="./hosts"
    regexp={{ item.public_ip }}
    insertafter="[semanadevops]" line={{ item.public_ip }}
  with_items: "{{ ec2.instances }}"

- name: Esperando o ssh
  local_action: wait_for
    host={{ item.public_ip }}
    port=22
    state=started
  with_items: "{{ ec2.instances }}"

- name: Adicionando um nome tag na instancia
  local_action: ec2_tag resource={{ item.id }} region={{ region }} state=present
  with_items: "{{ ec2.instances }}"
  args:
    tags:
      Name: SemanaDevOps

- name: Adicionando a maquina criada no known_hosts
  shell: ssh-keyscan -H {{ item.public_ip }} >> ~/.ssh/known_hosts
  with_items: "{{ ec2.instances }}"
EOF
```

```
printf "\n- include: provisioning.yml" >> tasks/main.yml
```

```
cat << EOF > vars/main.yml
---
# vars file for create
instance_type: t2.medium
security_group: giropops
image: ami-064a0193585662d74
keypair: semanadevops-ansible
region: us-east-1
count: 3
EOF
```

Na AWS, vá em IAM e crie um novo usuário.

Após criar um usuário pegue o AWS_ACCESS_KEY_ID (ID da chave de acesso) e AWS AWS_SECRET_ACCESS_KEY (Chave de acesso secreta).

E faça:

```
export AWS_ACCESS_KEY_ID="AKIA********729X"
export AWS_SECRET_ACCESS_KEY="Rd0****************cah"
```

Depois vá em EC2, no menu do lado esquerdo clique em 'Key Pairs' e crie uma chave. Chame a chave de 'semanadevops-ansible'. E faça o download desta chave.

E copie a chave para a pasta do projeto.

Depois faça:

```
chmod 0400 semanadevops-ansible.pem
```

```
ssh-add semanadevops-ansible.pem
```

Rodando o playbook

```
ansible-playbook -i hosts main.yml
```

Para se conectar em cada máquina, clique em cima dela no EC2, e clique em conectar, dai copie o comando de instrução para conexão via ssh.

Exemplo:

```
ssh -i "semanadevops-ansible.pem" ubuntu@ec2-90-22-1-40.compute-1.amazonaws.com
```

Ao rodar o comando

```
ansible -i hosts all -m ping
```

... ele dá um erro. Para resolver isso, faça:

```
ansible -i hosts all -m ping -u ubuntu
```

```
ansible -i hosts -u ubuntu semanadevops -a "/sbin/ifconfig eth0"
ansible -i hosts -u ubuntu semanadevops -m setup
```

Clonando um repo na máquina

```
ansible -i hosts -u ubuntu semanadevops -m git -a "repo=https://github.com/badtuxx/giropops-monitoring.git dest=/tmp/giropops version=HEAD"
```
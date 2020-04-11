# Ansible playbook deep dive
## Dynamic inventory 
Dynamic is useful for some automation, it is written is shell. They come generally come from cloud provider.
* Best practrices:
> Create an inventory file for each environment.

## Using YAML for ansible playbooks
> A yaml file is open by three hyphens (---) and close by three periods (...)
---
concentration: DevOps
### List of courses
courses:
- Ansible
- Openshift
- Configuration Management
- Containerized Application Development
...

### dictionary
```sh
user:
  name: Mouhamadou
  job: engineer
  salary: 100000
```
### Create new line with | or >
> | or > is to insert multiple line text which will be interpreted as a single text
include_newlines: |
   ansible is really a powerful tool
   is feel in love with this tool
   it is amazing
### Special characters
> In ansible the following characters have a meaning
> `[], {}, |, > and :`
> so if we want use them, we can put them in double quotes.

### Variables
> In ansible variable name are wrapped in {{}}
`"{{variable_name}}"`

### Boolean
`become: yes`

### Floating
`version: 1.0` this will interpreted as version: 1, if we want keep floating part we have to wrappe it in double quote `version: "1.0"`

## Creating an Ansible Play
An ansible play is :
- A set of instructions defined in a yaml file. 
- It targets a hosts or a group of hosts
- It provides a series of tasks (a sequence of ad-hoc commands that are executed sequentially)

```sh
---
- hosts: localhost
    become: yes
    gather_facts: yes
    tasks:
       - name: Install elinks
         yum:
           name: elinks
           state: latest
      - name: Install git
        yum:
          name: git
          state: installed
 
- hosts: all
    tasks: 
       - name: download google
         get_url:
            url: "www.google.com"
            dest: /home/ansible
       - name: add line in file
         lineinfile:
            path: path/to/file
            line: hello world
            state: present
```

### Ansible playbook commands
#
```sh
# Run ansible playbook
ansible-playbook -i inventory playbook.yml
# Run ansible playbook -k ask for the connected password
ansible-playbook -i inventory playbook.yml -k
# Run ansible playbook -K to ask to become password
ansible-playbook -i inventory playbook.yml -K
# Run ansible playbook -C run in check mode
ansible-playbook -i inventory playbook.yml -C
```

### Ansible tags
> We can add tag and use them later to execute selected tasks
```sh
ansible-playbook play.yml --tags tagname
ansible-playbook play.yml --tags tagname1,tagname2,..
ansible-playbook play.yml -t tagname1,tagname2,..

# skip-tags
ansible-playbook play.yml --skip-tags tagname1,tagname2,..
```

### Ansible vault
> Ansible valut is use to save sensitive data, we use ansible-vault to decrypt a vaault file
- --vault-id: use label for label
 ```sh
  # create a vault
  ansible-vault create vault.yml
  # crypting sensitive data
  ansible-vault encrypt vault
  # decrypting sensitive data
  ansible-vault decrypt vault
  # use --vault-id for label prod and prompt for id
  ansible-vault encrypt --vault-id prod@prompt vault
  # view vault
  ansible-vault view vault.yml
  # edit vault
  ansible-vault edit vault
  # create vault from string
  ansible-vault encrypt-string  --vault-id prod@prompt 'key: value'
```

- how to create a vault
```sh
--- # Vault example
- hosts: localhost
  vars_files:
    /home/ansible/vault
  tasks:
    - name: Add secret text to open.txt
      lineinfile:
        path: /home/ansible/open.txt
        create: yes
        line: "{{ password }}"
      no_log: true
```

### Errors Handling with block, rescue and always
> In ansible we can handle errors during execution of playbooks
```sh
--- 
- hosts: all
   tasks:
     - name: Install software
     block:
       - service:
           name: htttpd
           state: started
         register: service_status
     rescue:
       - debug:
           var: service_status
     always:
       - debug:
            msg: Ensure that the service is running.
```
### Asynchronuous tasks
> We can perform asynchronuous task in ansible, fro this we can define in playboo the following attributes
`async`: we define the time above which a task timeout
`poll`: how often ansible check if a task is completed or not

```sh
---
- hosts: <hostname ou group of host>
  tasks: 
    - name: Install a software
      command: /home/ansible/sleep.sh
      async: 60
      poll: 0 # 10s is the default polling time
```


### Delegating Playbook Execution with `delegate_to` and `local_action`
> We can delegate task execution to a host with delegate_to.

```sh
---
- hosts: <hostname ou group of host>
  become: yes
  tasks: 
    - name: Install a software
      command: /home/ansible/sleep.sh
      async: 60
      poll: 0 # 10s is the default polling time
      delegate_to: node1
    - name: Install mariadb
      yum:
        name: mariadb
        state: absent 
```
```sh
# we can perform a task on localhost wi local_action
local_action: yum name=httpd state=latest
```

### Ansible parallelism
#### forks
-  Allow execute of task on a set of servers in parallel using fork
- Number of forks is 5 by defaults (defined /etc/ansible/ansible.cfg)
- `ansible -i -inventory <host or group of hosts> -m <module name> -a <list of args> -f <number of forks>`
- `ansible -i -inventory <host or group of hosts> -m <module name> -a <list of args>  --forks <number of forks>`

#### serial
> we can target a batch of hosts rather than targeting all hosts simulteneously

- case of one parameters
```sh
---
- hosts: all
  # Perform task by batch with serial
  # batching hosts 
  serial: 5
  tasks:
    - name: Install elinks
      package:
        name: elinks
        state: latest 
```

- Case of a list of parameters
```sh
---
- hosts: all
  # Perform task by batch with serial
  # batching hosts 
  serial:
  - 1
  - 5
  - 3
  tasks:
    - name: Install elinks
      package:
        name: elinks
        state: latest 
```

- Case of a list of parameters with percentage
```sh
---
- hosts: all
  # Perform task by batch with serial
  # batching hosts 
  serial:
  - "10%"
  - "50%"
  - "40%"
  tasks:
    - name: Install elinks
      package:
        name: elinks
        state: latest 
```

- Case of a list of parameters with mixed of number and percentage
```sh
---
- hosts: all
  # Perform task by batch with serial
  # batching hosts 
  serial:
  - 5
  - "30%"
  - "50%"
  tasks:
    - name: Install elinks
      package:
        name: elinks
        state: latest 
```

#### max_fail_percentage 
> `max_fail_percentage` is used to abort tasks execution if many hosts fail

```sh
---
- hosts: all
  # abort a play if many hosts fails play execution
  max_fail_percentage: 10
  # batching hosts
  # remember batch size is limit by the number of forks bu default 5 max.
  serial:
  - 5
  - "30%"
  - "50%"
  tasks:
    - name: Install elinks
      package:
        name: elinks
        state: latest 
```
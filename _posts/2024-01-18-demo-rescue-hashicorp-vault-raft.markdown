---
title: "Rescue Vault from a broken raft setup"
excerpt: "This is a simple demonstration of how to rescue HashiCorp Vault from a broken quorum from a bad setup."
tags: 
  - vault
  - demo
---
All code can be found on [GitHub](https://github.com/fredrikwarfvinge/demos-vault-raft-lost-quorum){:target="_blank"}

This is a simple demonstration of how to rescue Vault from a broken quorum from a bad setup (2 nodes) and finally what happens if you are following the suggested 3 or more node setup and one node fails.

This is meant to be a purely educational experience, do not use this for production, or on a production cluster.

## Requirements

The setup expects VirtualBox hostnetwork to accept the following IP range: 192.168.56.10-192.168.56.12

- A virtual machine software (VirtualBox) [https://www.virtualbox.org/](link){:target="_blank"}
- HashiCorp Vagrant [https://www.vagrantup.com/](link){:target="_blank"}
- Ansible [https://www.ansible.com/](link){:target="_blank"}
- Python 3 [https://www.python.org/](link){:target="_blank"}
- Public ssh key saved in ~/.ssh/id_rsa.pem
- Strong enough machine to run up to 3 headless Ubuntu 22.04 servers

## Usage

Note: The Ansible code from the pre-setup stage can be used to configure Vault on any setup of your choosing, however, it is not meant for production use.

### Pre-setup - Server installation

For sake of ease, create these files in a subfolder named servers.
Create a new Vagrantfile with the following contents:

```hcl
Vagrant.configure("2") do |config|
  config.vm.define :first do |first|
    first.vm.box = "ubuntu/jammy64"
    first.vm.hostname = "first"
    first.vm.network :private_network, ip: "192.168.56.10"
    first.vm.synced_folder ".", "/vagrant", create: true
    ssh_pub_key = File.readlines("#{Dir.home}/.ssh/id_rsa.pub").first.strip
    first.vm.provision 'shell', inline: "echo #{ssh_pub_key} >> /home/vagrant/.ssh/authorized_keys", privileged: false    
    first.vm.provision "ansible_local" do |ansible|
      ansible.playbook = "install_vault.yml"
      ansible.extra_vars = {
        server_ip: "192.168.56.10"
      }
      ansible.pip_install_cmd = "sudo apt-get install -y python3-distutils && curl -s https://bootstrap.pypa.io/get-pip.py | sudo python3"
      ansible.install_mode = "pip"
    end
  end

  config.vm.define :second do |second|
    second.vm.box = "ubuntu/jammy64"
    second.vm.hostname = "second"
    second.vm.network :private_network, ip: "192.168.56.11"
    second.vm.synced_folder ".", "/vagrant", create: true
    ssh_pub_key = File.readlines("#{Dir.home}/.ssh/id_rsa.pub").first.strip
    second.vm.provision 'shell', inline: "echo #{ssh_pub_key} >> /home/vagrant/.ssh/authorized_keys", privileged: false    
    second.vm.provision "ansible_local" do |ansible|
      ansible.playbook = "install_vault.yml"
      ansible.extra_vars = {
        server_ip: "192.168.56.11"
      }
      ansible.pip_install_cmd = "sudo apt-get install -y python3-distutils && curl -s https://bootstrap.pypa.io/get-pip.py | sudo python3"
      ansible.install_mode = "pip"
    end
  end

  config.vm.define :third do |third|
    third.vm.box = "ubuntu/jammy64"
    third.vm.hostname = "third"
    third.vm.network :private_network, ip: "192.168.56.12"
    third.vm.synced_folder ".", "/vagrant", create: true
    ssh_pub_key = File.readlines("#{Dir.home}/.ssh/id_rsa.pub").first.strip
    third.vm.provision 'shell', inline: "echo #{ssh_pub_key} >> /home/vagrant/.ssh/authorized_keys", privileged: false    
    third.vm.provision "ansible_local" do |ansible|
      ansible.playbook = "install_vault.yml"
      ansible.extra_vars = {
        server_ip: "192.168.56.12"
      }
      ansible.pip_install_cmd = "sudo apt-get install -y python3-distutils && curl -s https://bootstrap.pypa.io/get-pip.py | sudo python3"
      ansible.install_mode = "pip"
    end
  end
end
```

Also ensure you create the playbook file as well as the jinja2 template for the Vault configuration used:

vault.hcl.j2
```hcl
storage "raft" {
  path = "/opt/vault/data/"
  node_id = "{{ hostname }}"
  retry_join {
    leader_api_addr = "http://192.168.56.10:8200"
  }
  retry_join {
    leader_api_addr = "http://192.168.56.11:8200"
  }
}

listener "tcp" {
  address = "0.0.0.0:8200"
  tls_disable = "true"
}

api_addr = "http://{{ ip }}:8200"
cluster_addr = "http://{{ ip }}:8201"
ui = true
cluster_name = "vault"
```

install_vault.yml
```yml
- name: Setup remote Vault servers
  hosts: all
  become: true
    
  tasks:
  - name: Add an Apt signing key, uses whichever key is at the URL
    ansible.builtin.apt_key:
      url: https://apt.releases.hashicorp.com/gpg
      state: present

  - name: Add specified repository into sources list
    ansible.builtin.apt_repository:
      repo: deb [arch=amd64] https://apt.releases.hashicorp.com jammy main
      state: present

  - name: Update repositories cache and install Vault
    apt:
      pkg:
      - vault
      - jq
      update_cache: yes
      state: present

  - name: Creates directory
    file:
      path: /opt/vault/data/
      state: directory
      owner: vault
      group: vault
      mode: 0775

  - name: Write the vault.hcl file
    template:
      src: vault.hcl.j2
      dest: /etc/vault.d/vault.hcl
      owner: vault
      group: vault
    vars:
      hostname: "{{ ansible_fqdn }}"
      ip: "{{ server_ip }}"

  - name: Recursively change ownership of a directory
    ansible.builtin.file:
      path: /opt/vault/
      state: directory
      recurse: yes
      owner: vault
      group: vault

  - name: Enable Vault
    ansible.builtin.service:
      name: vault
      enabled: yes
      state: stopped

  - name: Add Vault address export to .bashrc
    ansible.builtin.lineinfile:
      path: "/home/vagrant/.profile"
      line: 'export VAULT_ADDR=http://127.0.0.1:8200'
      state: present
```

### Pre-setup Ansible files

In this section, you will create 4 files. 2 playbooks, 1 inventory file and one configuration file for Ansible.
To ensure you don't mix these with the server setup files, make these in a folder named ansible, next to your servers folder that you used in the previous steps.

Let's start with the configuration file.
ansible.cfg
```
[defaults]
inventory = ./inventory.yml
host_key_checking = False
```
This ensures two things, one that the inventory we use is the one for the demo setup, and secondly we disregard the issue with self-signed certificates. Change this file to suit your own needs if you divert from the files we created in the Pre-setup servers stage. 

Second, we will create the inventory file. This is based on the Vagrantfile configuration above.

```yaml
vault:
    hosts:
        first:
            ansible_host: 192.168.56.10
            ansible_ssh_private_key_file: ~/.ssh/id_rsa
            ansible_user: vagrant
        second:
            ansible_host: 192.168.56.11
            ansible_ssh_private_key_file: ~/.ssh/id_rsa
            ansible_user: vagrant
        third:
            ansible_host: 192.168.56.12
            ansible_ssh_private_key_file: ~/.ssh/id_rsa
            ansible_user: vagrant

local:
    hosts:
        localhost:
            ansible_host: 127.0.0.1
            ansible_connection: local
```

Finally we will create two playbooks. The first will initialize and unseal Vault on your Vagrant boxes:
initialize_and_unseal_vault.yml
```yaml
- name: Initialize and unseal first Vault
  hosts: first
  become: true
  gather_facts: true
  environment:
    VAULT_ADDR: "http://127.0.0.1:8200"
    
  tasks:
  - name: Start Vault
    ansible.builtin.service:
      name: vault
      enabled: yes
      state: started

  - name: Check if Vault is initialized
    shell: vault status -format=json
    register: vault_status
    failed_when: ( vault_status.rc not in [ 0, 2 ] )

  - name: Set Vault status output as a variable
    set_fact:
      vault_status: "{{ vault_status.stdout | from_json }} "

  - name: Initialize Vault if necessary
    shell: vault operator init -key-shares=1 -key-threshold=1 -format=json
    register: vault
    when: not vault_status.initialized

  - name: Save Vault response as variable
    set_fact:
      vault_response: "{{ vault.stdout | from_json }}"
    when: not vault_status.initialized

  - name: Write root token to file
    copy:
      content: "{{ vault_response.root_token }}"
      dest: "root_token"
    when: not vault_status.initialized
  
  - name: Save root token to variable
    slurp:
      src: "root_token"
    register: root_token

  - name: Write unseal keys to file
    copy:
      content: "{{ vault_response.unseal_keys_hex }}"
      dest: "unseal_keys"
    when: not vault_status.initialized

  - name: Read unseal_keys from file
    slurp:
      src: "unseal_keys"
    register: slurpfile

  - name: Decode and clean the unseal key
    set_fact:
      unseal_key: "{% raw %}{{ slurpfile['content'] | b64decode | regex_replace('[\\[\\]]', '')}}{% endraw %}"

  - name: Check if Vault is initialized
    shell: vault status -format=json
    register: vault_status
    failed_when: ( vault_status.rc not in [ 0, 2 ] )

  - name: Set Vault status output as a variable
    set_fact:
      vault_status: "{{ vault_status.stdout | from_json }} "

  - name: Unseal Vault
    shell: vault operator unseal {{ unseal_key }}
    when: vault_status.initialized and vault_status.sealed

  - name: Write the root_token to a local file
    delegate_to: localhost
    become: false
    copy:
      content: "{{ root_token['content'] | b64decode }}"
      dest: "root_token.txt"

  - name: Write the unseal_key to a local file
    delegate_to: localhost
    become: false
    copy:
      content: "{{ unseal_key}}"
      dest: "unseal_key.txt"

  - name: Enable secret engine
    shell: vault secrets enable kv
    environment:
      VAULT_TOKEN: "{{  root_token['content'] | b64decode }}"
      VAULT_ADDR: "http://127.0.0.1:8200"
    register: enable_kv
    failed_when: ( enable_kv.rc not in [ 0, 2 ] )

- name: Initialize and unseal second/third Vault
  hosts: second third
  become: true
  gather_facts: true
  environment:
    VAULT_ADDR: "http://127.0.0.1:8200"
    
  tasks:
  - name: Read unseal_keys from file
    set_fact:
      unseal_key: "{% raw %}{{ lookup('file', 'unseal_key.txt') }}{% endraw %}"

  - name: Start Vault
    ansible.builtin.service:
      name: vault
      enabled: yes
      state: started

  - name: Check if Vault is initialized
    shell: vault status -format=json
    register: vault_status
    failed_when: ( vault_status.rc not in [ 0, 2 ] )

  - name: Set Vault status output as a variable
    set_fact:
      second_vault_status: "{{ vault_status.stdout | from_json }} "

  - name: Initialize Vault if necessary
    
    shell: vault operator init -key-shares=1 -key-threshold=1 -format=json
    register: second_vault
    when: not second_vault_status.initialized

  - name: Unseal Vault
    shell: vault operator unseal {{ unseal_key }}
    when: second_vault_status.sealed

  - name: Pause for 10 seconds
    ansible.builtin.pause:
      seconds: 10

- name: List enabled secrets engines
  hosts: vault
  become: false
  gather_facts: false
  tasks:
  - name: Read root token from file
    set_fact:
      root_token: "{% raw %}{{ lookup('file', 'root_token.txt') }}{% endraw %}"

  - name: List secrets engines and show that the accessor is the same on both nodes
    shell: vault secrets list -format=json | jq '."kv/".accessor'
    environment:
      VAULT_TOKEN: "{{  root_token }}"
      VAULT_ADDR: "http://127.0.0.1:8200"
    register: secret_list

  - name: Print listed accessors
    debug:
      msg: "{{ secret_list.stdout }}"
```

The last file we need to create is the one that causes the second Vault server to fail. It does this by forcibly removing the database for raft to simulate corruption. 

cause_quorum_break.yml
```yaml
- name: Split the brain by killing second
  hosts: second
  become: true
  gather_facts: false
  environment:
    VAULT_ADDR: "http://127.0.0.1:8200"

  tasks: 
  - name: Stop Vault
    ansible.builtin.systemd:
      name: vault
      state: stopped

  - name: Show that there are two folders
    ansible.builtin.find:
      paths: /opt/vault/
    register: vault_folder

  - name: Print nr of folders
    debug:
      msg: "{{ vault_folder.examined }}"

  - name: Delete Vault raft storage
    ansible.builtin.file:
      dest: /opt/vault/data/
      state: absent
  
  - name: Ensure the folder is deleted
    ansible.builtin.find:
      paths: /opt/vault/
    register: vault_folder

  - name: Print nr of folders
    debug:
      msg: "{{ vault_folder.examined }}"

  - name: Pause for 5 seconds
    ansible.builtin.pause:
      seconds: 5

  - name: Recreate raft storage folder
    file:
      path: /opt/vault/data/
      state: directory
      owner: vault
      group: vault
      mode: 0775

  - name: Start Vault
    ansible.builtin.systemd:
      name: vault
      state: started

  - name: Pause for 5 seconds
    ansible.builtin.pause:
      seconds: 5

  - name: Check if Vault is initialized
    shell: vault status -format=json
    register: vault_status
    failed_when: ( vault_status.rc not in [ 0, 2 ] )

  - name: Set Vault status output as a variable
    set_fact:
      vault_status: "{{ vault_status.stdout | from_json }} "

  - name: Initialize Vault if necessary
    shell: vault operator init -key-shares=1 -key-threshold=1 -format=json
    register: vault
    when: not vault_status.initialized

  - name: Save Vault response as variable
    set_fact:
      vault_response: "{{ vault.stdout | from_json }}"
    when: not vault_status.initialized

  - name: Read unseal_keys from file
    set_fact:
      unseal_key: "{% raw %}{{ lookup('file', 'unseal_key.txt') }}{% endraw %}"

  - name: Unseal Vault
    shell: vault operator unseal {{ unseal_key }}
    ignore_errors: true

- name: Verify split
  hosts: first
  become: false
  gather_facts: false
  environment:
    VAULT_ADDR: "http://127.0.0.1:8200"

  tasks:
  - name: Read root token from file
    set_fact:
      root_token: "{% raw %}{{ lookup('file', 'root_token.txt') }}{% endraw %}"
  
  - name: Check Vault peer list
    shell: vault operator raft list-peers -format=json
    register: vault_peers
    failed_when: ( vault_peers.rc not in [ 0, 2 ] )
    environment:
      VAULT_TOKEN: "{{ root_token }}"
    
  - name: Print Vault Peers list when error
    debug:
      msg: "{{ vault_peers.stderr }}"
    when: vault_peers.stderr | length > 0

  - name: Extract node_id values using json_query
    vars:
      query: "data.config.servers[*].node_id"
    set_fact:
      node_ids: "{% raw %}{{ vault_peers.stdout | from_json | json_query(query) }}{% endraw %}"
    when: vault_peers.stderr | length == 0

  - name: Print Vault Peers list when no error
    debug:
      msg: "{{node_ids}}"
    when: vault_peers.stderr | length == 0
```

### Setup initial Cluster (ansible)

Start by running `vagrant up first second` inside the servers folder.
This will provision two servers and provision them using Ansible within the virtual machines. 

Once the cluster is up and running, jump into the ansible folder and run `ansible-playbook initialize_and_unseal_vault.yml`. 
This will do the following:

1. Start Vault on the first server
2. Initialize it with 1 key-share
3. Record the unseal-key and root key and save it to your local machine
4. Unseal the first Vault
5. Enable the KV secrets engine in the cluster
6. Start Vault on the second server
7. Unseal the second Vault
8. List the accessor of the KV secrets engine from both vm's to show that they are identical (to prove that Raft is up and running)

### Destroy your Raft Quorum (ansible)

With the cluster up and running from the previous stage, you will now commence by destroying your quorum in the cluster.
Since your current setup is using a 2 node setup, this will cause Vault to be inoperable and because we destroy the Raft database on the second node it will be as if the node has been completely corrupted. 

To start, from the ansible folder, run: `ansible-playbook cause_quorum_break.yml`
This will do the following on the second server:

1. Stop Vault
2. List the amount of folders found in the `/opt/vault/` folder (should be 2 from the start).
3. Delete the data folder and show that the `/opt/vault/` folder now only contains one folder (the tls certs generated on Vault installation).
4. It will then recreate the data folder and start Vault and try to unseal Vault. For user readability, the error is ignored but printed to the terminal.

This next step will happen on the first server:
5. It tries to list the Raft peers in the Vault cluster and prints the error

### Rescue your first server (manual)

The next steps are all manual, meant to be done to practice how to restore quorum.
This will all be done on the first server. SSH into it using `ssh vagrant@192.168.56.10`.

1. Create a json file inside the folder `/opt/vault/data/raft/`, name it `peers.json` with the following content (if you run it in another server setup):
```json
[
    {
    "id": "first",
    "address": "192.168.56.10:8201",
    "non_voter": false
    }
]
```
2. Restart Vault by using `sudo systemctl restart vault`
3. Save the root token to environment variables on the server by using the following commands: `export VAULT_TOKEN=$(cat root_token)`
4. Unseal the server using: `vault operator unseal $(cat unseal_keys | tr -d '[]"')`
5. Check that the server is up and running, this can be done either with `vault status` and noting that HA Mode is "active", or by using `vault operator raft list-peers`

### Rescue the second server (manual)
After this, you can rescue the second server by the following on the second server:
1. Stop Vault by using `sudo systemctl stop vault`. 
2. Delete the folder `/opt/vault/data/` and recreate it using `sudo install -g vault -o vault -d /opt/vault/data/`.
3. Start up Vault by using `sudo systemctl start vault` and wait a couple of seconds.
4. Unseal Vault by using `vault operator unseal` and the unseal key you can find in the file unseal_key.txt on your computer.
5. Log in to Vault using `vault login` and use the root_token you find in the root_token.txt file on your computer.
6. You can see that you have restored the second server to the cluster by using `vault operator raft list-peers` or `vault secrets list` and noting that the secret engine `kv` is enabled (from the inital setup).

## Vault setup following recommendation

Back in the servers folder, run `vagrant destroy --force` followed by `vagrant up`. This will start three servers with Vault installed. 
Once they are up and running, jump into the ansible folder and run `ansible-playbook initialize_and_unseal_vault.yml`.
Once the playbook is done, run: `ansible-playbook cause_quorum_break.yml`.

It will finally print the list of Vault Peers from the first server, this means that even though the second server lost all Raft data Vault was able to retain quorum and bring in the second server. 
In a real scenario when you don't just delete the `/opt/vault/data/` folder but the server is fully broken, you would remove the dead server using `vault operator raft remove-peer node-id` command. 
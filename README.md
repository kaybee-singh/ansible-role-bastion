# ansible-role-bastion

Requirement:

- => An ansible controller node.
- => A Bastion node.

## We will do the configuration on the bastion node with the help of Ansible. Ansible will be running on a separate node.
## There are two roles in this repo.
```bash
ls -l roles/
total 0
drwxr-xr-x. 10 user user 188 Jul 19 22:52 ansible-role-bastion-half-automated
drwxr-xr-x. 10 user user 188 Jul 19 22:54 ansible-role-bastion-full-automated
```

- **First role ansible-role-bastion-half-automated** can configure DNS server, ha-proxy and will create an install-config.yaml file on the bastion node. Then you will have to add your pull-secret and bastion SSH key to the install-config.yaml and then create the manifests and ignition files manually on the bastion node. After that manually copy those ignition fiiles to the apache document root and configure apache to use 8080 port.
- **Second role ansible-role-bastion-full-automated** would do the fully automation from DNS configuration to creating ignition files and copying them to the apache document root and also configuring apache on 8080 port. For second role to do configuration properly you will have to place your pull-secret and bastion public-ssh key on the Ansible controller node at below place.
```bash
~/pull-secret
~/bastion_id_rsa.pub
```
---

In first section of this document - 1st role ansible-role-bastion-half-automated and how to use it is explained.

1. This role will configure a bastion node to install the Openshift UPI server.
2. This role will configure the DNS server, ha-proxy, and a http server to provide the ignition files to the OCP nodes.
3. To run this role follow below steps on the Ansible Controller node.

- Cone the git role
```bash
git clone https://github.com/kaybee-singh/ansible-role-bastion
```
4. Edit the Variables files in the role and add your values.
```bash
vim ansible-role-bastion/roles/ansible-role-bastion-half-automated/vars/main.yml 
```
5. Replace the default values for required parameters. DNS, HA-Proxy and install-config.yaml will be created on the basis of below variable file on the Bastion Node.
```yaml
cluster_name: ocp
base_domain: example.com
network_address: 192.168.100.0/24
gateway: 192.168.170.1
dns: 192.168.170.1
nodes:
  bastion:
    hostname: bastion
    ip: 192.168.122.170
    network_address: 192.168.122.0
    network_address_with_suffix: 192.168.122.0/24
  bootstrap:
    hostname: bootstrap
    ip: 192.168.170.150
  masters:
    - hostname: master-1
      ip: 192.168.170.151
    - hostname: master-2
      ip: 192.168.170.152
    - hostname: master-3
      ip: 192.168.170.153
  workers:
    - hostname: worker-1
      ip: 192.168.170.154
    - hostname: worker-2
      ip: 192.168.170.155
    - hostname: worker-3
      ip: 192.168.170.156
```
6. Create a test user on Bastion node and alow thkarane user to run root commands without password.
```bash
useradd test
echo "test	ALL=(ALL)	NOPASSWD: ALL" >> /etc/sudoers
```
7. Copy your SSH key from the Ansible Controller node to the bastion node.
```bash
ssh-keygen
ssh-copy-id test@192.168.122.170
```
7.1 Make sure following role is mentioned in playbook.yaml
```yaml
cat ansible-role-bastion/playbook.yaml
- name: Configure Bastion
  hosts: all
  become: yes
  vars_prompt:
    - name: username
      prompt: "Enter the rhsm username"
      private: no
    - name: password
      prompt: "Enter the rhsm password"
      private: yes
  roles:
    - role: ansible-role-bastion-half-automated
```
8. Add you bastion node's IP address in inventory hosts file
```bash
echo 192.168.122.170 >> ansible-role-bastion/hosts
```
10. Before running the role verify that the bastion node responds.
```bash
ansible all -m ping
```
10. Above command should response **pong** as below from the bastion Node's IP.
```bash
192.168.122.170 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "ping": "pong"
}
```
11. Now run the ansible-role from the controller node.
```bash
cd ansible-role-bastion; ansible-playbook playbook.yaml
```
12. First it will ask for the username and password to registry your bastion node with Red Hat to get access to the Red Hat repositories.
13 you already have access to a yum repository, either by local dvd or a third party repository you can provide any dummy username and password when asked.
```bash
Enter the rhsm username:
Enter the rhsm password: 
```
14. Download the client and install Openshift tar files.
```bash
wget https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/stable/openshift-client-linux.tar.gz
wget https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/stable/openshift-install-linux.tar.gz
```
15. Now extract these files
```bash
tar -xvf openshift-client-linux.tar.gz -C /usr/bin
tar -xvf openshift-install-linux.tar.gz -C /usr/bin -C /usr/bin
```
16. Check the installed versions and verify if oc commands works.
```
oc version
kubectl version
openshift-install version
```
17. Edit the install-config.yaml file on the bastion node and add the pull-secret and your SSH key. If you do not have a SSH key on the bastion node then you can create it first.
```bash
cd ~/ocpinstall
vim install-config.yaml
```
```bash
pullSecret: '{"auths": ...}'
sshKey: "ssh-ed25519 AAAA..."
```
18. Then create the manifests
```bash
openshift-install create manifests --dir=/root/ocpinstall
```
19. Create hte ignition files
```bash
openshift-install create ignition-configs --dir=/root/ocpinstall
```
20. Configure apache and copy the ingition files to the Apache directory.
```bash
mkdir /var/www/html/ocp4
cp /root/ocpinstall/*.ign /var/www/html/ocp4
restorecon -RFv /var/www/html/
sed -i 's/Listen 80/Listen 8080/g' /etc/httpd/conf/httpd.conf
systemctl enable httpd --now
```
---
## This is the second part of the document and illustrates how to use Role ansible-role-bastion-full-automated, to completely automate the process.

1. This role will configure a bastion node to install the Openshift UPI server.
2. This role will configure the DNS server, ha-proxy, and a http server to provide the ignition files to the OCP nodes.
3. To run this role follow below steps on the Ansible Controller node.

- Cone the git role
```bash
git clone https://github.com/kaybee-singh/ansible-role-bastion
```
4. Edit the Variables files in the role and add your values.
```bash
vim ansible-role-bastion/roles/ansible-role-bastion-full-automated/main.yml 
```
5. Replace the default values for required parameters. DNS, HA-Proxy and install-config.yaml will be created on the basis of below variable file on the Bastion Node.
```yaml
cluster_name: ocp
base_domain: example.com
network_address: 192.168.100.0/24
gateway: 192.168.170.1
dns: 192.168.170.1
nodes:
  bastion:
    hostname: bastion
    ip: 192.168.122.170
    network_address: 192.168.122.0
    network_address_with_suffix: 192.168.122.0/24
  bootstrap:
    hostname: bootstrap
    ip: 192.168.170.150
  masters:
    - hostname: master-1
      ip: 192.168.170.151
    - hostname: master-2
      ip: 192.168.170.152
    - hostname: master-3
      ip: 192.168.170.153
  workers:
    - hostname: worker-1
      ip: 192.168.170.154
    - hostname: worker-2
      ip: 192.168.170.155
    - hostname: worker-3
      ip: 192.168.170.156
```
6. Create a test user on Bastion node and alow the user to run root commands without password.
```bash
useradd test
echo "test	ALL=(ALL)	NOPASSWD: ALL" >> /etc/sudoers
```
7. Copy your SSH key from the Ansible Controller node to the bastion node.
```bash
ssh-keygen
ssh-copy-id test@192.168.122.170
```
8. Add you bastion node's IP address in inventory hosts file
```bash
echo 192.168.122.170 >> ansible-role-bastion/hosts
```
10. Before running the role verify that the bastion node responds.
```bash
ansible all -m ping
```
10. Above command should response **pong** as below from the bastion Node's IP.
```bash
192.168.122.170 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "ping": "pong"
}
```
11. Make sure that following role is mentioned in the playbook.yaml before running the playbook.
```yaml
cat playbook.yaml 
---
- name: Configure Bastion
  hosts: all
  become: yes
  vars_prompt:
    - name: username
      prompt: "Enter the rhsm username"
      private: no
    - name: password
      prompt: "Enter the rhsm password"
      private: yes
  roles:
    - role: ansible-role-bastion-full-automated
```
12. Also verify that your pull-secret and bastion_id_rsa.pub files are present at the following location.
```bash
ls -l ~/pull-secret ~/bastion_id_rsa.pub 
-rw-r--r--. 1 user user 574 Jul 19 22:46 /home/karan/bastion_id_rsa.pub
-rw-r--r--. 1 user user  24 Jul 19 22:47 /home/karan/pull-secret

```
13. Now run the ansible-role from the controller node.
```bash
cd ansible-role-bastion; ansible-playbook playbook.yaml
```
14. First it will ask for the username and password to registry your bastion node with Red Hat to get access to the Red Hat repositories.
15 you already have access to a yum repository, either by local dvd or a third party repository you can provide any dummy username and password when asked.
```bash
Enter the rhsm username:
Enter the rhsm password: 
```
16. It will do the compelte automation and create the ingition files on the bastion node.
```bash
ls -l ~/ocpinstall
ls -l /var/www/html/ocp4

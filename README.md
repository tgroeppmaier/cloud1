# cloud1
automated deployment of wordpress, mysql and caddy on a remote server using ansible


## Steps for ansible

- update system
- install utilites
- Install Docker GPG key
- Add docker APT repo
- Install docker and compose
- Start docker service and on boot
- 

## Useful commands

Check ports of host `nmap -Pn -p 22,80,443,3306,8080 YOUR.IP`

Check Docker service `systemctl status docker`

check running services `systemctl --type=service --state=running`

create random base 16 pw `openssl rand -hex 16`

remove key from known hosts `ssh-keygen -R YOUR_VPS_IP`

add remote key to known hosts `ssh-keyscan -H YOUR_VPS_IP >> ~/.ssh/known_hosts`


## Ansible 

Use Ansible from python virtual environment
`cd cloud1/ansible
python3 -m venv .venv
source .venv/bin/activate
pip install ansible
ansible-galaxy collection install community.docker`

checking connection `ansible -i ansible/inventory.ini cloud1 -m ping`
run playbook with vault pw `ansible-playbook -i inventory.ini site.yml --ask-vault-pass`

create vault `ansible-vault create vault.yml`
view vault `ansible-vault view vault.yml`
edit vault `ansible-vault edit vault.yml`





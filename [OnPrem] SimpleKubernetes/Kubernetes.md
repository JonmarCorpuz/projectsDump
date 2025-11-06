# Partie 1: Mise en place de service de déploiement (manuel ou automatique) de l’application WordPress sur un cluster Kubernetes

## Installer un cluster Kubernetes avec un master et deux workers nodes

1. Définir les hostnames dans le master node et les worker nodes

```Bash
# Dans le master node
sudo hostnamectl set-hostname "master.victim.local"
```

```Bash
# Dans le worker1 node
sudo hostnamectl set-hostname "worker1.victim.local"
```

```Bash
# Dans le worker2 node
sudo hostnamectl set-hostname "worker2.victim.local"
```

2. Ajouter les hostnames dans le fichier /etc/hosts pour la resolution des nom

```Bash
# Dans le master node et les worker nodes
NODE_IP control.victim.local control
NODE_IP worker1.victim.local worker1 
NODE_IP worker2.victim.local worker2 
```

3. Désactiver l'espace d'échange sur tout les nodes (master et worker nodes)

```Bash
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

4. Charger les modules nécessaires pour containerd sur tout les nodes

```Bash
sudo tee /etc/modules-load.d/containerd.conf <<EOF
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter
```

5. Configurer les paramètres sysctl pour Kubernetes sur tout les nodes

```Bash
sudo tee /etc/sysctl.d/kubernetes.conf <<EOT
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOT

sudo sysctl --system
```

6. Installer les paquets nécessaires

```Bash
sudo apt install -y curl gnupg2 software-properties-common apt-transport-https ca-certificates
```

7. Ajouter le dépôt Docker et installer containerd sur tout les nodes

```Bash
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/trusted.gpg.d/docker.gpg
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"

sudo apt update
sudo apt install -y containerd.io
```

8. Configurer activer containerd sur tout les nodes

```Bash
containerd config default | sudo tee /etc/containerd/config.toml >/dev/null 2>&1
sudo sed -i 's/SystemdCgroup \= false/SystemdCgroup \= true/g' /etc/containerd/config.toml

sudo systemctl restart containerd
sudo systemctl enable containerd
```

9. Installer kubelet, kubeadm, et kubectl sur tout les nodes

```Bash
sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

10. Initialiser le master node pour le Kubernetes cluster

```Bash
# Dans le master node 
sudo kubeadm init --control-plane-endpoint=k8smaster.example.net
```

![](/Screenshots/Labo9-1.png)

11. Configurer kubectl pour l'utilisateur local dans le master node

```Bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

12. Vérifier l'état du master node

```Bash
kubectl cluster-info
kubectl get nodes
```

![](/Screenshots/Labo9-2.png)

13. Joindre les worker nodes au cluster

```Bash
# Run ca dans chacun des worker nodes
kubeadm join control.victim.local:6443 --token TOKEN \
        --discovery-token-ca-cert-hash sha256:CLUSTER_HASH
```

![](/Screenshots/Labo9-4.png)

![](/Screenshots/Labo9-6.png)

14. Déployer Calico pour la gestion du réseau

```Bash
# Déployer sur chaque node
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.0/manifests/calico.yaml
```

![](/Screenshots/Labo9-7.png)

15. Vérifiez que tout les nodes sont bien READY et que les pods sont dans un RUNNING state apres avoir installé Calico dans le cluster

```Bash
# Run cette commande dans n'importe quel node 
kubectl get pods -n kube-system
kubectl get nodes
```

![](/Screenshots/Labo9-9.png)

## Tester le fonctionnement de votre cluster en déployant par exemple l’application nginx sur les deux nœuds de votre cluster

1. Définissez un storageClass

```YAML
#storageClass.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-storage
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer

```

2. Créez le storageClass

```Bash
kubectl apply -f storageClass.yaml
```

3. Créez des répertoires de stockage sur tous les nodes 

```Bash
sudo mkdir -p /mnt/data/mysql /mnt/data/wordpress
sudo chmod 777 /mnt/data/mysql /mnt/data/wordpress
```

4. Définissez les volumes persistants

```YAML
#persistantVolume.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mysql-pv
spec:
  capacity:
    storage: 20Gi
  accessModes:
    - ReadWriteOnce
  storageClassName: local-storage
  hostPath:
    path: /mnt/data/mysql
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: wordpress-pv
spec:
  capacity:
    storage: 20Gi
  accessModes:
    - ReadWriteOnce
  storageClassName: local-storage
  hostPath:
    path: /mnt/data/wordpress
```

5. Créez les volumes persistants

```Bash
kubectl apply -f persistantVolume.yaml
```

6. Créez un fichier de kustomization

```YAML
# kustomization.yaml
secretGenerator:
- name: mysql-pass
  literals:
  - password=Password123!
```

7. Créez le déploiement pour MySQL

```YAML
# mysql-deployment.yaml
apiVersion: v1
kind: Service
metadata:
  name: wordpress-mysql
  labels:
    app: wordpress
spec:
  ports:
    - port: 3306
  selector:
    app: wordpress
    tier: mysql
  clusterIP: None
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pv-claim
  labels:
    app: wordpress
spec:
  storageClassName: local-storage
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress-mysql
  labels:
    app: wordpress
spec:
  selector:
    matchLabels:
      app: wordpress
      tier: mysql
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: wordpress
        tier: mysql
    spec:
      containers:
      - image: mysql:8.0
        name: mysql
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-pass
              key: password
        - name: MYSQL_DATABASE
          value: wordpress
        - name: MYSQL_USER
          value: wordpress
        - name: MYSQL_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-pass
              key: password
        ports:
        - containerPort: 3306
          name: mysql
        volumeMounts:
        - name: mysql-persistent-storage
          mountPath: /var/lib/mysql
      volumes:
      - name: mysql-persistent-storage
        persistentVolumeClaim:
          claimName: mysql-pv-claim
```

8. Créez le déploiement pour WordPress

```YAML
# wordpress-deployment.yaml
apiVersion: v1
kind: Service
metadata:
  name: wordpress
  labels:
    app: wordpress
spec:
  ports:
    - port: 80
  selector:
    app: wordpress
    tier: frontend
  type: LoadBalancer
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: wp-pv-claim
  labels:
    app: wordpress
spec:
  storageClassName: local-storage
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress
  labels:
    app: wordpress
spec:
  selector:
    matchLabels:
      app: wordpress
      tier: frontend
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: wordpress
        tier: frontend
    spec:
      containers:
      - image: wordpress:6.2.1-apache
        name: wordpress
        env:
        - name: WORDPRESS_DB_HOST
          value: wordpress-mysql
        - name: WORDPRESS_DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-pass
              key: password
        - name: WORDPRESS_DB_USER
          value: wordpress
        ports:
        - containerPort: 80
          name: wordpress
        volumeMounts:
        - name: wordpress-persistent-storage
          mountPath: /var/www/html
      volumes:
      - name: wordpress-persistent-storage
        persistentVolumeClaim:
          claimName: wp-pv-claim
```

9. Mettez à jour le fichier de kustomization

```Bash
echo """
resources:
  - mysql-deployment.yaml
  - wordpress-deployment.yaml
""" >> ./kustomization.yaml
```

10. Déployez les déploiements et vérifiez les

```Bash
# Déployer les déploiements
kubectl apply -k ./

# Vérifier les déploiements
kubectl get storageclass
kubectl get pv
kubectl get pvc
kubectl get pods
kubectl get svc wordpress
```

11. Créer le script à éxécuter sur chaque worker nodes et le rendre disponible

**Le script à copier**
```Bash
#!/bin/sh
echo "Hello from $(hostname)" > /scripts/demoLog.txt
```

**Rendre le script disponible via un server HTTP pour que le DaemonSet soit capable de le chercher**
```Python
sudo python3 -m http.server 80
```

12. Créez et déployez le DaemonSet qui va copier un scrip sur tout les worker nodes et les éxécuter

**Créez le DaemonSet**
```YAML
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: script-runner
  labels:
    app: script-runner
spec:
  selector:
    matchLabels:
      app: script-runner
  template:
    metadata:
      labels:
        app: script-runner
    spec:
      containers:
      - name: runner
        image: alpine:latest
        command: ["/bin/sh"]
        args: ["-c", "wget -O /tmp/demoScript.sh http://MASTER_NODE_IP/demoScript.sh && chmod +x /tmp/demoScript.sh && /tmp/demoScript.sh"]
        volumeMounts:
        - name: shared-scripts
          mountPath: /scripts
      volumes:
      - name: shared-scripts
        hostPath:
          path: /mnt/scripts
          type: DirectoryOrCreate
```

**Déployez le DaemonSet**
```Bash
kubectl apply -f script-runner.yaml
```

13. Vérification du output du script sur les worker nodes

```Bash
cat /mnt/scripts/demoLog.txt
```

## Documentations

* [DaemonSet](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/)

# Partie 2: Ansible, Kubernetes et Vagrant pour déployer une architecture de VM comprenant un serveur Active directory, un serveur de messagerie, des clients Windows, Linux. Ce déploiement doit se faire de façon automatique

1. Créez le Vagrantfile

```Ruby
Vagrant.configure("2") do |config|
   # Configuration commune
   config.vm.provider "virtualbox" do |vb|
      vb.cpus = 2
      vb.memory = 2048
      vb.linked_clone = true
   end

   # Configuration du serveur AD
   config.vm.define "ad" do |ad|
      ad.vm.box = "gusztavvargadr/windows-server-2019-standard"
      ad.vm.hostname = "ad"
      ad.vm.network "private_network", ip: "192.168.10.100"
      ad.vm.provider "virtualbox" do |vb|
         vb.gui = true
      end
      
      # Configuration WinRM pour AD
      ad.vm.communicator = "winrm"
      ad.winrm.host = "127.0.0.1"
      ad.winrm.transport = "plaintext"
      ad.winrm.basic_auth_only = true
      ad.winrm.username = "vagrant"
      ad.winrm.password = "vagrant"

      # Script PowerShell pour WinRM
      ad.vm.provision "shell", inline: <<-SHELL
         Set-ExecutionPolicy Bypass -Scope Process -Force
         winrm quickconfig -q
         winrm set winrm/config/winrs '@{MaxMemoryPerShellMB="2048"}'
         winrm set winrm/config '@{MaxTimeoutms="1800000"}'
         winrm set winrm/config/service '@{AllowUnencrypted="true"}'
         winrm set winrm/config/service/auth '@{Basic="true"}'
         Start-Service WinRM
         Set-Service WinRM -StartupType Automatic
         netsh advfirewall firewall add rule name="WinRM HTTP" dir=in localport=5985 protocol=TCP action=allow
      SHELL

      # Ansible pour AD
      ad.vm.provision "ansible" do |ansible|
         ansible.playbook = "playbook-ansible-ad.yml"
         ansible.compatibility_mode = "2.0"
         ansible.host_vars = {
            "ad" => {
               "ansible_user" => "vagrant",
               "ansible_password" => "vagrant",
               "ansible_connection" => "winrm",
               "ansible_winrm_transport" => "basic",
               "ansible_port" => 55985,
               "ansible_host" => "127.0.0.1",
               "ansible_winrm_scheme" => "http",
               "ansible_winrm_server_cert_validation" => "ignore"
            }
         }
      end
   end

   # Configuration du serveur Ubuntu
   config.vm.define "ubuntu" do |ubuntu|
      ubuntu.vm.box = "ubuntu/jammy64"
      ubuntu.vm.hostname = "mail-server"
      
      # Configuration réseau pour Ubuntu
      ubuntu.vm.network "private_network", ip: "192.168.10.101"

      # Ansible provisioning for mail server
      ubuntu.vm.provision "ansible" do |ansible|
         ansible.playbook = "playbook-ansible-mail.yml"
         ansible.compatibility_mode = "2.0"
         ansible.host_vars = {
            "ubuntu" => {
               "ansible_user" => "vagrant",
               "ansible_password" => "vagrant",
               "ansible_port" => "2200",
               "ansible_host" => "127.0.0.1",
               "ansible_become" => true
            }
         }
      end
   end

      # Configuration du Clien Ubuntu
   config.vm.define "client-linux" do |ubuntu|
      ubuntu.vm.box = "fasmat/ubuntu2204-desktop"
      ubuntu.vm.hostname = "client-linux"

      # Configuration réseau pour Ubuntu
      ubuntu.vm.network "private_network", ip: "192.168.10.102"

      # Ansible provisioning for mail server
      ubuntu.vm.provision "ansible" do |ansible|
         ansible.playbook = "playbook-ansible-client-linux.yml"
         ansible.compatibility_mode = "2.0"
         ansible.host_vars = {
            "client-linux" => {
               "ansible_user" => "vagrant",
               "ansible_password" => "vagrant",
	           "ansible_port" => "2201",
               "ansible_host" => "127.0.0.1",
               "ansible_become" => true
            }
         }
      end
   end

    # Configuration du Clien WIndows
   config.vm.define "win10-client" do |win|
      win.vm.box = "StefanScherer/windows_10"
      win.vm.hostname = "win10-client"
      win.vm.network "private_network",
         ip: "192.168.56.40",
         adapter: 2,
         netmask: "255.255.255.0",
         auto_config: true

      win.vm.provider "virtualbox" do |vb|
         vb.gui = true
         vb.memory = "4096"
         vb.cpus = 2
      end

      # Configuration WinRM
      win.vm.communicator = "winrm"
      win.winrm.host = "127.0.0.1"
      win.winrm.transport = "plaintext"
      win.winrm.basic_auth_only = true
      win.winrm.username = "vagrant"
      win.winrm.password = "vagrant"

      # Script PowerShell pour WinRM et configuration réseau
      win.vm.provision "shell", inline: <<-SHELL
         # Set network to Private
         Set-NetConnectionProfile -NetworkCategory Private

         # Configuration WinRM
         Set-ExecutionPolicy Bypass -Scope Process -Force
         winrm quickconfig -q -force
         winrm set winrm/config/winrs '@{MaxMemoryPerShellMB="2048"}'
         winrm set winrm/config '@{MaxTimeoutms="1800000"}'
         winrm set winrm/config/service '@{AllowUnencrypted="true"}'
         winrm set winrm/config/service/auth '@{Basic="true"}'
         Start-Service WinRM
         Set-Service WinRM -StartupType Automatic
         netsh advfirewall firewall add rule name="WinRM HTTP" dir=in localport=5985 protocol=TCP action=allow

         # Configure network adapter
         $adapter = Get-NetAdapter | Where-Object {$_.Status -eq "Up" -and $_.InterfaceDescription -like "*VirtualBox*"} | Select-Object -Last 1
         if ($adapter) {
            Remove-NetIPAddress -InterfaceIndex $adapter.ifIndex -Confirm:$false -ErrorAction SilentlyContinue
            New-NetIPAddress -InterfaceIndex $adapter.ifIndex -IPAddress "192.168.56.40" -PrefixLength 24
            Set-DnsClientServerAddress -InterfaceIndex $adapter.ifIndex -ServerAddresses "192.168.56.10"
         }
      SHELL

      # Ansible provisioning
      win.vm.provision "ansible" do |ansible|
         ansible.playbook = "playbook-ansible-win10.yml"
         ansible.compatibility_mode = "2.0"
         ansible.host_vars = {
            "win10-client" => {
               "ansible_user" => "vagrant",
               "ansible_password" => "vagrant",
               "ansible_connection" => "winrm",
               "ansible_winrm_transport" => "basic",
               "ansible_port" => 2202,  # Fixed port instead of dynamic
               "ansible_host" => "127.0.0.1",
               "ansible_winrm_scheme" => "http",
               "ansible_winrm_server_cert_validation" => "ignore"
            }
         }
      end
   end

end
```

2. Créez le ansible playbook pour l'AD 

```YAML
---
- name: Create new Active-Directory Domain & Forest
  hosts: ad  # Changed from localhost to match Vagrant config
  vars:
    temp_address: 127.0.0.1  # Changed to match Vagrant's default address
    dc_address: 192.168.10.100 
    dc_netmask_cidr: 24
    dc_gateway: 192.168.10.1  # Adjust to your network
    dc_hostname: 'victim'
    domain_name: "victim.local"
    local_admin: 'vagrant'  # Changed to match Vagrant user
    temp_password: 'vagrant'  # Changed to match Vagrant password
    dc_password: 'P@ssw0rd'
    recovery_password: 'P@ssw0rd'
    upstream_dns_1: 8.8.8.8
    upstream_dns_2: 8.8.4.4
    reverse_dns_zone: "192.168.0.0/24"  # Adjust to your network
    ntp_servers: "0.us.pool.ntp.org,1.us.pool.ntp.org"
  gather_facts: no

  tasks:
    # No need for initial add_host task as Vagrant handles initial connection

    - name: Set static IP address
      win_shell: "(new-netipaddress -InterfaceAlias Ethernet0 -IPAddress {{ dc_address }} -prefixlength {{dc_netmask_cidr}} -defaultgateway {{ dc_gateway }})"
      ignore_errors: True 

    - name: Set Password
      win_user:
        name: administrator
        password: "{{dc_password}}"
        state: present
      ignore_errors: True  

    - name: Set upstream DNS server 
      win_dns_client:
        adapter_names: '*'
        ipv4_addresses:
        - '{{ upstream_dns_1 }}'
        - '{{ upstream_dns_2 }}'

    - name: Stop the time service
      win_service:
        name: w32time
        state: stopped

    - name: Set NTP Servers
      win_shell: 'w32tm /config /syncfromflags:manual /manualpeerlist:"{{ntp_servers}}"'

    - name: Start the time service
      win_service:
        name: w32time
        state: started  

    - name: Disable firewall for Domain, Public and Private profiles
      win_firewall:
        state: disabled
        profiles:
        - Domain
        - Private
        - Public

    - name: Change the hostname 
      win_hostname:
        name: '{{ dc_hostname }}'
      register: res

    - name: Reboot
      win_reboot:
      when: res.reboot_required   

    - name: Install Active Directory
      win_feature:
        name: AD-Domain-Services
        include_management_tools: yes
        include_sub_features: yes
        state: present
      register: result

    - name: Create Domain
      win_domain:
        dns_domain_name: '{{ domain_name }}'
        safe_mode_password: '{{ recovery_password }}'
      register: ad

    - name: reboot server
      win_reboot:
        msg: "Installing AD. Rebooting..."
        pre_reboot_delay: 15
      when: ad.changed

    - name: Set internal DNS server 
      win_dns_client:
        adapter_names: '*'
        ipv4_addresses:
        - '127.0.0.1'

    - name: Create reverse DNS zone
      win_shell: "Add-DnsServerPrimaryZone -NetworkID {{reverse_dns_zone}} -ReplicationScope Forest"
      retries: 30
      delay: 60
      register: result           
      until: result is succeeded
```

3. Créez le ansible playbook pour le mail serveur

```YAML
---
- name: Setup Postfix and Dovecot with AD + Local Authentication and Join AD Domain
  hosts: ubuntu
  become: yes
  vars:
    mail_domain: victim.local
    ad_server: 192.168.10.101
    ad_base_dn: "dc=victim,dc=local"
    ad_bind_dn: "CN=Administrator,CN=Users,DC=victim,DC=local"
    ad_bind_pw: "P@ssw0rd"
    local_mail_password: "P@ssw0rd"  # Password for local test user
    domain_name: victim.local
    admin_user: Administrator
    admin_password: P@ssw0rd

  tasks:
    - name: Update apt cache
      apt:
        update_cache: yes

    - name: Install required packages
      apt:
        name:
          - realmd
          - sssd
          - sssd-tools
          - libnss-sss
          - libpam-sss
          - adcli
          - samba-common-bin
          - oddjob
          - oddjob-mkhomedir
          - packagekit
          - chrony
          - postfix
          - dovecot-imapd
          - dovecot-ldap
          - postfix-ldap
          - python3-passlib
          - dovecot-lmtpd
        state: present

    - name: Create mail group
      group:
        name: vmail
        gid: 5000
        state: present

    - name: Create mail user
      user:
        name: vmail
        uid: 5000
        group: vmail
        home: /var/mail
        shell: /sbin/nologin
        state: present

    - name: Create test local mail user
      user:
        name: testmail
        password: "P@ssw0rd"
        shell: /sbin/nologin
        state: present

    - name: Create mail directory
      file:
        path: /var/mail/vhosts/victim.local
        state: directory
        mode: '0775'
        owner: vmail
        group: vmail
        recurse: yes

    - name: Configure DNS resolution
      lineinfile:
        path: /etc/resolv.conf
        line: "nameserver {{ ad_server }}"
        insertbefore: BOF

    - name: Configure Postfix main.cf
      template:
        src: main.cf.j2
        dest: /etc/postfix/main.cf
      notify: restart postfix

    - name: Configure Dovecot master config
      template:
        src: 10-master.conf.j2
        dest: /etc/dovecot/conf.d/10-master.conf
      notify: restart dovecot

    - name: Configure Dovecot auth
      template:
        src: 10-auth.conf.j2
        dest: /etc/dovecot/conf.d/10-auth.conf
      notify: restart dovecot

    - name: Configure Dovecot LDAP
      template:
        src: dovecot-ldap.conf.ext.j2
        dest: /etc/dovecot/conf.d/auth-ldap.conf.ext
      notify: restart dovecot

    - name: Configure Dovecot mail location
      template:
        src: 10-mail.conf.j2
        dest: /etc/dovecot/conf.d/10-mail.conf
      notify: restart dovecot

    - name: Create Dovecot password file
      template:
        src: passwd.j2
        dest: /etc/dovecot/passwd
        mode: '0600'
        owner: dovecot
        group: dovecot

    - name: Set correct permissions for postfix private directory
      file:
        path: /var/spool/postfix/private
        state: directory
        owner: postfix
        group: postfix
        mode: '0755'

    - name: Configure chrony for AD time sync
      template:
        src: chrony.conf.j2
        dest: /etc/chrony/chrony.conf
      notify: restart chrony

    - name: Enable home directory creation
      command: pam-auth-update --enable mkhomedir
      changed_when: false

    - name: Enable home directory creation
      command: pam-auth-update --enable mkhomedir
      changed_when: false

    - name: Check domain join status
      command: realm list
      register: realm_status
      changed_when: false
      failed_when: false

    - name: Join AD domain
      expect:
        command: realm join -U Administrator victim.local
        responses:
          "Password for Administrator:": "{{ ad_bind_pw }}"
      when: realm_status.rc != 0 or "victim.local" not in realm_status.stdout
      register: join_result
      failed_when: false


  handlers:
    - name: restart postfix
      service:
        name: postfix
        state: restarted

    - name: restart dovecot
      service:
        name: dovecot
        state: restarted

    - name: restart chrony
      service:
        name: chronyd
        state: restarted

    - name: restart sssd
      service:
        name: sssd
        state: restarted
```

4. Créez les fichiers src pour les services dans le mail server

**10-auth.conf.j2**
```Text
disable_plaintext_auth = no
auth_mechanisms = plain login

# Local users
passdb {
  driver = passwd-file
  args = /etc/dovecot/passwd
}

# AD users
passdb {
  driver = ldap
  args = /etc/dovecot/conf.d/auth-ldap.conf.ext
}

userdb {
  driver = static
  args = uid=vmail gid=vmail home=/var/mail/vhosts/%d/%n
}
```

**10-mail.conf.j2**
```Text
mail_location = maildir:/var/mail/vhosts/%d/%n
mail_privileged_group = mail

namespace inbox {
  inbox = yes
  location = maildir:/var/mail/vhosts/%d/%n
  mailbox Sent {
    auto = subscribe
    special_use = \Sent
  }
  mailbox Drafts {
    auto = subscribe
    special_use = \Drafts
  }
  mailbox Trash {
    auto = subscribe
    special_use = \Trash
  }
}

protocols = imap lmtp
```

**10-master.conf.j2**
```Text
service imap-login {
  inet_listener imap {
    port = 143
  }
}

service auth {
  unix_listener /var/spool/postfix/private/auth {
    mode = 0660
    user = postfix
    group = postfix
  }
}

service lmtp {
  unix_listener /var/spool/postfix/private/dovecot-lmtp {
    mode = 0600
    user = postfix
    group = postfix
  }
}
```

**dovecot-ldap.conf.ext.j2**
```Text
hosts = {{ ad_server }}
auth_bind = yes
ldap_version = 3
base = {{ ad_base_dn }}

# Bind DN
dn = {{ ad_bind_dn }}
dnpass = {{ ad_bind_pw }}

# Authentication - using exact DN format
auth_bind_userdn = CN=%n,CN=Users,DC=demo,DC=lab

# Search filters
user_filter = (sAMAccountName=%n)
pass_filter = (sAMAccountName=%n)

# Debug
debug_level = 2
```

**main.cf.j2**
```Text
myhostname = mail.{{ mail_domain }}
mydomain = {{ mail_domain }}
myorigin = $mydomain
inet_interfaces = all
mydestination = $myhostname, localhost.$mydomain, localhost

# LDAP configuration
mailbox_transport = lmtp:unix:/var/spool/postfix/private/dovecot-lmtp
virtual_transport = lmtp:unix:private/dovecot-lmtp
virtual_mailbox_domains = victim.local

# Authentication
smtpd_sasl_type = dovecot
smtpd_sasl_path = private/auth
smtpd_sasl_auth_enable = yes
smtpd_recipient_restrictions = permit_sasl_authenticated,permit_mynetworks,reject_unauth_destination
```

**passwd.j2**
```Text
testmail@{{ mail_domain }}:{PLAIN}{{ local_mail_password }}
```

**chrony.conf.j2**
```Text
server {{ ad_server }} iburst
```

5. Créez le ansible playbook pour le client Ubuntu

```YAML
---
- name: AD Domain Join for Linux
  hosts: client-linux
  become: yes
  vars:
    mail_domain: victim.local
    ad_server: 192.168.10.100
    ad_base_dn: "dc=victim,dc=local"
    ad_bind_dn: "CN=Administrator,CN=Users,DC=victim,DC=local"
    ad_bind_pw: "P@ssw0rd"
    local_mail_password: "P@ssw0rd"  # Password for local test user
    domain_name: victim.local
    admin_user: Administrator
    admin_password: P@ssw0rd

  tasks:
    - name: Update apt cache
      apt:
        update_cache: yes

    - name: Install required packages
      apt:
        name:
          - realmd
          - sssd
          - sssd-tools
          - libnss-sss
          - libpam-sss
          - adcli
          - samba-common-bin
          - oddjob
          - oddjob-mkhomedir
          - packagekit
          - chrony

    - name: Configure DNS resolution
      lineinfile:
        path: /etc/resolv.conf
        line: "nameserver {{ ad_server }}"
        insertbefore: BOF

    - name: Enable home directory creation
      command: pam-auth-update --enable mkhomedir
      changed_when: false

    - name: Enable home directory creation
      command: pam-auth-update --enable mkhomedir
      changed_when: false

    - name: Check domain join status
      command: realm list
      register: realm_status
      changed_when: false
      failed_when: false

    - name: Join AD domain
      expect:
        command: realm join -U Administrator victim.local
        responses:
          "Password for Administrator:": "{{ ad_bind_pw }}"
      when: realm_status.rc != 0 or "victim.local" not in realm_status.stdout
      register: join_result
      failed_when: false


  handlers:
    - name: restart chrony
      service:
        name: chronyd
        state: restarted

    - name: restart sssd
      service:
        name: sssd
        state: restarted
```

6. Créez le ansible playbook pour le client Windows

```YAML
---
- name: AD Domain Join for Windows
  hosts: win10-client
  vars:
    domain_name: victim.local
    ad_server: 192.168.10.100
    admin_user: Administrator@victim.local
    admin_password: P@ssw0rd

  tasks:
    - name: Configure DNS settings
      win_dns_client:
        adapter_names: '*'
        ipv4_addresses:
          - "{{ ad_server }}"

    - name: Ensure hostname is set
      win_hostname:
        name: win10-client
      register: hostname_result

    - name: Join computer to domain
      win_domain_membership:
        dns_domain_name: "{{ domain_name }}"
        domain_admin_user: "{{ admin_user }}"
        domain_admin_password: "{{ admin_password }}"
        state: domain
      register: domain_join
      notify: reboot_after_join

  handlers:
    - name: reboot_after_join
      win_reboot:
        msg: "Rebooting after domain join"
        pre_reboot_delay: 15
        post_reboot_delay: 60
```

7. Déployer l'architecture

```Bash
vagrant up
```

# Partie 3: Initiative personelle de recherche et ajouts sur les travaux partie 1 et 2

* [DaemonSet](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/)
* [Déployer AD Avec Vagrant](https://www.it-connect.fr/deployer-un-domaine-active-directory-avec-vagrant-et-ansible/)
* [Vagrant Documentation](https://developer.hashicorp.com/vagrant/docs)
* [Ansible Documentation](https://docs.ansible.com/)
* [Kubernetes Documentation](https://kubernetes.io/docs/home/)

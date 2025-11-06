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

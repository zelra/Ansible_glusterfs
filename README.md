# Déploiement rapide de cluster Glusterfs avec Ansible
-------------------------------------------

 Ce "Playbook" est composé de 5 roles : "glusterfs_esclave", "glusterfs_maitre", " setup", "ucarp_esclave" et "ucarp_maitre"

- "setup" permet d'installer quelques packages de base sur les machines distantes
- "glusterfs_esclave" et "glusterfs_maitre" permettent d'installer et de configurer glusterfs sur les machines distantes
- "ucarp_esclave" et "ucarp_maitre" permettent d'installer et de configurer ucarp sur les machines distantes

Dans Glusterfs il n'y pas de status maitre/esclave. 
Le role "glusterfs_maitre" permet juste de lancer en plus de l'installation les commandes pour lier les machines et créer le volume de partage
Il y a 2 roles "ucarp" car la configuration n'est pas la même sur toutes les machines : une doit être maitre.

J'aurais pu regrouper les roles et exécuter les actions séparément avec "hosts: maitre" ou "hosts: esclave" mais je trouve ça plus clair comme ça.

Ce playbook a été développé et testé sur VMWare. Il se peut qu'il faille faire des modifications pour l'utiliser sur des vrais machines.
Toutes les machines étaient sur "Debian jessie 8.9".
La version de Glsuterfs utilisée est " 3.5.2".

### Pré-requis
-------------------------------------------
Pour utiliser ce playbook il nous faut:
- 5 machines "glusterfs" (qui serviront pour le cluster)
- 1 machine "administration" (qui exécutera Ansible)
- 1 machine "client" (qui montera sur une de ses partitions le volume glusterfs partagé par le cluster)

Dans mon cas la machine cliente et la machine d'administration sont confondues

TOUTES les machines doivent avoir: 
- une adresse ip valable
- avoir son fichier /etc/hosts renseigné avec tous les hotes du cluster
- avoir un nom propre à la machine 
- avoir le package openssh-server d'installé

Dans mon cas je me connecte en root sur les machines. Je vous conseille donc de modifier le fichier:

    /etc/ssh/sshd_config
    
Et modifier la ligne:

    PermitRootLogin without-password
    
En:

    PermitRootLogin yes
    

### Mise en place (administration)
-------------------------------------------

Pour utiliser ce playbook il faut commencer par configurer son dns local:
 
   nano /etc/hosts

Puis rentrer les IPs de vos machines distantes:

    192.168.220.10 glusterfs1
    192.168.220.20 glusterfs2
    192.168.220.30 glusterfs3
    192.168.220.40 glusterfs4
    192.168.220.50 glusterfs5
    192.168.220.100 glusterfsVIP

Après cela il faut générer les clefs pour pouvoir se connecter sur les machines en SSH:

    ssh-keygen -t rsa -b 4096

Puis envoyer les clefs aux machines distantes:

    ssh-copy-id -i ~/.ssh/id_rsa.pub root@ma_machine_distante
    
Pensez à l'exécuter sur toutes vos machines présentes dans le " /etc/hosts ".

### Installation du cluster (administration)
-------------------------------------------
Sur la machine d'administration, téléchargez Ansible:

    apt-get install ansible
    
Placez ensuite le playbook "Ansible_glusterfs" dans le fichier:

    /etc/ansible/
    
Vérifiez que les hotes sont bien configurés sur ansible puis palcez vous dans le playbook:

    cd /etc/ansible/Ansible_glusterfs
    
 Puis lancez le déploiement des serveur avec la commande suivante:
 
    ansible-playbook server.yml

Cela prend un certains temps.

### Montage de la partition du glusterfs (client)
-------------------------------------------
Une fois que le déploiement s'est bien effectué, créez sur le client un répertoire pour monter la partition:

    mkdir /srv/shared
    
Installez ensuite le glusterfs-client:

    apt-get isntall glusterfs-client
    
Montez ensuite la partition sur votre répertoire créé:

    mount -t glusterfs glusterfsVIP:/volume1 /srv/shared
    
Testez le cluster en créant un fichier sur le client puis en vérifiant sur le cluster que le fichier est bien dupliqué.
Une fois testé, vous pouvez rentrer la configuration du montage sur le /etc/fstab pour que la connexion au cluster perdure après le reboot de la machine.

### A noter 
-------------------------------------------
Dans un glusterfs, on utilise en général (par machine) un périphérique de stockage dédié qu'au Glusterfs.
Il faut donc pour cela le monter sur une partition puis appeller cette partition avec Glusterfs.
Dans mon cas j'ai choisi (par simplicité) de créer un dossier puis de la partager :

     - name: Création du fichier de partage
       file: path=/srv/shared state=directory

Il faut donc modifier le playbook pour ajouter un périphérique dédié.


De plus l'IP sur mon réseau local de VM était 192.168.220.x . Il faut aussi que vous l'adaptiez à votre configuration.
De même pour les templates des roles "ucarp_esclave" et ucarp_maitre" (la configuration des template est en 192.168.220.x).

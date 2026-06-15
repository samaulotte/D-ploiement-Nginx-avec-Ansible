# Déploiement Nginx avec Ansible

Voici une version propre adaptée à mon lab Ansible.

## Contexte du lab

```text
Machine de contrôle : toto@ansible
Client cible        : client-nginx
IP client           : 172.16.93.172
User SSH distant    : ansible
Sudo                : sans mot de passe
SSH                 : par clé
Inventaire          : /home/toto/ansible/inventories/hosts.ini
```

Ce projet utilise uniquement des modules `ansible.builtin.*`, inclus dans `ansible-core`.

Modules utilisés :

```text
ansible.builtin.apt
ansible.builtin.file
ansible.builtin.template
ansible.builtin.service
ansible.builtin.ping
```

La documentation officielle Ansible recommande l’utilisation des FQCN, par exemple `ansible.builtin.apt`, `ansible.builtin.service` et `ansible.builtin.template`, afin d’éviter les conflits de noms avec d’autres collections.

Documentation officielle Ansible :

```text
https://docs.ansible.com/projects/ansible/latest/index.html
```

---

## 1. Arborescence recommandée

Dans `/home/toto/ansible`, garder cette structure :

```text
/home/toto/ansible/
├── ansible.cfg
├── inventories/
│   └── hosts.ini
├── playbooks/
│   └── nginx.yml
└── roles/
    └── nginx/
        ├── defaults/
        │   └── main.yml
        ├── handlers/
        │   └── main.yml
        ├── tasks/
        │   └── main.yml
        └── templates/
            ├── default.conf.j2
            └── index.html.j2
```

Commande pour créer les dossiers :

```bash
mkdir -p /home/toto/ansible/{inventories,playbooks}
mkdir -p /home/toto/ansible/roles/nginx/{defaults,handlers,tasks,templates}
```

Commande pour créer les fichiers :

```bash
touch /home/toto/ansible/ansible.cfg
touch /home/toto/ansible/inventories/hosts.ini
touch /home/toto/ansible/playbooks/nginx.yml
touch /home/toto/ansible/roles/nginx/defaults/main.yml
touch /home/toto/ansible/roles/nginx/tasks/main.yml
touch /home/toto/ansible/roles/nginx/handlers/main.yml
touch /home/toto/ansible/roles/nginx/templates/default.conf.j2
touch /home/toto/ansible/roles/nginx/templates/index.html.j2
```

Les rôles Ansible chargent automatiquement les fichiers `tasks`, `handlers`, `defaults`, `templates`, etc., selon l’arborescence officielle des rôles.

---

## 2. Fichier `ansible.cfg`

Chemin :

```text
/home/toto/ansible/ansible.cfg
```

Contenu :

```ini
[defaults]
inventory = ./inventories/hosts.ini
roles_path = ./roles
host_key_checking = False
interpreter_python = auto_silent

[privilege_escalation]
become = True
become_method = sudo
become_ask_pass = False
```

Comme le fichier `ansible.cfg` est placé dans `/home/toto/ansible`, le chemin relatif suivant :

```text
./inventories/hosts.ini
```

pointe vers :

```text
/home/toto/ansible/inventories/hosts.ini
```

Comme l’utilisateur distant `ansible` possède déjà les droits sudo sans mot de passe, la directive suivante est correcte :

```ini
become_ask_pass = False
```

---

## 3. Inventaire `hosts.ini`

Chemin :

```text
/home/toto/ansible/inventories/hosts.ini
```

Contenu :

```ini
[webservers]
client-nginx ansible_host=172.16.93.172 ansible_user=ansible ansible_port=22

[webservers:vars]
ansible_become=true
ansible_become_method=sudo
```

L’inventaire définit les hôtes gérés, les groupes et les variables associées aux hôtes.

Ici, le client est placé dans le groupe :

```text
webservers
```

L’alias Ansible du client est :

```text
client-nginx
```

L’adresse réelle du client est définie avec :

```ini
ansible_host=172.16.93.172
```

---

## 4. Playbook `nginx.yml`

Chemin :

```text
/home/toto/ansible/playbooks/nginx.yml
```

Contenu :

```yaml
---
- name: Déployer Nginx sur le client Ansible
  hosts: webservers
  become: true

  roles:
    - nginx
```

Ce playbook cible le groupe `webservers` défini dans l’inventaire et appelle le rôle `nginx`.

---

## 5. Rôle `nginx`

### 5.1 Fichier `defaults/main.yml`

Chemin :

```text
/home/toto/ansible/roles/nginx/defaults/main.yml
```

Contenu :

```yaml
---
nginx_package_name: nginx
nginx_service_name: nginx

nginx_document_root: /var/www/html
nginx_index_file: /var/www/html/index.html

nginx_site_available: /etc/nginx/sites-available/default
nginx_site_enabled: /etc/nginx/sites-enabled/default

nginx_listen_port: 80
nginx_server_name: "_"
```

Ce fichier contient les variables par défaut du rôle `nginx`.

---

### 5.2 Fichier `tasks/main.yml`

Chemin :

```text
/home/toto/ansible/roles/nginx/tasks/main.yml
```

Contenu :

```yaml
---
- name: Mettre à jour le cache APT
  ansible.builtin.apt:
    update_cache: true
    cache_valid_time: 3600

- name: Installer Nginx
  ansible.builtin.apt:
    name: "{{ nginx_package_name }}"
    state: present

- name: Vérifier que le dossier web existe
  ansible.builtin.file:
    path: "{{ nginx_document_root }}"
    state: directory
    owner: root
    group: root
    mode: "0755"

- name: Déployer la configuration du site Nginx par défaut
  ansible.builtin.template:
    src: default.conf.j2
    dest: "{{ nginx_site_available }}"
    owner: root
    group: root
    mode: "0644"
  notify: Reload nginx

- name: Activer le site Nginx par défaut
  ansible.builtin.file:
    src: "{{ nginx_site_available }}"
    dest: "{{ nginx_site_enabled }}"
    state: link
    force: true
  notify: Reload nginx

- name: Déployer la page d'accueil
  ansible.builtin.template:
    src: index.html.j2
    dest: "{{ nginx_index_file }}"
    owner: root
    group: root
    mode: "0644"
  notify: Reload nginx

- name: Démarrer et activer le service Nginx
  ansible.builtin.service:
    name: "{{ nginx_service_name }}"
    state: started
    enabled: true
```

Le module `ansible.builtin.apt` gère les paquets APT. Il est donc adapté pour installer `nginx` sur une distribution Debian ou Ubuntu.

Le module `ansible.builtin.file` permet de gérer les fichiers, dossiers et liens symboliques.

Le module `ansible.builtin.template` génère un fichier distant depuis un template Jinja2.

Le module `ansible.builtin.service` permet de gérer l’état d’un service, par exemple `started`, `reloaded` ou `enabled`.

---

### 5.3 Fichier `handlers/main.yml`

Chemin :

```text
/home/toto/ansible/roles/nginx/handlers/main.yml
```

Contenu :

```yaml
---
- name: Reload nginx
  ansible.builtin.service:
    name: "{{ nginx_service_name }}"
    state: reloaded
```

Le handler sera appelé uniquement si une tâche avec :

```yaml
notify: Reload nginx
```

provoque un changement.

C’est adapté pour recharger Nginx seulement quand sa configuration ou sa page HTML change.

---

### 5.4 Template Nginx `default.conf.j2`

Chemin :

```text
/home/toto/ansible/roles/nginx/templates/default.conf.j2
```

Contenu :

```nginx
server {
    listen {{ nginx_listen_port }} default_server;
    listen [::]:{{ nginx_listen_port }} default_server;

    root {{ nginx_document_root }};
    index index.html index.htm;

    server_name {{ nginx_server_name }};

    location / {
        try_files $uri $uri/ =404;
    }
}
```

---

### 5.5 Template HTML `index.html.j2`

Chemin :

```text
/home/toto/ansible/roles/nginx/templates/index.html.j2
```

Contenu :

```html
<!DOCTYPE html>
<html lang="fr">
<head>
    <meta charset="UTF-8">
    <title>Nginx déployé avec Ansible</title>
</head>
<body>
    <h1>Nginx fonctionne sur {{ inventory_hostname }}</h1>
    <p>Déploiement réalisé avec Ansible et des modules ansible.builtin.</p>
    <p>Serveur cible : {{ ansible_host | default(inventory_hostname) }}</p>
</body>
</html>
```

---

## 6. Vérifications et lancement

Se placer dans le dossier du projet :

```bash
cd /home/toto/ansible
```

Vérifier l’inventaire :

```bash
ansible-inventory --list
```

Tester la connexion SSH Ansible :

```bash
ansible webservers -m ansible.builtin.ping
```

Résultat attendu :

```text
client-nginx | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

Vérifier la syntaxe du playbook :

```bash
ansible-playbook playbooks/nginx.yml --syntax-check
```

Résultat attendu :

```text
playbook: playbooks/nginx.yml
```

Lancer le déploiement :

```bash
ansible-playbook playbooks/nginx.yml
```

Tester Nginx depuis la machine de contrôle Ansible :

```bash
curl http://172.16.93.172
```

Résultat attendu dans la page HTML :

```html
<h1>Nginx fonctionne sur client-nginx</h1>
```

---

## 7. Commandes utiles de debug

Vérifier l’inventaire :

```bash
ansible-inventory --list
```

Tester la connexion :

```bash
ansible webservers -m ansible.builtin.ping
```

Vérifier la syntaxe :

```bash
ansible-playbook playbooks/nginx.yml --syntax-check
```

Faire un dry-run :

```bash
ansible-playbook playbooks/nginx.yml --check
```

Relancer le déploiement :

```bash
ansible-playbook playbooks/nginx.yml
```

Tester l’accès HTTP :

```bash
curl http://172.16.93.172
```

---

## 8. Vérification côté client

Connexion au client :

```bash
ssh ansible@172.16.93.172
```

Vérifier le service Nginx :

```bash
sudo systemctl status nginx
```

Tester la configuration Nginx :

```bash
sudo nginx -t
```

Tester localement depuis le client :

```bash
curl http://localhost
```

---

## 9. Résultat attendu

À la fin du déploiement, le client doit avoir :

```text
Nginx installé
Service nginx démarré
Service nginx activé au démarrage
Configuration Nginx déployée
Page index.html déployée
Site accessible sur http://172.16.93.172
```

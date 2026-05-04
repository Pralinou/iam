# Load Balancing SSO Keycloak avec Ansible

Déploiement automatisé d'un portail SSO Keycloak en cluster avec load balancing Nginx via Ansible.
Projet personnel pour manipuler Ansible + Docker.

> **Mots de passe par défaut** — Les credentials fournis dans ce dépôt sont des valeurs d'exemple pour un usage local/test.
>
> | Variable | Valeur par défaut | Fichier à modifier |
> |---|---|---|
> | Admin Keycloak | `admin / admin` | `docker-compose.yml` → `KEYCLOAK_ADMIN_PASSWORD` |
> | Base de données | `keycloak_password` | `docker-compose.yml` → `POSTGRES_PASSWORD` / `KC_DB_PASSWORD` |
> | Admin Ansible | `changeme_in_vault` | `inventory/group_vars/all.yml` |
>
> Chiffrement avec [ansible-vault](https://docs.ansible.com/ansible/latest/vault_guide/index.html).

| Conteneur      | Rôle                     | Port |
|----------------|--------------------------|------|
| `iam_lb`       | Nginx – load balancer    | 80   |
| `lb_1`         | Keycloak instance 1      | 8080 |
| `lb_2`         | Keycloak instance 2      | 8080 |
| `lb_3`         | Keycloak instance 3      | 8080 |
| `iam_postgres` | Base de données partagée | 5432 |

## Prérequis

- Docker et Docker Compose installés
- Python 3.x et pip (pour Ansible)

### Installation d'Ansible

```bash
pip3 install ansible --break-system-packages
export PATH="$HOME/.local/bin:$PATH"   # à ajouter aussi dans ~/.bashrc
ansible-galaxy collection install -r requirements.yml
```

## Test en local avec Docker Compose

### 1. Démarrer l'infrastructure

```bash
docker compose up -d
```

Vérifie que tous les conteneurs sont up :
```bash
docker compose ps
```

Les instances Keycloak mettent ~60s à démarrer (initialisation de la base).

### 2. Accéder aux interfaces

**Portail SSO Keycloak** → [http://localhost](http://localhost)

Console d'administration Keycloak → [http://localhost/admin](http://localhost/admin)
- Login : `admin` / Mot de passe : `admin`

### 3. Vérifier le load balancing

```bash
for i in 1 2 3; do
  curl -s http://localhost/health/ready && echo " → OK"
done
```

### 4. Arrêter

```bash
docker compose down          # arrêt sans supprimer les données
docker compose down -v       # arrêt + suppression des volumes
```

---

## Déploiement production avec Ansible

### 1. Configurer l'inventaire

Édite [inventory/hosts.ini](inventory/hosts.ini) avec tes IPs réelles :

```ini
[lb]
loadbalancer ansible_host=192.168.1.10 ansible_user=ubuntu

[app_servers]
app1 ansible_host=192.168.1.11 ansible_user=ubuntu
app2 ansible_host=192.168.1.12 ansible_user=ubuntu
app3 ansible_host=192.168.1.13 ansible_user=ubuntu

[db]
db_server ansible_host=192.168.1.14 ansible_user=ubuntu
```

### 2. Changer les mots de passe

Édite [inventory/group_vars/all.yml](inventory/group_vars/all.yml) :

```yaml
keycloak_admin_password: "mot_de_passe_fort"
db_password: "mot_de_passe_fort"
```

> En production, utilise `ansible-vault encrypt_string` pour chiffrer les secrets.

### 3. Vérifier la connexion SSH

```bash
ansible all -m ping
```

### 4. Déployer

```bash
ansible-playbook playbooks/deploy.yml
```

### Mise à jour sans downtime (rolling update)

```bash
ansible-playbook playbooks/rolling_update.yml
```

### Ajouter des instances (scale out)

Ajoute les hôtes dans `inventory/hosts.ini`, puis :

```bash
ansible-playbook playbooks/scale.yml -e "target_hosts=app4,app5"
```

---

## Reste à faire

**Déploiement des comptes Keycloak**
Créer et configurer les utilisateurs, groupes et rôles dans Keycloak via Ansible pour automatiser la gestion des comptes.

**Samba Active Directory (optionnel)**
Éventuellement déployer un Samba AD et le connecter à Keycloak pour gérer les comptes depuis un domaine Active Directory.

**Centralisation des logs**
Collecter les logs de tous les services (Keycloak, Nginx) dans un outil centralisé pour faciliter la supervision.

**VIP + certificats TLS**
Mettre en place une IP virtuelle pour la haute disponibilité et activer HTTPS sur le load balancer avec un certificat SSL.

---

## Structure du projet

```
.
├── ansible.cfg
├── requirements.yml
├── docker-compose.yml              # stack principale (versionnée)
├── docker/
│   └── nginx.conf                  # config Nginx pour docker-compose
├── roles/
│   ├── docker/                     # installation de Docker
│   ├── postgres/                   # déploiement PostgreSQL
│   ├── app/                        # déploiement Keycloak
│   └── nginx_lb/                   # configuration du load balancer
└── playbooks/
    ├── deploy.yml                  # déploiement complet (production)
    ├── rolling_update.yml          # mise à jour sans downtime
    └── scale.yml                   # ajout de nouveaux noeuds
```


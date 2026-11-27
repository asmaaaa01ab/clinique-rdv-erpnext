# Installation d'ERPNext + Healthcare (Windows + WSL2)

Environnement testé le 27 novembre 2026 :

| Élément | Version |
|---|---|
| Windows | 10/11 avec WSL2 |
| Distribution | Ubuntu 22.04 (`Ubuntu-22.04`) |
| Python | 3.11 |
| Node.js | 18 (via nvm) |
| MariaDB | 10.6 |
| Frappe / ERPNext | branche `version-15` |
| Healthcare | https://github.com/frappe/health |

## 1. Installer Ubuntu 22.04 (PowerShell)
```powershell
wsl --install -d Ubuntu-22.04
wsl -d Ubuntu-22.04
```
Activer systemd dans Ubuntu, puis redémarrer WSL depuis PowerShell :
```bash
sudo bash -c 'printf "[boot]\nsystemd=true\n" > /etc/wsl.conf'
```
```powershell
wsl --shutdown
```

## 2. Dépendances système (Ubuntu)
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y git curl python3-dev python3-pip python3-setuptools python3-venv \
  software-properties-common mariadb-server mariadb-client redis-server xvfb \
  libfontconfig1 libmariadb-dev pkg-config wkhtmltopdf cron

# Python 3.11 (requis par Frappe v15 récent)
sudo add-apt-repository -y ppa:deadsnakes/ppa
sudo apt update
sudo apt install -y python3.11 python3.11-dev python3.11-venv

# Node 18 + yarn
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
source ~/.bashrc
nvm install 18
npm install -g yarn
```

## 3. MariaDB
```bash
sudo mysql_secure_installation
sudo bash -c 'cat >> /etc/mysql/mariadb.conf.d/50-server.cnf <<EOF

[mysqld]
character-set-client-handshake = FALSE
character-set-server = utf8mb4
collation-server = utf8mb4_unicode_ci

[mysql]
default-character-set = utf8mb4
EOF'
sudo systemctl restart mariadb
```
Faire fonctionner l'utilisateur root avec mot de passe (sinon erreur 1698) :
```bash
sudo mysql -e "ALTER USER 'root'@'localhost' IDENTIFIED VIA mysql_native_password USING PASSWORD('MOT_DE_PASSE_ROOT'); FLUSH PRIVILEGES;"
```

## 4. Bench, ERPNext et site
```bash
sudo pip3 install --no-cache-dir --default-timeout=300 --retries 10 frappe-bench
bench init --frappe-branch version-15 --python python3.11 ~/frappe-bench
cd ~/frappe-bench

bench new-site clinique.localhost --admin-password admin --mariadb-root-password MOT_DE_PASSE_ROOT
bench use clinique.localhost
bench get-app --branch version-15 erpnext
```
Ouvrir **un second terminal** et lancer le serveur (nécessaire avant l'installation de l'app) :
```bash
cd ~/frappe-bench
bench start
```
Dans le premier terminal :
```bash
bench --site clinique.localhost install-app erpnext
```

## 5. Module Healthcare et mode développeur
```bash
bench get-app --branch version-15 https://github.com/frappe/health
bench --site clinique.localhost install-app healthcare
bench --site clinique.localhost set-config developer_mode 1
bench --site clinique.localhost clear-cache
```

## 6. Vérification
```bash
bench --site clinique.localhost list-apps     # frappe, erpnext, healthcare
curl -s http://localhost:8000/api/method/ping # {"message":"pong"}
```
Ouvrir http://clinique.localhost:8000 (utilisateur `Administrator`), terminer l'assistant de configuration (société « Clinique Noor », abréviation CN, pays Maroc, devise MAD, **sans** données de démonstration). Le menu **Healthcare** doit apparaître dans la barre latérale.

## 7. Démarrage quotidien
```powershell
wsl -d Ubuntu-22.04
```
```bash
cd ~/frappe-bench
bench start
```

## 8. Problèmes rencontrés et solutions

| Problème | Cause | Solution |
|---|---|---|
| `Temporary failure resolving 'archive.ubuntu.com'` | DNS de WSL | Fixer `/etc/resolv.conf`, `generateResolvConf = false` dans `/etc/wsl.conf`, puis `wsl --shutdown` |
| `frappe depends on Python>=3.11` | Ubuntu 22.04 fournit Python 3.10 | Installer python3.11 et `bench init --python python3.11` |
| `Access denied for user '_xxxx'@'localhost'` (1698) | Authentification `unix_socket` de MariaDB | Utilisateur root en `mysql_native_password`, recréer le site |
| `Error 111 connecting to 127.0.0.1:11000` | Redis de bench non démarré | Lancer `bench start` dans un second terminal avant `install-app` |
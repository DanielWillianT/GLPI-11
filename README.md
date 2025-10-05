# Guia Genérico de Instalação do **GLPI 11** (Debian 12/13 + Hyper‑V)

> **Objetivo**: Passo a passo reproduzível para preparar uma VM Debian 12/13 no Hyper‑V e instalar o GLPI 11 com Apache, MariaDB e PHP 8.2. Todos os valores sensíveis e de rede aparecem como ***placeholders*** para você substituir.

> 🔁 **Como usar**: Substitua tudo que estiver **entre `<>`** pelos seus valores. Ex.: `<IP_VM>` → `192.168.50.100`.

---

## 📦 Pré‑requisitos (decida agora)

* ISO do **Debian 12/13 (amd64)**.
* IP fixo que você **vai usar**:

  * `<IP_VM>` = `192.168.50.100`
  * `<NETMASK>` = `255.255.255.0`
  * `<GATEWAY>` = `192.168.50.1`
  * `<DNS1>` = `8.8.8.8`, `<DNS2>` = `1.1.1.1`
* Hostname da VM: `<HOSTNAME>` (ex.: `vm-glpi`).
* Interface de rede: `<IFACE>` (ex.: `eth0` / `ens33` / `enp0s3` — confirme com `ip a`).
* Usuário do banco: **`glpiuser`**.
* Senhas que você vai definir:

  * `<SENHA_ROOT_DB>` → senha do **root** do MariaDB (no `mysql_secure_installation`).
  * `<SENHA_DB_GLPI>` → senha do **glpiuser** (ex.: `Glpi@2025!`).
* Domínio local (opcional): `<DOMINIO_LOCAL>` (ex.: `glpi.local`).

---

## 🖥️ Parte A — Ativar Hyper‑V no Windows

**GUI**
Painel de Controle → Programas e Recursos → **Ativar ou desativar recursos do Windows** → marque **Hyper‑V** (Plataforma + Ferramentas de Gerenciamento) → OK → reinicie.

**PowerShell (admin)**

```powershell
Enable-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V -All -NoRestart
Restart-Computer
```

---

## 🌐 Parte B — Criar Switch **Externo** (acesso pela LAN)

Hyper‑V Manager → **Virtual Switch Manager…** → **New virtual network switch** → **External** → **Create**

* **External network**: sua placa de rede física
* Marque **Allow management OS to share this adapter** → **OK**

> Sem **External Switch**, outros PCs da rede **não** alcançam a VM.

---

## 🧰 Parte C — Criar a VM no Hyper‑V

Hyper‑V → **New → Virtual Machine…**

* **Name**: `<HOSTNAME>`
* **Generation**: **Generation 2**
* **Memory**: 4096–8192 MB (✓ **Use Dynamic Memory**)
* **Network**: seu **External Switch**
* **Disk**: 40–80 GB (ou mais)
* **Attach ISO**: Debian 12/13

**Settings → Firmware**

* **Enable Secure Boot** ✓
* **Template**: *Microsoft UEFI Certificate Authority*

Inicie a VM e instale o Debian.

---

## 🐧 Parte D — Instalar o Debian (resumo)

* Idioma/teclado/timezone
* Particionamento: **Guided – use entire disk**
* Crie **usuário** (ex.: `admin`) e senha
* Selecione **OpenSSH server**
* Rede: deixe **DHCP** (fixaremos depois)
* Ao terminar, conecte por SSH (VS Code Remote‑SSH / PuTTY)

---

## 🌎 Parte E — *(Opcional)* Rede estática no Debian

> **Pule se quiser manter DHCP por enquanto.**

```bash
sudo -i
hostnamectl set-hostname <HOSTNAME>

cp /etc/network/interfaces /etc/network/interfaces.bak

cat >/etc/network/interfaces <<EOF
auto lo
iface lo inet loopback

auto <IFACE>
iface <IFACE> inet static
    address <IP_VM>
    netmask <NETMASK>
    gateway <GATEWAY>
    dns-nameservers <DNS1> <DNS2>
EOF

systemctl restart networking

# testes
ip a | grep -A2 <IFACE>
ping -c 4 8.8.8.8
ping -c 4 deb.debian.org
```

Se nome não resolver (DNS imediato):

```bash
printf "nameserver <DNS1>\nnameserver <DNS2>\n" >/etc/resolv.conf
```

---

## 📦 Parte F — APT básico + repositório PHP (Sury) *(opcional, recomendado)*

```bash
echo 'Acquire::ForceIPv4 "true";' >/etc/apt/apt.conf.d/99force-ipv4

apt update
apt install -y gnupg ca-certificates lsb-release curl wget apt-transport-https

wget -qO - https://packages.sury.org/php/apt.gpg | gpg --dearmor -o /usr/share/keyrings/sury.gpg
echo "deb [signed-by=/usr/share/keyrings/sury.gpg] https://packages.sury.org/php $(lsb_release -sc) main" > /etc/apt/sources.list.d/php.list

apt update
```

> Se preferir, use o PHP do Debian (≥ 8.1) e pule o Sury.

---

## 🧱 Parte G — Apache, MariaDB, PHP 8.2 (+ módulos GLPI)

```bash
apt install -y apache2 mariadb-server mariadb-client unzip wget curl

apt install -y php8.2 \
 php8.2-cli php8.2-common php8.2-mysql php8.2-gd php8.2-xml \
 php8.2-mbstring php8.2-curl php8.2-intl php8.2-ldap php8.2-zip \
 php8.2-bz2 php8.2-apcu php8.2-bcmath php8.2-imagick

a2enmod php8.2 rewrite headers expires dir
systemctl enable --now apache2
```

**php.ini (opcional)**

```bash
sed -i 's|^;*date.timezone.*|date.timezone = America/Fortaleza|' /etc/php/8.2/apache2/php.ini
sed -i 's/^memory_limit.*/memory_limit = 512M/' /etc/php/8.2/apache2/php.ini
sed -i 's/^upload_max_filesize.*/upload_max_filesize = 64M/' /etc/php/8.2/apache2/php.ini
sed -i 's/^post_max_size.*/post_max_size = 64M/' /etc/php/8.2/apache2/php.ini
sed -i 's/^max_execution_time.*/max_execution_time = 300/' /etc/php/8.2/apache2/php.ini
systemctl restart apache2
```

---

## 🗄️ Parte H — MariaDB (proteger + banco do GLPI)

```bash
mysql_secure_installation
# defina <SENHA_ROOT_DB>, remova anônimos, desabilite root remoto, etc.
```

Crie banco/usuário (troque `<SENHA_DB_GLPI>`):

```bash
mysql -u root -p
```

```sql
CREATE DATABASE glpi CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

DROP USER IF EXISTS 'glpiuser'@'localhost';
DROP USER IF EXISTS 'glpiuser'@'%';

CREATE USER 'glpiuser'@'localhost' IDENTIFIED BY '<SENHA_DB_GLPI>';
CREATE USER 'glpiuser'@'%'         IDENTIFIED BY '<SENHA_DB_GLPI>';

GRANT ALL PRIVILEGES ON glpi.* TO 'glpiuser'@'localhost';
GRANT ALL PRIVILEGES ON glpi.* TO 'glpiuser'@'%';
FLUSH PRIVILEGES;
EXIT;
```

> Se o instalador disser “Access denied”, use **Servidor SQL = `127.0.0.1`** (em vez de `localhost`).

---

## 📥 Parte I — Baixar e posicionar o GLPI 11

**Última versão estável:**

```bash
cd /var/www
wget https://github.com/glpi-project/glpi/releases/latest/download/glpi.tgz
tar -xzf glpi.tgz   # cria /var/www/glpi
```

**Permissões (crítico):**

```bash
chown -R www-data:www-data /var/www/glpi
chmod -R 755 /var/www/glpi
chmod -R 775 /var/www/glpi/files /var/www/glpi/config

apt install -y acl
setfacl -R -m u:www-data:rwx /var/www/glpi/files /var/www/glpi/config
setfacl -R -m d:u:www-data:rwx /var/www/glpi/files /var/www/glpi/config
```

---

## 🌍 Parte J — VirtualHost do Apache (**GLPI 11 usa `/public`**)

> **Abra no editor** (não cole o bloco `<VirtualHost>` direto no shell):

```bash
nano /etc/apache2/sites-available/glpi.conf
```

Cole (ajuste `<DOMINIO_LOCAL>` e `<IP_VM>`):

```apache
<VirtualHost *:80>
    ServerName <DOMINIO_LOCAL>
    ServerAlias <IP_VM>

    DocumentRoot /var/www/glpi/public

    <Directory /var/www/glpi/public>
        AllowOverride All
        Require all granted
        DirectoryIndex index.php
    </Directory>
</VirtualHost>
```

Ative:

```bash
a2dissite 000-default.conf
a2ensite glpi.conf
a2enmod rewrite dir php8.2 headers expires
systemctl reload apache2
```

**Tirar warning FQDN (opcional)**

```bash
echo "ServerName <HOSTNAME>" >/etc/apache2/conf-available/servername.conf
a2enconf servername
systemctl reload apache2
```

Teste:

```bash
curl -I http://127.0.0.1/
# Deve retornar HTTP/1.1 200 OK
```

---

## 🧩 Parte K — Instalação Web do GLPI

No seu PC:

```
http://<IP_VM>/
```

* Idioma: Português (Brasil)
* **Instalar**
* Banco:

  * Servidor: `127.0.0.1`
  * Usuário: `glpiuser`
  * Senha: `<SENHA_DB_GLPI>`
  * Base: `glpi`

Deixe o instalador criar as tabelas. Ao final, remova o instalador:

```bash
rm -rf /var/www/glpi/install
```

**Login inicial**: `glpi / glpi` (troque a senha na 1ª entrada).

---

## ⏱️ Parte L — CRON do GLPI

```bash
echo "* * * * * www-data /usr/bin/php8.2 /var/www/glpi/bin/console glpi:cron --quiet" > /etc/cron.d/glpi
systemctl restart cron
```

---

## 🔐 Parte M — Segurança básica

* Troque a senha do `glpi`.
* Crie um **admin** próprio; desabilite contas padrão se não usar (`tech`, `normal`, `postonly`).
* **HTTPS** (opcional):

```bash
apt install -y certbot python3-certbot-apache
certbot --apache -d <DOMINIO_LOCAL>
```

---

## 🧪 Parte N — Publicar para outros PCs da LAN

Acesse por:

```
http://<IP_VM>/
```

Se não abrir, ajuste firewall:

```bash
ufw allow 80/tcp
ufw allow 443/tcp
ufw reload
```

**Nome amigável nos PCs (opcional)** — arquivo `hosts` (Windows/Linux):

```
<IP_VM>  <DOMINIO_LOCAL>
```

> Sem acesso ao roteador e quer internet? Use túnel temporário:

```bash
ngrok config add-authtoken <SEU_TOKEN>
ngrok http 80
# acesse o link https://xxxxx.ngrok-free.app
```

---

## 🛠️ Troubleshooting rápido

* **404 ao acessar** → VHost errado ou faltando `/public`. Confira `DocumentRoot /var/www/glpi/public` e `a2ensite glpi.conf`.
* **“Unable to write in files/_cache”** → permissões/ACL (reaplique `chown`, `chmod` e `setfacl`).
* **“Access denied for 'glpiuser'”** → crie usuário para `localhost` **e** `%`; use `127.0.0.1` no instalador.
* **APT 0%** → DNS/IPv6. Ajuste `/etc/resolv.conf`, force IPv4, teste `curl -I http://deb.debian.org`.
* **Sury GPG / InRelease** → confirme chave em `/usr/share/keyrings/sury.gpg` e `[signed-by=...]` no `php.list`.
* **bcmath faltando** → `apt install -y php8.2-bcmath && systemctl restart apache2`.

---

### 📄 Licença sugerida

Inclua um arquivo `LICENSE` com **MIT License** ou outra de sua preferência, e um `.gitignore` básico (ex.: `*.log`, `*.swp`, `.DS_Store`).
#   G L P I - 1 1  
 
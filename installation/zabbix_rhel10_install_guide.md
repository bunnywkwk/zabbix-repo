# Zabbix 7.0 LTS Native Installation on RHEL 10
(Designed to coexist with Wazuh Manager)

## Phase 1: Installation and Repository Setup

**1. Add Zabbix RHEL 10 Repo**
Installs the official Zabbix 7.0 repository definition tailored for RHEL 10.
```bash
rpm -Uvh https://repo.zabbix.com/zabbix/7.0/rhel/10/x86_64/zabbix-release-latest.el10.noarch.rpm
```

**2. Install Zabbix Components**
Installs the required backend, frontend, agent, and database packages while bypassing EPEL conflicts and beta GPG key blocks.
```bash
dnf --disablerepo="epel" install zabbix-server-pgsql zabbix-web-pgsql zabbix-nginx-conf zabbix-sql-scripts zabbix-selinux-policy zabbix-agent postgresql-server --nogpgcheck -y
```

---

## Phase 2: Database Setup (PostgreSQL)

**1. Initialize PostgreSQL**
Generates the raw database directory structure and initial system files.
```bash
postgresql-setup --initdb
```

**2. Start and Enable Database Service**
Starts the PostgreSQL daemon and ensures it boots automatically on server restart.
```bash
systemctl enable --now postgresql
```

**3. Create the Database User**
Creates a dedicated PostgreSQL user named 'zabbix' with a secure password for application access.
```bash
sudo -u postgres psql -c "CREATE USER zabbix WITH PASSWORD 'zabbix_password';"
```

**4. Create the Database**
Creates the main database named 'zabbix' and grants full ownership to the user we just created.
```bash
sudo -u postgres psql -c "CREATE DATABASE zabbix OWNER zabbix;"
```

**5. Import Zabbix Schema**
Decompresses and injects the massive default Zabbix table structures and initial data into the blank database.
```bash
zcat /usr/share/zabbix-sql-scripts/postgresql/server.sql.gz | sudo -u zabbix psql zabbix
```

---

## Phase 3: Configure Zabbix Backend

**1. Open Zabbix Server Configuration**
Opens the main configuration file that controls how the background daemon operates.
```bash
vi /etc/zabbix/zabbix_server.conf
```

**2. Update Database Password**
Tells the Zabbix Server daemon the exact password to use when connecting to PostgreSQL (Find `# DBPassword=` and update it).
```text
DBPassword=zabbix_password
```

---

## Phase 4: Configure Web Frontend (Wazuh Fix)

**1. Open Nginx Zabbix Configuration**
Opens the web server configuration file specific to the Zabbix dashboard.
```bash
vi /etc/nginx/conf.d/zabbix.conf
```

**2. Change the Listening Port**
Changes the web port from the default (80/8080) to 8081 to prevent fatal port conflicts with the existing Wazuh Dashboard.
```nginx
listen 8081;
server_name example.com;
```

---

## Phase 5: SELinux Configuration (Port 8081 Fix)

**1. Install SELinux Management Tools**
Installs the semanage utility required to modify advanced SELinux security policies on RHEL.
```bash
dnf install policycoreutils-python-utils -y
```

**2. Whitelist Port 8081**
Explicitly tells SELinux that port 8081 is a safe, valid port for HTTP traffic so Nginx is not blocked from binding to it.
```bash
semanage port -a -t http_port_t -p tcp 8081
```

**3. Enforce SELinux**
Ensures the server's security shield is actively enforcing rules to keep the lab secure.
```bash
setenforce 1
```

---

## Phase 6: Firewall and Start Services

**1. Open Web Port in Firewall**
Permanently opens port 8081 in the RHEL firewall so you can access the dashboard from your browser.
```bash
firewall-cmd --add-port=8081/tcp --permanent
```

**2. Open Agent Port in Firewall**
Permanently opens port 10051 in the firewall so remote Zabbix Agents can send monitoring data back to this server.
```bash
firewall-cmd --add-port=10051/tcp --permanent
```

**3. Reload Firewall**
Applies the permanent port rules immediately to the active firewall environment.
```bash
firewall-cmd --reload
```

**4. Restart All Services**
Restarts the backend, agent, web server, and PHP processor to strictly apply all the config and SELinux changes we made.
```bash
systemctl restart zabbix-server zabbix-agent nginx php-fpm
```

**5. Enable All Services on Boot**
Configures all four crucial services to automatically start themselves if the Proxmox VM is rebooted.
```bash
systemctl enable zabbix-server zabbix-agent nginx php-fpm
```

---

## Phase 7: Access the Web UI

**1. Open Web Browser**
Navigate to the IP address of your RHEL 10 VM on the custom port we configured.
```text
http://<YOUR_RHEL10_VM_IP>:8081
```

**2. Complete Setup Wizard**
Follow the on-screen prompts (the Database password is `zabbix_password`). The default login credentials are:
* Username: Admin
* Password: zabbix

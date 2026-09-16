# Zabbix 8.0 Pre-Release Native Installation on RHEL 10

## Phase 1: Repository Preparation
Because EPEL contains older versions of Zabbix, we must permanently exclude them so they do not conflict with the 8.0 installation.

**1. Open the EPEL repository file**
Opens the configuration file for the EPEL repository.
```bash
vi /etc/yum.repos.d/epel.repo
```

**2. Add the exclusion rule**
Add this line directly under the `[epel]` heading to force the system to ignore EPEL's Zabbix packages.
```ini
excludepkgs=zabbix*
```

---

## Phase 2: Installation

**1. Add Zabbix 8.0 RHEL 10 Repo**
Installs the official Zabbix 8.0 repository definition tailored for RHEL 10.
```bash
rpm -Uvh https://repo.zabbix.com/zabbix/8.0/release/rhel/10/noarch/zabbix-release-latest-8.0.el10.noarch.rpm
```

**2. Install Zabbix Components**
Installs the backend, Apache frontend, agent, and PostgreSQL database, using `--nogpgcheck` per-command to bypass RHEL 10 beta key mismatches without lowering global security.
```bash
dnf install zabbix-server-pgsql zabbix-web-pgsql zabbix-apache-conf zabbix-sql-scripts zabbix-selinux-policy zabbix-agent postgresql-server --nogpgcheck -y
```

---

## Phase 3: Database Setup (PostgreSQL)

**1. Initialize PostgreSQL**
Generates the raw database directory structure and initial system files.
```bash
postgresql-setup --initdb
```

**2. Apply Local Authentication Fix**
Automatically changes the strict local authentication from 'ident' to 'md5' so Zabbix can log in using a password.
```bash
sed -i 's/ident/md5/g' /var/lib/pgsql/data/pg_hba.conf
```

**3. Start and Enable Database Service**
Starts the PostgreSQL daemon and ensures it boots automatically on server restart.
```bash
systemctl enable --now postgresql
```

**4. Create the Database User**
Interactively creates the Zabbix database user (it will prompt you to type a password).
```bash
sudo -u postgres createuser --pwprompt zabbix
```

**5. Create the Database**
Creates the main database named 'zabbix' and grants full ownership to the user we just created.
```bash
sudo -u postgres createdb -O zabbix zabbix
```

**6. Import Zabbix 8.0 Schema**
Decompresses and injects the massive default Zabbix 8.0 table structures into the blank database (it will prompt for your password).
```bash
zcat /usr/share/zabbix/sql-scripts/postgresql/server.sql.gz | sudo -u zabbix psql zabbix
```

---

## Phase 4: Configure Zabbix Backend

**1. Open Zabbix Server Configuration**
Opens the main configuration file that controls how the background daemon operates.
```bash
vi /etc/zabbix/zabbix_server.conf
```

**2. Update Database Password**
Tells the Zabbix Server daemon the exact password to use when connecting to PostgreSQL (Find `# DBPassword=` and update it).
```text
DBPassword=your_password_here
```

---

## Phase 5: Environment Tailoring (Optional Port Workaround)

**Note:** If your server is dedicated to Zabbix and ports 80 and 443 are free, you can safely skip this entire phase and **[click here to jump directly to Phase 6](#phase-6-firewall-and-start-services)**. We are only changing these settings because an existing application (Wazuh Manager) already occupies the default web ports.

**1. Open Apache Configuration**
Opens the main Apache web server configuration file.
```bash
vi /etc/httpd/conf/httpd.conf
```

**2. Change the Listening Port**
Find the line that says `Listen 80` and change it to `Listen 8081` to avoid fighting with Wazuh.
```apache
Listen 8081
```

**3. Whitelist Port 8081 in SELinux**
Explicitly tells SELinux that port 8081 is a safe, valid port for HTTP traffic so Apache is not blocked from binding to it.
```bash
semanage port -a -t http_port_t -p tcp 8081
```

---

## Phase 6: Firewall and Start Services

**1. Open Web Port in Firewall**
Permanently opens the web port in the RHEL firewall. 
*(Run the first command if you skipped Phase 5, OR the second command if you used the custom 8081 port)*
```bash
# If you skipped Phase 5 (Standard Port 80):
firewall-cmd --add-service=http --permanent

# If you completed Phase 5 (Custom Port 8081):
firewall-cmd --add-port=8081/tcp --permanent
```

**2. Open Agent Port in Firewall**
Permanently opens port 10051 in the firewall so remote Zabbix Agents can send monitoring data.
```bash
firewall-cmd --add-port=10051/tcp --permanent
```

**3. Reload Firewall**
Applies the permanent port rules immediately to the active firewall environment.
```bash
firewall-cmd --reload
```

**4. Start and Enable All Services**
Restarts the backend, agent, Apache web server, and PHP processor and locks them in to start automatically on server reboot.
```bash
systemctl enable --now zabbix-server zabbix-agent httpd php-fpm
```

---

## Phase 7: Troubleshooting Pre-Release Dependencies (c-ares Error)

**Note:** Because Zabbix 8.0 is a pre-release and RHEL 10 is very new, the Zabbix Server may fail to start with a specific systemd error referencing the DNS library:
`undefined symbol: ares_queue_wait_empty`

**1. Update the c-ares library**
Forcefully updates the core DNS lookup library to the latest version to satisfy the Zabbix 8.0 requirement.
```bash
dnf update c-ares --nogpgcheck -y
```

**2. Restart the Zabbix Server**
Restarts the server to successfully apply the newly updated system library.
```bash
systemctl restart zabbix-server
```

---

## Phase 8: Access the Web UI and Setup Wizard

**1. Open Web Browser**
Navigate to the IP address of your RHEL 10 VM, appending `/zabbix` which is required by the default Apache configuration.
* *(If you skipped Phase 5)*: `http://<YOUR_RHEL10_VM_IP>/zabbix`
* *(If you used Port 8081)*: `http://<YOUR_RHEL10_VM_IP>:8081/zabbix`

**2. Configure DB Connection**
On the database configuration screen, enter the password you created in Phase 3. **Crucially**, ensure that the "Database TLS encryption" box is unchecked, since we did not configure local SSL certificates for this lab setup.
![Configure DB Connection](images/configure_db.png)

**3. Pre-Installation Summary**
Review the summary screen. If you reach this page, it is a success check—it guarantees that the Zabbix Web UI successfully authenticated with your PostgreSQL database.
![Pre-Installation Summary](images/pre_install_summary.png)

**4. Dashboard Login**
When you finally reach the main Zabbix login screen, do **not** use your database credentials. You must use the built-in default Super Admin account:
* Username: **Admin** *(must be a capital 'A')*
* Password: **zabbix**
![Login Screen Error](images/login_screen.png)

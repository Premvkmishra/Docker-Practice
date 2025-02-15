# Docker

You will find all the Docker services set up in this folder.

## Services

### VSCode

:link: [docker image](https://hub.docker.com/r/codercom/code-server)
:link: [github repository](https://github.com/coder/code-server)

Access VSCode through [localhost:8080](http://localhost:8080).

:lock:
The password to access VSCode is `yourpassword` it can be set it in the [docker-compose.yaml file](docker-compose.yaml).

### QuestDB

:link: [docker image](https://hub.docker.com/r/questdb/questdb)
:link: [github repository](https://github.com/questdb/questdb)
:link: [questdb Docker documentation](https://questdb.io/docs/get-started/docker/)

Access QuestDB GUI through [localhost:9000](http://localhost:9000).
Access the database using [localhost:8812](http://localhost:8812).

:lock:
The user/password are the default one: `admin:quest` ([see the documentation](https://questdb.io/docs/reference/configuration/#postgres-wire-protocol)) and the database name is `qdb`.

### Grafana

:link: [docker image](https://hub.docker.com/r/grafana/grafana)
:link: [github repository](https://github.com/grafana/grafana)

Access Grafana through [localhost:3000](http://localhost:3000).

:lock:
The user/password are the default one: `admin:admin`.
You can add set the password adding the environment variable `GF_SECURITY_ADMIN_PASSWORD`.

:wrench: To configure Grafana, follow [this documentation](./grafana/README.md).

## Usage

**Note**: A **fully qualified domain name**  (FQDN) is mandatory for running any notebooks from VSCode over **HTTPS**.

To run the VSCode notebooks over HTTPS and ensure secure communication across your services, it's essential to set up port forwarding for critical ports. You'll need to forward the following ports from your host machine or cloud provider (e.g., AWS, GCP, or Azure) to your server's local environment:

- **Port 443 (HTTPS)**: This is necessary for serving the VSCode instance and other web services securely over HTTPS using SSL certificates. Ensure your server has a fully qualified domain name (FQDN) configured, as it is mandatory for enabling HTTPS.
  
- **Port 80 (HTTP)**: Port 80 is used to redirect HTTP traffic to HTTPS and for the initial Let's Encrypt SSL certificate challenge during the Certbot process.

- **Port 22 (SSH)**: For secure access to your server via SSH. This is useful for remote management, debugging, and manual configuration if needed.

- **Port 8812 (QuestDB REST API)**: QuestDB uses port 8812 for its REST API. Make sure this port is forwarded if you plan to access or query QuestDB data remotely.

- **Port 9009 (QuestDB Influx Line Protocol)**: This port is used for the InfluxDB line protocol in QuestDB, allowing remote connections to ingest time-series data. Forwarding this port is essential if you're collecting time-series data from remote sources.

### Router Configuration (if applicable):

If you're running your server from a local network (e.g., at home or in an office), you will need to configure port forwarding on your router to allow external traffic to access your server. Here's a general guide on how to configure your router for port forwarding:

1. **Login to your router**: Access your router’s configuration page via its IP address (usually something like `192.168.0.1` or `192.168.1.1`).
   
2. **Navigate to Port Forwarding section**: Look for a section called "Port Forwarding" or "Virtual Server." The location of this section varies depending on the router model.

3. **Add new port forwarding rules** for each required port:
   - **External Port**: Enter the external port number (e.g., 443, 80, 22, 8812, 9009).
   - **Internal IP Address**: Enter the local IP address of the machine running the services (e.g., your server's local IP address).
   - **Internal Port**: Enter the same port number to forward the traffic internally.
   - **Protocol**: Choose either TCP or UDP, or select "Both" depending on the service requirements.
   
4. **Save the settings** and restart your router if necessary.

You can set up port forwarding either through your cloud provider's security group settings (for example, AWS EC2 security groups) or by configuring firewall rules in your server's networking settings (e.g., using `iptables` or `ufw` on a Linux server). Be sure to secure these ports and restrict access to trusted IPs where necessary to avoid unauthorized access.



### 1 Generate Certificates with Certbot

We will use the Certbot Docker image to generate certificates. This service will bind on ports 80 and 443, which are the standard HTTP and HTTPS ports, respectively. It will also bind mount some directories for persistent storage of certificates and challenge responses. 

If you want to obtain separate certificates for each subdomain, you will need to run the **certbot certonly** command for each one. You can specify the subdomain for which you want to obtain a certificate with the -d option, like this:

```
docker compose -f init.yaml run certbot certonly -d vscode.yourdomain.tld
```


```
docker compose -f init.yaml run certbot certonly -d questdb.yourdomain.tld
```


```
docker compose -f init.yaml run certbot certonly -d grafana.yourdomain.tld
```


```
docker compose -f init.yaml run certbot certonly -d aiagentui.yourdomain.tld
```



**Important:** Replace **'yourdomain.tld'** with your actual domain in the commands above.

Please note that the **certonly** command will obtain the certificate but not install it. You will have to configure your Nginx service to use the certificate. Additionally, make sure that your domain points to the server on which you're running this setup, as Let's Encrypt validates domain ownership by making an HTTP request.

Also, remember to periodically renew your certificates, as Let's Encrypt's certificates expire every 90 days. You can automate this task by setting up a cron job or using a similar task scheduling service.


```
# /etc/cron.d/certbot: crontab entries for the certbot package (5 am, every monday)
SHELL=/bin/sh
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin

 0 5 * * 1 docker compose run certbot renew

```

### 2 Setup Environment Variables

Since the '**.env**' file already exists in the docker folder, please update it to include or modify the necessary variables as needed for your setup:

```
# VSCode
PASSWORD=yourpassword

# Grafana
GRAFANA_QUESTDB_PASSWORD=quest
GF_AUTH_ANONYMOUS_ENABLED=true
GF_AUTH_ANONYMOUS_ORG_ROLE=Viewer
GF_AUTH_ANONYMOUS_ORG_NAME=Main Org.
GF_AUTH_ANONYMOUS_ALLOW_EMBEDDING=true
GF_SECURITY_ALLOW_EMBEDDING=true
GF_SECURITY_ADMIN_USER=admin
GF_SECURITY_ADMIN_PASSWORD=admin

# QuestDB
QDB_PG_USER=admin
QDB_PG_PASSWORD=quest
QDB_PG_NAME=qdb
QDB_PG_HOST=docker_host_ip_address
QDB_PG_PORT=8812

# VSCode Grafana QuestDB AI Agent UI
DOMAIN=domain.tld

# Chatbot
OPENAI_API_KEY=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx


# secret key

SECRET_KEY=your_very_secure_secret_key

```


Remember to replace the placeholders with your actual domain, passwords, and usernames. 

The environment variables will be replaced directly within the Nginx configuration file when the Docker services are started.


### 3 Generate dhparam.pem file

The **dhparam.pem** file is used for Diffie-Hellman key exchange, which is part of establishing a secure TLS connection. You can generate it with OpenSSL. Here's how to generate a 2048-bit key:

```
# Note: For deployment testing, we are using an existing dhparam.pem file. This step is skipped because the file is already configured.
# Normally, you would generate a new dhparam.pem file for Diffie-Hellman key exchange with the following command:
# openssl dhparam -out ./nginx/certs/dhparam.pem 2048
# However, for testing purposes, this is not required.


openssl dhparam -out ./nginx/certs/dhparam.pem 2048

```

Generating a dhparam file can take a long time. For a more secure (but slower) 4096-bit key, simply replace 2048 with 4096 in the above command.

### 4 Generate .htpasswd file for QuestDB 

The user/password are the default one: admin:admin

The **.htpasswd** file is used for basic HTTP authentication. You can change it using the **htpasswd** utility, which is part of the Apache HTTP Server package. Here's how to create an **.htpasswd** file with a user named **yourusername**:

```
# Note: For deployment testing, we are using an existing .htpasswd file. This step is skipped as the authentication setup is already in place.
# Typically, you would create a new .htpasswd file for basic HTTP authentication using:
# htpasswd -c ./nginx/.htpasswd yourusername
# But for the purposes of testing, this step can be omitted.

htpasswd -c ./nginx/.htpasswd yourusername

```

This command will prompt you for the password for **yourusername**. The **-c** flag tells **htpasswd** to create a new file. **Caution**: Using the **-c** flag will overwrite any existing **.htpasswd** file. 

If **htpasswd** is not installed on your system, you can install it with **apt** on Ubuntu:

```
sudo apt-get install apache2-utils
```


### 5 **Metric Monitoring with Prometheus**: 

To monitor system metrics like CPU, memory, and disk usage, you can now leverage the fully provisioned Prometheus database and the Node Exporter Full dashboard (ID 1860) within your Grafana instance. This setup provides a comprehensive view of system metrics collected via the Prometheus Node Exporter.

To ensure proper configuration, you need to modify the `prometheus-ip-address` URL in the `prometheus.yml` file as shown below:

```
# /grafana/provisioning/datasources/prometheus.yml

apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus-ip-address:9090
    isDefault: true
    editable: false
    uid: rYdddlPWj  # Ensure this UID is unique, used in the dashboard JSON
```

Make sure your Prometheus server is configured to scrape metrics from the Node Exporter.

```
#  /etc/prometheus/prometheus.yml

global:
  scrape_interval: 10s

scrape_configs:
  - job_name: 'prometheus'
    scrape_interval: 5s
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'node_exporter_metrics'
    scrape_interval: 5s
    static_configs:
      - targets: ['docker_host_ip_address:9100']

```

### 6 Launching the Docker Stack and Starting Services

After completing these steps, you can bring up the Docker stack using the following command:

```
docker compose up --build -d
```
This will start all services as defined in your **docker-compose.yaml** file.



## Understanding and Managing Docker Permissions in a Docker Stack

### 1 Docker Volumes Permissions 

The Docker Compose configuration mounts the local directory **../notebooks** to **/home/coder/project** inside the **vscode** service using Docker volumes. This is specified in the **volumes** section of the **vscode** service:

``````
volumes:
  # volume used to access the `notebooks` folder
  - ../notebooks:/home/coder/project
``````

However, this configuration doesn't specify any permission settings for the **../notebooks** directory. Therefore, the permissions inside the container for the mounted directory will be the same as the permissions on the host for **../notebooks.**

In other words, the permissions are determined by the file system of the host machine where the Docker Compose file is run, not by the Docker Compose configuration itself. You can check these permissions using the **ls -l** command in your host system's terminal.

If you want to change the permissions, you would have to do this at the operating system level, outside of Docker. For example, in a Unix-like system, you might use the **chmod** command to change the permissions of the **../notebooks** directory.

Please note that if the **coder** user in your Docker container needs specific permissions (e.g., to write to the **/home/coder/project** directory), you will need to ensure that the host-level permissions for **../notebooks** allow for this. Otherwise, you may run into permission errors.

If you wish to enforce specific permissions within the container regardless of host permissions, you would likely need to handle this in the Dockerfile used to build your image, for example by adding **RUN chmod** commands to change the permissions after the volume is mounted. But keep in mind that it might not always work depending on the volume's nature, and the Dockerfile does not have direct access to the volumes at build time.

Another option would be to use an entrypoint script in your Dockerfile that modifies the permissions of the directory when the container starts, but again, be cautious about potential security implications of changing permissions in this way.


### 2 Setting Default File and Directory Permissions for Git Clone Operations

The permissions for files and directories created by **git clone** are determined by your system's settings, not by Git itself. The most important of these settings is the **umask**, a value that determines the default permissions for newly created files and directories.

When you run **git clone**, Git creates a new directory for the repository, and the permissions for this directory are determined by subtracting the umask from **777** (for directories) or **666** (for files). The resulting permissions apply to the cloned directory and all files within.

If you want to change the default permissions for **git clone**, you can temporarily change the umask before running the command. For example, if you want the cloned directory to have permissions of **755** (rwxr-xr-x), you could set the umask to **022** like so:

```
umask 022
git clone https://github.com/quantiota/AI-Agent-Lab.git
```

This will only affect the current shell session. If you want to change the umask permanently, you can add the **umask 022** command to your shell's initialization file (like **~/.bashrc** or **~/.bash_profile** for the Bash shell).

Keep in mind that this will affect all file and directory creation operations in your system, not just **git clone**, so be careful when changing the umask.

Remember that you can always change the permissions of a directory or file after it's created using the **chmod** command. For example:

```
chmod 755 /path/to/directory
```

This command will change the permissions of the specified directory to **755**, regardless of the umask or the directory's initial permissions.
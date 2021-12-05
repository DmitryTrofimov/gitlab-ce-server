# Setup Gitlab in Docker Compose

### Requirements

* Ubuntu Server 20.04
* VPS with min 2 CPU and 8GB of RAM
* Admin privileges

### Step 1 - Install Docker and Docker Compose

Update packages index and install dependencies to allow apt to use a repository over HTTPS:
```
$ apt-get update
$ apt-get install -y apt-transport-https ca-certificates curl gnupg lsb-release
```

Add Docker’s official GPG key:
```
$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
```

Use the following command to set up the stable repository:
```
$ echo \
    "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
    $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

Update the apt package index, and install the latest version of `Docker Engine ` and `containerd`:
```
$ apt-get update
$ apt-get install -y docker-ce docker-ce-cli containerd.io
```

Download the current stable release of Docker Compose:
```
$ curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
```

Apply executable permissions to the binary and create a symbolic link to `/usr/bin`:
```
$ chmod +x /usr/local/bin/docker-compose
$ ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose
```

### Step 2 - Change System's SSH Port

GitLab will use port 22 for pushing repositories via SSH. Your server's SSH is also running on it, which will generate conflict. There are two ways to solve it. Choose one you prefer.

1. Change the SSH port you use to connect to your system (recommended).

    Edit the `/etc/ssh/sshd_config` file. Find the following line and change 22 to 2224 and remove the `#` in front of it. You can choose any port you want.

    ```
    # Port 22
    ```

    Restart the SSH service.
    ```
    $ sudo systemctl restart sshd
    ```
    
    Close your current SSH session and create a new one with the port `2224` and connect to your server again.
    ```
    $ ssh user@server -p 2224
    ```

    Setup and Configuring Firewall.
    ```
    $ sudo apt install ufw apt-transport-https
    $ sudo ufw allow OpenSSH
    $ sudo ufw allow 2224
    $ sudo ufw enable
    $ sudo ufw allow http
    $ sudo ufw allow https
    $ sudo ufw status
    ```

    Output shouls looks like:
    ```
    Status: active

    To                         Action      From
    --                         ------      ----
    OpenSSH                    ALLOW       Anywhere                  
    2224                       ALLOW       Anywhere                  
    80/tcp                     ALLOW       Anywhere                  
    443/tcp                    ALLOW       Anywhere                  
    OpenSSH (v6)               ALLOW       Anywhere (v6)             
    6622 (v6)                  ALLOW       Anywhere (v6)      
    80/tcp (v6)                ALLOW       Anywhere (v6)             
    443/tcp (v6)               ALLOW       Anywhere (v6)    
    ```

2. Change the port GitLab will use for SSH. 
  
    In this case you need to add the line `gitlab_rails['gitlab_shell_ssh_port'] = ${SSH_PORT}` under `GITLAB_OMNIBUS_CONFIG` in your `docker-compose.yml` file, line in example below:

    ```
    docker-compose.yml
    ------------------

    version: '3.8'
    services:
    ...
    environment:
        GITLAB_OMNIBUS_CONFIG: |
            external_url 'https://$GITLAB_HOST'
            gitlab_rails['gitlab_shell_ssh_port'] = 2224
    ...
    ports:
        - '2224:22'
    ...
    ```

    In this example `2224` port will be used to connect to Gitlab server.
    ```
    $ git clone git@github.com:user/repo.git -p 2224
    ```

### Step 3 - Setup Environment

Clone this project and create a copy of `.env` file from template:
```
$ git clone
$ cp .env.template .env
```
Set `$GITLAB_HOME` and `GITLAB_RUNNER_HOME` environment variables from `.env` file. Create data directories:

```
$ mkdir -p ${GITLAB_HOME}/{config/ssl,logs,data}
$ mkdir -p ${GITLAB_RUNNER_HOME}
```

### Generate SSL and DHAPARAM Certificates
Install the certbot tool:
```
$ apt install -y certbot
```

Generate the SSL certificate for GitLab (Set `$EMAIL` and `$GITLAB_HOST` from `.env` file):
```
$ certbot certonly --rsa-key-size 2048 --standalone --agree-tos --no-eff-email --email ${EMAIL} -d ${GITLAB_HOST}
```

Copy generated certificates to the related directories for Gitlab:
```
$ cp /etc/letsencrypt/live/${GITLAB_HOST}/* ${GITLAB_HOME}/config/ssl/
```

Generate the DHPARAM certificate:
```
$ openssl dhparam -out ${GITLAB_HOME}/config/ssl/dhparams.pem 2048
```

### Build Container

Build a GitLab container 
```
$ docker-compose up -d
```

Check the running containers
```
$ docker-compose ps
```

### Post Installation

Open https://$GITLAB_HOST in your browser. Set up initial root user password and log in to the gitlab.
Perform next operations:

* Change username from `root` to $YOUR_USERNAME

* Restrict Public Sign-ups ❗

* Add SSH key:
```
$ ssh-keygen -t ed25519 -C $EMAIL
```

* Test connection via ssh:
```
$ ssh -T git@$GITLAB_HOST
```

## Additional

### Add Docker-in-Docker Runner
Access to gitlab runner shell: 
`$ docker exec -it gitlab_runner /bin/bash`

To register a new gitlab runner you can follow official gitlab instructions in scope of [interactive mode](https://docs.gitlab.com/runner/register/#linux), or reach the same goal in [non-interactive mode](https://docs.gitlab.com/runner/register/#one-line-registration-command).

Add `docker-latest-dind` as runner:
```
$ gitlab-runner register -n \
    --url $GITLAB_HOST \
    --registration-token $REGISTRATION_TOKEN \
    --description "docker-latest-dind" \
    --tag-list "docker-latest-dind" \
    --executor docker \
    --docker-image "docker:latest" \
    --docker-privileged \
    --docker-volumes "/certs/client"
```

### Quick Notes

* Check logs:
`$ docker-compose logs -f`

* Access to container:
`$ docker exec -it gitlab /bin/bash`

* Check gitlab status:
`$ docker exec -it gitlab gitlab-ctl status`

* Gitlab Registry will be awailable after login:
`$ docker login $GITLAB_HOST:5050`

* Reconfigure gitlab after manually update `gitlab.rb` file:
`$ docker exec -it gitlab gitlab-ctl reconfigure`

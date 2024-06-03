
## Purpose
This document is a tutorial for preparing Synology DSM for the installation of the [Matrix Docker Ansible Deploy](https://github.com/spantaleev/matrix-docker-ansible-deploy) project. It is intended for users who are already familiar with DSM, SSH, and Matrix Docker Ansible Deploy project. Please ensure you understand each step before executing commands or making configuration changes.

### Assumption/Prerequisite
1. You are using DSM to handle reverse proxy requests.
2. You are using the Ansible project.
3. Your DSM OS version is 7 or higher.
4. you are using Volume1 as the default location for docker

### Synology GUI Preparation
1. Enable SSH service
    - From `Control Panel` > `Terminal & SNMP` > `Enable SSH service`
2. Enable UserHome Directory
    - From `Control Panel` > `User & User Group` > `Enable user home service`
3. Install `Container Manager` from `Package Center`
4. Install `Web Station` (may not be needed).
5. Create Users and User group for `matrix`. 
6. Create Reverse Proxy for your docker container. e.g. matrix, element, admin page or other services you have enabled.
    From `Control Panel` > `Login Portal` > `Advanced` > `Reverse Proxy`
    Below is one of the examples
    - Reverse Proxy Name: (free text)
    - Source
        - Protocol: `HTTPS`
        - hostname: `element.example.com`
        - Port: `443`
    - Destination
        - Protocol: `HTTP`
        - hostname: `localhost`
        - port: `81` 
7. Create two Task Scheduler. 
This is the required step, otherwise you need to execute this command everytime after system reboot. If failed to do so, matrix container will not work.
    From `Control Panel` > `Task Scheduler` > `Create` > `Trigered Task`
    - Boot up Schedule
        - General
            - Task: `Bootup_Matrix(Free Text)`
            - User: `root`
            - Event: `boot-up`
        - Task Settings
            - Run Command: 
            ```SH
                mount --make-shared /volume1 
                ln -s /usr/local/lib/systemd/system/pkg-ContainerManager-dockerd.service /etc/systemd/system/docker.service
                systemctl daemon-reload
            ```
    - Shutdown Schedule
        - General
            - Task: `Shutdown_Matrix(Free Text)`
            - User: `root`
            - Event: `Shutdown`
        - Task Settings
            - Run Command: 
            ```SH
                rm /etc/systemd/system/docker.service
            ```
    
### SSH to your DSM
0. (Optional but strongly recommend) Please Enable Key Authentication login for SSH. 
    Please Google it by yourselves if you don't know how to do it. There are plenty of resource online.
1. Create a project folder for installing python package
```Shell
mkdir ~/path/to/your/project/folder
cd ~/path/to/your/project/folder
```
2. Install python `requests` package
```Shell
# create virtual environment
python -m venv ./myenv
# activate created environment
source ./myenv/bin/activate
# (optional) you don't have to upgrade your pip
python -m pip install --upgrade pip

# normally, you don't have to specify the version while installing the requests package. As of May of 2024, you will encounter an exception while using the latest version of this package.
# ref: https://stackoverflow.com/a/78508652
pip install requests==2.31.0

```

3. Create docker service alias. If you don't remove this symbolic link, DSM will prompt you to repair Container Manager every time DSM restarts. That's why we need to create `Shutdown` scheduler task to remove link and rebuild it during `Bootup`.
```Shell
sudo ln -s /usr/local/lib/systemd/system/pkg-ContainerManager-dockerd.service /usr/local/lib/systemd/system/docker.service
sudo systemctl daemon-reload
# checking service status, you should be able to see it running.
sudo systemctl status docker

# please execute below code to remove the link if you want.
# sudo rm /usr/local/lib/systemd/system/docker.service
```

4. Mount Volume for `matrix-synapse.service`
```Shell
sudo mount --make-shared /volume1
```

5. check the user id and group id for `matrix` user
```Shell
id matrix
# The output will be: uid=1027(matrix) gid=100(users) groups=100(users),65536(matrix)
# 1027 is the uid and 65536 is the gid
```


### Prepare vars.yml and hosts file

1. Add custom ansible python interpreter to your hosts file
```shell
matrix.<domain> ansible_host=<your-dsm-ip> ansible_ssh_user=<dsm-ssh-user> ~/path-to-your-python-virtual-env/myenv/bin/python ansible_sudo_pass='your-password'
```

2. Modified your `vars.yml` file accordingly. The reverse proxy configuration provided has been tested and works. Only change it if you know what you are doing.

```YAML
# Synology Tailored Parameters:

# please change based on your actual username and path.
matrix_user_username: "matrix"
matrix_user_groupname: "matrix"
matrix_user_uid: 1027
matrix_user_gid: 65536
matrix_base_data_path: "/volume2/docker/matrix"

# don't change this. And it only works on DSM7
devture_systemd_docker_base_host_command_docker: "/usr/local/bin/docker"
devture_timesync_ntpd_service: "chronyd"
matrix_playbook_docker_installation_enabled: false


# Ensure that public urls use https
matrix_playbook_ssl_enabled: true

# Disable the web-secure (port 443) endpoint, which also disables SSL certificate retrieval
devture_traefik_config_entrypoint_web_secure_enabled: false

# If your reverse-proxy runs on another machine, consider using `0.0.0.0:81`, just `81` or `SOME_IP_ADDRESS_OF_THIS_MACHINE:81`
devture_traefik_container_web_host_bind_port: '127.0.0.1:81'
matrix_playbook_public_matrix_federation_api_traefik_entrypoint_host_bind_port: '127.0.0.1:8449'

# We bind to `127.0.0.1` by default (see above), so trusting `X-Forwarded-*` headers from
# a reverse-proxy running on the local machine is safe enough.
devture_traefik_config_entrypoint_web_forwardedHeaders_insecure: true

# Or, if you're publishing the port (`devture_traefik_container_web_host_bind_port` above) to a public network interfaces:
# - remove the `devture_traefik_config_entrypoint_web_forwardedHeaders_insecure` variable definition above
# - uncomment and adjust the line below
# devture_traefik_config_entrypoint_web_forwardedHeaders_trustedIPs: ['IP-ADDRESS-OF-YOUR-REVERSE-PROXY']

# Likewise (to `devture_traefik_container_web_host_bind_port` above),
# if your reverse-proxy runs on another machine, consider changing the `host_bind_port` setting below.
matrix_playbook_public_matrix_federation_api_traefik_entrypoint_config_custom:
  forwardedHeaders:
    insecure: true
    # If your reverse-proxy runs on another machine, remove the config above and use this config instead:
    # config:
    #   forwardedHeaders:
    #     insecure: true
    #     # trustedIPs: ['IP-ADDRESS-OF-YOUR-REVERSE-PROXY']

```

3. command for my own reference
```SH
ansible-playbook -i inventory/hosts setup.yml --tags=setup-all
ansible-playbook -i inventory/hosts setup.yml --tags=install-all,start
ansible-playbook -i inventory/hosts setup.yml --tags=stop

#.service file location: /etc/systemd/system
```

### Create Matrix user
Don't know why I can't create the user from the Ansible playbook. I didn't pay too much attention on troubleshooting the issue. Below is the command I used to create any Matrix user.
```Shell
# Please execute it from DSM SSH session
sudo docker exec -it matrix-synapse register_new_matrix_user http://localhost:8008 -c /data/homeserver.yaml -u your_user_name -p your_password
```

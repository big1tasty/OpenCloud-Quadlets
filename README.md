# OpenCloud-Quadlets
This is an example of an OpenCloud setup using Podman Quadlets.
The result of this setup is a fully functional OpenCloud server with an advanced search service (Tika) and OnlyOffice integration.
Authentication is handled by Keycloak.
This example assumes that a Keycloak instance is already up and running.

# Prepare Filesystem
```
sudo rm -Rf /container/opencloud && \
mkdir /container/opencloud && \
mkdir /container/opencloud/config && \
touch /container/opencloud/config/csp.yaml && \
mkdir /container/opencloud/data && \
podman unshare chown -R $USER:$USER /container/opencloud
``` 

**csp.yaml**: edit `sudo nano /container/opencloud/config/csp.yaml`
```yaml
directives:
  child-src:
    - 'self'
  connect-src:
    - 'self'
    - 'https://keycloak.domain.tld'
    - 'https://onlyoffice.domain.tld'
    - 'https://raw.githubusercontent.com/opencloud-eu/awesome-apps/'
    - 'https://update.opencloud.eu/'
  default-src:
    - 'none'
  font-src:
    - 'self'
  frame-ancestors:
    - 'self'
  frame-src:
    - 'self'
    - 'https://keycloak.domain.tld'
    - 'https://onlyoffice.domain.tld'
    - 'https://embed.diagrams.net/'
    - 'https://docs.opencloud.eu'
  img-src:
    - 'self'
    - 'data:'
    - 'blob:'
    - 'https://raw.githubusercontent.com/opencloud-eu/awesome-apps/'
    - 'https://tile.openstreetmap.org/'
    # In contrary to bash and docker the default is given after the | character
    - 'https://onlyoffice.domain.tld/'
  manifest-src:
    - 'self'
  media-src:
    - 'self'
  object-src:
    - 'self'
    - 'blob:'
  script-src:
    - 'self'
    - 'unsafe-inline'
    - 'https://keycloak.domain.tld'
  style-src:
    - self'
    - 'unsafe-inline'
``` 

## keycloak: 
Add Roles according to official OpenCloud Documentation and add realm roles mapper with token claim `roles` to the clients: OpenCloudWeb, OpenCloudDesktop, OpenCloudIOS, OpenCloudAndroid
type = string
enable: 
- Multivalued
- add id token
- add to access token
- add token inspection


## create secret for the smtp service
```bash
printf "proton-secret" | podman secret create opencloud_smtp_pw -
```

# Init Opencloud
```
podman run --rm -it \
-v /container/opencloud/config:/etc/opencloud:z \
-v /container/opencloud/data:/var/lib/opencloud:z \
-e IDM_ADMIN_PASSWORD={somerandompassword} \
docker.io/opencloudeu/opencloud:4 init
```

# Create Network 
```bash
nano /etc/containers/systemd/users/1000/opencloud.network
```

```bash
[Unit]
Description=Opencloud Network Service

[Network]
NetworkName=opencloud_net
Driver=bridge

[Install]
WantedBy=multi-user.target default.target
```
# Create POD
```bash
nano /etc/containers/systemd/users/1000/opencloud.pod
```

```bash
[Unit]
Description=Opencloud Pod Service
After=opencloud-network.service
Requires=opencloud-network.service

[Pod]
PodName=opencloud
Network=opencloud_net

[Install]
WantedBy=multi-user.target default.target
```

## Tika (search)
```bash
nano /etc/containers/systemd/users/1000/opencloud_tika.container
```

```bash
[Unit]
Description=Opencloud Tika Service
After=opencloud-pod.service
Requires=opencloud-pod.service

[Container]
ContainerName=opencloud_tika
Image=docker.io/apache/tika:latest
Network=opencloud_net
Pod=opencloud.pod
AutoUpdate=registry

[Service]
Restart=always

[Install]
WantedBy=multi-user.target default.target
```

## opencloud_server
```bash
nano /etc/containers/systemd/users/1000/opencloud_server.container
```

```bash
[Unit]
Description=Opencloud Server Service
After=opencloud_tika.service
Requires=opencloud_tika.service

[Container]
ContainerName=opencloud_server
Image=docker.io/opencloudeu/opencloud:4
Network=opencloud_net
Pod=opencloud.pod
AutoUpdate=registry
AddHost=opencloud.domain.tld:{your_host_IP}
AddHost=onlyoffice.domain.tld:{your_host_IP}
AddHost=keycloak.domain.tld:{your_host_IP}
Environment=OC_URL=https://opencloud.domain.tld
Environment=OC_LOG_LEVEL=warn
Environment=OC_LOG_COLOR=true
Environment=OC_LOG_PRETTY=true
Environment=PROXY_TLS=false
Environment=OC_INSECURE=false
Environment=PROXY_ENABLE_BASIC_AUTH=false
Environment=IDM_CREATE_DEMO_USERS=false
Environment=OC_OIDC_ISSUER="https://keycloak.domain.tld/realms/{REALMNAME}"
Environment=WEB_OIDC_CLIENT_ID=OpenCloudWeb
Environment=OC_EXCLUDE_RUN_SERVICES=idp
Environment=PROXY_OIDC_REWRITE_WELLKNOWN="true"
Environment=PROXY_USER_OIDC_CLAIM=preferred_username
Environment=PROXY_USER_CS3_CLAIM=username
Environment=PROXY_AUTOPROVISION_ACCOUNTS=true
Environment=GRAPH_USERNAME_MATCH=none
Environment=PROXY_ROLE_ASSIGNMENT_DRIVER=oidc
Environment=PROXY_ROLE_ASSIGNMENT_OIDC_CLAIM=roles
Environment=GRAPH_ASSIGN_DEFAULT_USER_ROLE=false
Environment=PROXY_CSP_CONFIG_FILE_LOCATION=/etc/opencloud/csp.yaml
Environment=GATEWAY_GRPC_ADDR=0.0.0.0:9142
Environment=NATS_NATS_HOST=0.0.0.0
Environment=NATS_NATS_PORT=9233
#onlyoffice
Environment=COLLABORA_DOMAIN=onlyoffice.domain.tld
Environment=GRAPH_AVAILABLE_ROLES="b1e2218d-eef8-4d4c-b82d-0f1a1b48f3b5,a8d5fe5e-96e3-418d-825b-534dbdf22b99,fb6c3e19-e378-47e5-b277-9732f9de6e21,58c63c02-1d89-4572-916a-870abc5a1b7d,2d00ce52-1fc2-4dbc-8b95-a73b73395f5a,1c996275-f1c9-4e71-abdf-a42f6495e960,312c0871-5ef7-4b3a-85b6-0e4074c64049,aa97fe03-7980-45ac-9e50-b325749fd7e6"
#SMTP
Environment=START_ADDITIONAL_SERVICES=notifications
Environment=NOTIFICATIONS_SMTP_HOST=smtp.protonmail.ch
Environment=NOTIFICATIONS_SMTP_PORT=587
Environment=NOTIFICATIONS_SMTP_SENDER=opencloud@mail.tld
Environment=NOTIFICATIONS_SMTP_USERNAME=opencloud@mail.tld
Secret=opencloud_smtp_pw,type=env,target=NOTIFICATIONS_SMTP_PASSWORD
Environment=NOTIFICATIONS_SMTP_INSECURE=false
Environment=NOTIFICATIONS_SMTP_AUTHENTICATION=auto
Environment=NOTIFICATIONS_SMTP_ENCRYPTION=none
#sharing
Environment=OC_SHARING_PUBLIC_SHARE_MUST_HAVE_PASSWORD=false
Environment=OC_SHARING_PUBLIC_WRITEABLE_SHARE_MUST_HAVE_PASSWORD=false
#tika
Environment=SEARCH_EXTRACTOR_TYPE=tika
Environment=SEARCH_EXTRACTOR_TIKA_TIKA_URL=http://opencloud_tika:9998
Environment=FRONTEND_FULL_TEXT_SEARCH_ENABLED=true
#volumes
Volume=/container/opencloud/config:/etc/opencloud:z
Volume=/container/opencloud/data:/var/lib/opencloud:z
Volume=/etc/localtime:/etc/localtime:ro
PublishPort=9200:9200
Exec=server

[Service]
Restart=always

[Install]
WantedBy=multi-user.target default.target
```
## onlyoffice

```bash
nano /etc/containers/systemd/users/1000/opencloud_onlyoffice.container
```

```bash
[Unit]
Description=Opencloud Onlyoffice Service
Requires=opencloud_server.service
After=opencloud_server.service

[Container]
ContainerName=opencloud_onlyoffice
Image=docker.io/onlyoffice/documentserver:latest
Network=opencloud_net
Pod=opencloud.pod
AutoUpdate=registry
AddHost=wopi.domain.tld:{your_host_IP}
Environment=JWT_ENABLED=true
Environment=WOPI_ENABLED=true
Volume=/etc/localtime:/etc/localtime:ro
PublishPort=8084:80

[Service]
Restart=always

[Install]
WantedBy=multi-user.target default.target
```

## opencloud collaboration
```bash
nano /etc/containers/systemd/users/1000/opencloud_collaboration.container
```

```bash
[Unit]
Description=Opencloud Collaboration Service
After=opencloud_onlyoffice.service
Requires=opencloud_onlyoffice.service

[Container]
ContainerName=opencloud_collaboration
Image=docker.io/opencloudeu/opencloud:4
Network=opencloud_net
Pod=opencloud.pod
AutoUpdate=registry
AddHost=opencloud.domain.tld:{your_host_IP}
AddHost=onlyoffice.domain.tld:{your_host_IP}
Environment=COLLABORATION_GRPC_ADDR=0.0.0.0:9301
Environment=COLLABORATION_HTTP_ADDR=0.0.0.0:9300
Environment=MICRO_REGISTRY=nats-js-kv
Environment=MICRO_REGISTRY_ADDRESS=opencloud_server:9233
Environment=COLLABORATION_WOPI_SRC=https://wopi.domain.tld
Environment=COLLABORATION_APP_NAME=OnlyOffice
Environment=COLLABORATION_APP_PRODUCT=OnlyOffice
Environment=COLLABORATION_APP_ADDR=https://onlyoffice.domain.tld
Environment=COLLABORATION_APP_ICON=https://onlyoffice.domain.tld/favicon.ico
Environment=COLLABORATION_APP_INSECURE=false
Environment=COLLABORATION_CS3API_DATAGATEWAY_INSECURE=false
Environment=COLLABORATION_LOG_LEVEL=warn
Environment=OC_URL=https://opencloud.domain.tld
Volume=/container/opencloud/config:/etc/opencloud:z
Volume=/container/opencloud/data:/var/lib/opencloud:z
Volume=/etc/localtime:/etc/localtime:ro
PublishPort=9300:9300
Exec=collaboration server

[Service]
Restart=always
ExecStartPre=/bin/sleep 20

[Install]
WantedBy=multi-user.target default.target
```

# run the pod
```
systemctl --user daemon-reload && \
systemctl --user start opencloud-pod.service
```

# Info:
Don't forget to open the ports on the Firewall and add the services (opencloud_server, opencloud_collaboration, opencloud_onlyoffice) to the reverse proxy and set up the DNS provider.

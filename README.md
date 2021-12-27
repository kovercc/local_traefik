# Local Traefik

[Traefik](https://github.com/traefik/traefik) is the open-source edge router. You can use it as a reverse proxy for the services and applications in docker containers. This is the configuration and usage guide for running Traefik on the local computer in docker with self-signed SSL certificates.

## Requirements

Before starting, you need to install Docker and Docker-compose on your local machine (Docker desktop for windows).
Also, you need to install mkcert for using self-signed certificates:

```sh
mkcert -install
```

## Configure environment

- Create docker network named `proxy` (the network name can be any, just keep it for all services). This network will be common (external) for all services and applications which are proxied by the Traefik:

```sh
docker network create proxy
```

- Go to the folder `certs` in the shell and generate a self-signed certificate for Traefik dashboard with mkcert. In this example, a domain name is `traefik.local`. If you want to use another domain name, you also need to change it in the host rule of Traefik router in `docker-compose.yml` and pay attention to the domain name in the other steps

```sh
mkcert --cert-file traefik.local.crt --key-file traefik.local.key traefik.local
```

- Add generated certificate's and key's paths to the dynamic configuration file `tls_certificates.local.yml` in the `conf` folder. If you used domain 'traefik.local' for self-signed certificates, you don't need to do anything, because these paths are already in the file. If you used another domain name, you need to change `traefik.local` by your domain name in `tls_certificates.local.yml` file)

- Add the domain name of Traefik dashboard (`traefik.local`) into the `hosts` file (use ip of the localhost)

```text
127.0.0.1 traefik.local
```

## Run the local Traefik

After finishing with the configuration steps open root folder of the local_traefik repository in the shell and run the command

```sh
docker-compose up --build -d
```

After that, open `https://traefik.local` in browser to see Traefik dashboard. If the browser does not complain about the certificate, then everything is done correctly.

## Connect containerized services to Traefik

In a typical scenario, we have the external docker network, the container with Traefik, and several containers with services and applications.
After starting Traefik, we can connect clients to it **without restarting it**.
For example, we want to connect service `testapp` with domain `testapp.local` to our local_traefik.
For doing that, do these steps:

### Configuration on the operation system side

Add the domain name of service (`testapp.local` in our example) into the `hosts` file (use ip of the localhost):

```text
127.0.0.1 testapp.local
```

### Configuration on the Traefik side

- Go to the folder `certs` in the shell and generate a self-signed certificate for the `testapp.local` domain name with mkcert:

```sh
mkcert --cert-file testapp.local.crt --key-file testapp.local.key testapp.local
```

- Add generated certificate's and key's paths to the dynamic configuration file `tls_certificates.local.yml` in the `conf` folder:

```yml
        ...        
        -   certFile: "/etc/traefik/certs/testapp.local.crt"
            keyFile: "/etc/traefik/certs/testapp.local.key"
```

**One more time:** you don't need to restart your local_traefik container after these steps, the configuration applies dynamically.

### Configuration on the Service side

- Define the external docker network `proxy` (common with traefik) in the `docker-compose.yml` file:

```yml
networks:
  proxy:
    external: true
```

- Set network `proxy` for service configuration in the `docker-compose.yml` file:

```yml
  testapp:
    networks:
      - proxy
    ...
```

- Add labels for the service configuration in the `docker-compose.yml` file:

```yml
 labels:
      - traefik.enable=true
      - traefik.http.routers.{ROUTER_NAME}.service={SERVICE_NAME}
      - traefik.http.routers.{ROUTER_NAME}.entrypoints=websecure
      - traefik.http.routers.{ROUTER_NAME}.rule=Host(`testapp.local`) 
      - traefik.http.routers.{ROUTER_NAME}.tls=true
      - traefik.http.services.{ROUTER_NAME}.loadbalancer.server.port=80     
```

The whole `docker-compose.yml` file:

```yml
  testapp:
    networks:
      - proxy
    image: testapp
    container_name: "local_testapp"
    build:
      context: .
      dockerfile: TestApp/Dockerfile
    env_file: 
      - .env
    labels:
      - traefik.enable=true
      - traefik.http.routers.testapp-router.service=testapp-service
      - traefik.http.routers.testapp-router.entrypoints=websecure
      - traefik.http.routers.testapp-router.rule=Host(`testapp.local`) 
      - traefik.http.routers.testapp-router.tls=true
      - traefik.http.services.testapp-router.loadbalancer.server.port=80     
networks:
  proxy:
    external: true
```

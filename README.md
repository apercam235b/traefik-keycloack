# traefik_test
En este post vamos a ver como podemos autenticarnos en aplicaciones a traves de Keycloak usando traefik como middleware

Vamos a necesitar 3 contenedores:

Traefik: Proxy donde vamos a configurar el middleware.
Auth-Forward: Nos hara de intermediramos para la Autenticacion.
Keycloak: Hara los servicios de autenticacion.

## Contenedores

En el propio repositorio vamos a tener los docker compose necesarios para los contenedores 

## Traefik

Vamos a ver la configuracion necesaria para hacer funcionar traefik:

```
version: "3"
 
networks: 
  LAN: 
    external: true

services: 
    traefik: # The official v2 Traefik docker image 
        container_name: traefik 
        image: traefik:latest  
        command: 
            - "--entryPoints.pruebas.address=:80" 
            - "--entryPoints.pruebas_seguro.address=:443" 
            - "--api.insecure=true" 
            - "--providers.docker" 
            - "--providers.docker.exposedByDefault=false"
            - "--providers.file.filename=/etc/traefik/traefik.toml" # Configuracion estatica 
        # 	- "--providers.file.directory=/etc/traefik/" # Para indicarle la configuración dinamica
        restart: always 
        ports: 
        	- 80:80 # HTTP port 
        	- 8080:8080 # Dashboard port 
        	- 443:443 # Https port 
        volumes: 
       		# So that Traefik can listen to the Docker events 
       		- /var/run/docker.sock:/var/run/docker.sock 
       		- ./config:/etc/traefik/ 
        networks: 
            - LAN 
        labels: 
          		- "traefik.enable=true" 
          		- "traefik.docker.network=LAN" 
          		- "traefik.http.routers.traefik.rule=Host(`traefik.alvaro.civica.lab`)" 
          		- "traefik.http.services.traefik.loadbalancer.server.port=8080" 
          		- "traefik.http.routers.traefik.entrypoints=pruebas" 
                # Con la siguiente linea hariamos que hiciera falta authenticarse para ver el dashboard de Traefik.       
                - "traefik.http.routers.traefik.middlewares=traefik-forward-auth@docker"



```

## Forward Auth


```
   forwardauth: 
        image: thomseddon/traefik-forward-auth:2 
        container_name: forward-auth 
        networks: 
            - LAN 
        ports: 
            - 4181:4181 
        environment: 
            - DEFAULT_PROVIDER=oidc 
            - PROVIDERS_OIDC_ISSUER_URL=http://keycloak.alvaro.civica.lab/realms/prueba 
            - PROVIDERS_OIDC_CLIENT_ID=auth 
            - PROVIDERS_OIDC_CLIENT_SECRET=a3LlxXbhvUjXQMBrj5Ztyo9xNyGzud6j 
                # INSECURE_COOKIE is required if not using a https entrypoint 
            - INSECURE_COOKIE=true 
            - LOG_LEVEL=debug 
            - SECRET=dhfbvgsahodgbvsdhsdhfs 
        labels: 
            - "traefik.enable=true" 
            - "traefik.docker.network=LAN"
            - "traefik.http.services.forwardauth.loadbalancer.server.port=4181" 
            - "traefik.http.routers.forwardauth.entrypoints=pruebas" 
            - "traefik.http.routers.forwardauth.rule=Path(`/_oauth`)" 
            - "traefik.http.routers.forwardauth.middlewares=traefik-forward-auth" 
            - "traefik.http.middlewares.traefik-forward-auth.forwardauth.address=http://forward-auth:4181" 
            - "traefik.http.middlewares.traefik-forward-auth.forwardauth.authResponseHeaders=X-Forwarded-User" 
            - "traefik.http.middlewares.traefik-forward-auth.forwardauth.trustForwardHeader=true" 
            # - traefik.http.routers.keycloak.tls=true 
            # - traefik.http.routers.keycloak.tls.certresolver=letsencrypt 

```

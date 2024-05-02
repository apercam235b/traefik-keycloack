# traefik_test
En este post vamos a ver como podemos autenticarnos en aplicaciones a traves de Keycloak usando traefik como middleware

Vamos a necesitar 3 contenedores:

Traefik: Proxy donde vamos a configurar el middleware.
Auth-Forward: Nos hara de intermediramos para la Autenticacion.
Keycloak: Hara los servicios de autenticacion.

## Traefik

Empezamos levantando lo basico de Traefik con el que vamos a ir sumandole cosas para que todo fucione bien al final:

El dominio que yo voy a usar va a ser alvaro.civica.lab

Esto no esta montando de momento con SSL, ya habra en algun futuro no muy lejano alguna actualizacion.


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
            - "--api.insecure=true" 
            - "--providers.docker" 
            - "--providers.docker.exposedByDefault=false"
        restart: always 
        ports: 
        	- 80:80 # HTTP port 
        	- 8080:8080 # Dashboard port 
        volumes: 
       		# So that Traefik can listen to the Docker events 
       		- /var/run/docker.sock:/var/run/docker.sock 
        networks: 
            - LAN 
        labels: 
          		- "traefik.enable=true" 
          		- "traefik.docker.network=LAN" 
          		- "traefik.http.routers.traefik.rule=Host(`traefik.alvaro.civica.lab`)" 
          		- "traefik.http.services.traefik.loadbalancer.server.port=8080" 
          		- "traefik.http.routers.traefik.entrypoints=pruebas" 

```
Con esto ya tendriamos montado un contenedor con nuestro traefik fucionando.

### Configuracion Estatica y Dinamica o Configuracion por Socket:

Hay dos maneras para que traefik aprenda y pueda gestionar contenedores de docker, a traves de label y que aprenda del propio socket de docker, o mediantes archivos de configuracion, que tenemos dos tipos de configuración.
#### Estática:
Vamos a indicarle un fichero donde tendremos configuración sobre nuestro traefik. Para esta configuracion es necesario pasarle como command la siguiente linea:
    
```
        - "--providers.file.filename=/etc/traefik/traefik.toml" # Configuracion estatica 
```
    
Es importante crear el fichero y mapearlo con un volumen que tengamos en nuestro equipo

Dentro de este fichero entre otras configuraciones, podemos indicar los entrypoint, asi como el socket de docker, y tambien sera necesario añadir donde se va a ubicar la configuracion dinámica:

```
## STATIC CONFIGURATION
log:
  level: INFO

api:
  insecure: true
  dashboard: true

entryPoints:
  pruebas:
    address: ":80"
  pruebas_seguro:
    address: ":443"

providers:
  docker:
    endpoint: "unix:///var/run/docker.sock"
    exposedByDefault: false
  file:
    directory: "/etc/traefik/"
```

Lo indicaremos con providers  ->  file  ->  directory  ->  "directorio donde tendremos nuestras configuraciones dinámicas"

#### Dinámica:

Para la configuracion dinamica, una buena practica puede ser crear 1 fichero por cada servicio que queremos que nuestro traefik filtre, para ello un ejemplo de como configurar un fichero, en mi caso va a ser un contenedor de uptime-kuma:

```
http:
  routers:
    uptime-kuma:
      rule: "Host(`uptime.alvaro.civica.lab`)"
      service: uptime-kuma
        priority: 1000
      entryPoints:
        - pruebas

services:
  uptime-kuma:
    loadBalancer:
      servers:
        - url: "http://192.168.99.198:3001"
```
Con esta configuración tendremos nuestro servicio configurado para que sea reconocido por traefik.

Cuando le indicamos la ip, si fuera necesario, tendriamos que agregarle el puerto que usaria nuestro contenedor.

![image](https://github.com/apercam235b/traefik-keycloack/assets/146701978/d10e353e-21fa-480f-95ca-7ccf7f2d3242)



## Keycloak

Ahora vamos a levantar el contendor de keycloak, podemos levantarlo dentro del mismo docker compose de Traefik, pero mi experiencia es mejor hacerlo en un contenedor a parte.

```
services:
networks:
  LAN:
    external: true

  keycloak:
    image: quay.io/keycloak/keycloak:latest
    container_name: keycloak
    dns: #En mi caso las DNS son necesarias por que mis usuarios son importados de un ldap
      - 192.168.10.11
      - 172.17.24.5
      - 8.8.8.8
    command: start-dev
    environment:
      KEYCLOAK_ADMIN: admin
      KEYCLOAK_ADMIN_PASSWORD: admin
      KC_HTTP_PORT: 8088
    ports:
      - 8088:8088
    networks:
      - LAN
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.keycloak.rule=Host(`keycloak.alvaro.civica.lab`)"
      - "traefik.http.routers.keycloak.entrypoints=pruebas"
      - "traefik.docker.network=LAN"
      - "traefik.http.services.keycloak.loadbalancer.server.port=8088"
    volumes:
      - ./keycloak-db:/opt/keycloak/data/h2

```

Una vez dentro de Keycloak tendremos que iniciar sesion, en este caso el usuario admin y pass admin, que tendremos que cambiar como es logico.

En Keycloak tendremos que crear un client, para ello voy a dejar la configuracion basica y varios datos que vamos a necesitar mas adelante, para saber localizarlos.

### Client
El nombre que le ponemos va a ser necesario mas adelante, en nuestro caso es auth
![image](https://github.com/apercam235b/traefik-keycloack/assets/146701978/65921c69-f219-495e-99b5-6b16722c49ae)

#### Configuración del Client (auth)

![image](https://github.com/apercam235b/traefik-keycloack/assets/146701978/e366efef-9bd4-40e9-a6f6-04a64b3ed00f)

![image](https://github.com/apercam235b/traefik-keycloack/assets/146701978/10a20393-eb0e-4e4b-9768-8b03f5b2c353)

Importante en el apartado de "Credentials" vamos a necesitar el secret.

![image](https://github.com/apercam235b/traefik-keycloack/assets/146701978/14666cf2-bbb0-48f4-82b9-61c49e85eb3d)




----------------------------------------
----------------------------------------

```
version: "3" 
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

networks: 
  LAN: 
    external: true 

```

# Traefik

Ce depot permet de décrire une utilisation générique de Traefik

## Docker Swarm

Sur l'exemple donné, traefik a été configuré sur un cluster docker swarm constitué de 2 raspberry-pi et d'une machine AMD64.

Le raspberry-pi est la machine frontale utilisée depuis internet.

Le configuration docker swarm est transposable sur docker-compose (utilisation mono-machine)

## Configuration

Traefik lit sa configuration depuis un fichier dans `/etc/traefik/traefik.yaml`

```yaml
providers:
    # docker-compose ou docker swarm 
    docker:
        endpoint: unix:///var/run/docker.sock
        watch: true
        exposedByDefault: false
    # configuration dans le fichier 
    file:
      filename: /etc/traefik/dynamic.yaml
```

## docker-compose.yaml ou traefik.yaml

Suivant que nous utilisons traefik sur une architecture mono machine, ou docker swarm nous nommerons le fichier docker-compose.yaml ou traefik.yaml

* **traefik.yaml**

```yaml
version: '3'
services:

  Traefik:
    image: traefik:v2.4.8
    # Publication des ports (8080 est le port d'administration)
    ports:
      - 80:80
      - 443:443
      - 8080:8080
    deploy:
      labels:
        # Utiliser le réseau Backend
        - traefik.docker.network=Backend
        - traefik.enable=true
        # <url> permet d'accéder à traefik
        - traefik.http.routers.Traefik.rule=Host(`<url>`)
        - traefik.http.services.Traefik.loadbalancer.server.port=8080
        - traefik.http.routers.Traefik.entrypoints=https
        - traefik.http.routers.Traefik.tls=true
        # <le> désigne let's encrypt
        - traefik.http.routers.Traefik.tls.certResolver=le
        # redirection du traffic http -> https
        - traefik.http.routers.Traefik-http.entryPoints=http
        - traefik.http.routers.Traefik-http.rule=Host(`<url>`)
        - traefik.http.routers.Traefik-http.middlewares=redirect-to-https@docker 
      placement:
        # Déployer uniquement sur un manager
        constraints:
           - node.role == manager
    volumes:
      # Liaison avec docker 
      - /var/run/docker.sock:/var/run/docker.sock
      # Fichiers de configuration
      - /Data/Apps/Traefik/:/etc/traefik
      # Fichiers temporaires
      - /Data/Apps/Traefik/tmp:/tmp
      # Stockage des fichiers Let's encrypt
      - /Data/Apps/Traefik/letsencrypt:/letsencrypt
    networks:
      # Connecté au réseau extBackend
      - extBackend

networks:
  # Le réseau extBackend est un réseau défini à l'extérieur de ce fichier, il s'appelle 'Backend'
  extBackend:
    external: 
      name: Backend
```

* **docker-compose.yaml**

```yaml
version: '3'
services:

  Traefik:
    image: traefik:v2.4.8
    # Publication des ports (8080 est le port d'administration)
    ports:
      - 80:80
      - 443:443
      - 8080:8080
    labels:
      - traefik.enable=true
      # <url> permet d'accéder à traefik
      - traefik.http.routers.Traefik.rule=Host(`<url>`)
      - traefik.http.services.Traefik.loadbalancer.server.port=8080
      - traefik.http.routers.Traefik.entrypoints=https
      - traefik.http.routers.Traefik.tls=true
      # <le> désigne let's encrypt
      - traefik.http.routers.Traefik.tls.certResolver=le
      # redirection du traffic http -> https
      - traefik.http.routers.Traefik-http.entryPoints=http
      - traefik.http.routers.Traefik-http.rule=Host(`<url>`)
      - traefik.http.routers.Traefik-http.middlewares=redirect-to-https@docker 
    volumes:
      # Liaison avec docker 
      - /var/run/docker.sock:/var/run/docker.sock
      # Fichiers de configuration
      - /Data/Apps/Traefik/:/etc/traefik
      # Fichiers temporaires
      - /Data/Apps/Traefik/tmp:/tmp
      # Stockage des fichiers Let's encrypt
      - /Data/Apps/Traefik/letsencrypt:/letsencrypt
```

Traefik peut être configuré sur base d'un fichier (relu dynamiquement)

**dynamic.yaml**

```yaml
http:
  routers:
    router-homeassistant:
      entryPoints:
        - https
      rule: Host(`ha.123.com`)
      service: service-homeassistant
      tls:
        certResolver: le
    router-blueiris:
      entryPoints:
        - https
      rule: Host(`bi.123.com`)
      service: service-blueiris
      tls:
        certResolver: le
  services:
    service-homeassistant:
      loadBalancer:
        servers:
        - url: "http://192.168.1.99:8123"
    service-blueiris:
      loadBalancer:
        servers:
        - url: "http://192.168.1.200:81"
``̀

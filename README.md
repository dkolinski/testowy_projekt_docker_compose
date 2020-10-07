# Testowy Docker Compose

- Środowisko Testowe : CentOS Linux release 7.5.1804
- Wersja Docker: Client: Docker Engine - Community : Version: 19.03.13
- Wersja Docker Compose: docker-compose version 1.25.0, build 0a186604

Przed przystąpieniem do uruchomienia: `docker-compose up lub docker-compose up -d` należy uzupełnić plik `.env` o poprawną domenę i email, dodać plik `index.html` oraz stworzyc sieć dla kontenerów. 

Docker-Compose był testowany na vps'ie z dostępem do internetu i publicznym IP oraz przypisaną domeną (bez i z www). Firewall'a na portach 80 i 443 został wyłączony.

przykład pliku `.env`:

```
DOMAIN=example.com
EMAIL=admin@example.com
```

Tworzymy testowy plik `index.html` dla nginx'a : `mkdir -p nginx-data/html/ && echo "test123" > nginx-data/html/index.html`

Tworzymy sieć dla kontenerów: `docker network create proxy`

Po uzupełnieniu odpalamy polecenie: `docker-compose up lub docker-compose up -d`, następnie `docker ps` i `docker logs nazwa_kontenera` jeżeli, któryś kontener nie wstanie w celu weryfikacji.

# Struktura plików po odpaleniu:

```
.
├── docker-compose.yml
├── .env
├── letsencrypt
│   └── acme.json
├── nginx-data
│   └── html
│       └── index.html
├── postgres-data
│   └── var
│       └── lib
│           └── postgresql
│               └── data
├── python-data
└── redis-data
    └── appendonly.aof
```

Autogenerowalny certyfikat letsencrypt zostaje zachowany w pliku: 
```
├── letsencrypt-
│   └── acme.json
```

# Weryfikacja działania

Po uruchomieniu kontenera możemy zweryfikować działanie:

 Testowe wejście na kontener:
   - `docker exec -it nazwa_kontenera /bin/bash lub /bin/sh`

 Sprawdzenie działania sieci: 
    - `docker exec -ti nazwa_kontenera_1 ping nazwa_kontenera_2`

 Dla kontenera nginx:
    - `curl -IL -vvv http://example.com`
    - `curl -IL -vvv http://www.example.com`
    - `curl -H 'Cache-Control: no-cache'  -s -L -D -v http://example.com -o /dev/null -w '%{url_effective}'`
    - `curl -H 'Cache-Control: no-cache'  -s -L -D -v http://www.example.com -o /dev/null -w '%{url_effective}'`

 Dla kontenera pythonowego (tutaj niestety nie wiedziałem jak przekierować port 8000 na https ponieważ pierwszy raz spotkałem się z narzędziem traefik):
    - `curl http://example.com:8000`
    - `curl http://www.example.com:8000`
    - `curl -IL -vvv http://example.com:8000`
    - `curl -IL -vvv http://www.example.com:8000`

# Opis Docker Compose

Opisy w komentarzach

```
version: "3"

### Definicja Sieci
networks:
  proxy:
    external: true

### Definicja Serwisow
services:

  traefik:
    image: "traefik:latest" ### Używałem ostatniej wersji ponieważ, korzystałem z dokumentacji dla ostatniej wersji traefik, zrezygnowałem z wersji alpine ponieważ część funkcji nie działała, plus wersja pełna nie jest zbyć ciężka (92,4MB)
    container_name: "traefik-router"
    env_file:
      - ".env"
    command:
      - "--log.level=DEBUG"
      - "--providers.docker=true" 
      - "--providers.docker.exposedbydefault=true"
      - "--entrypoints.web.address=:80"
      - "--entrypoints.websecure.address=:443"
      - "--entrypoints.python.address=:8000"
      # Autogenerowanie certyfikatów ssl
      - "--certificatesresolvers.le.acme.httpchallenge=true"
      - "--certificatesresolvers.le.acme.httpchallenge.entrypoint=web"
      - "--certificatesresolvers.le.acme.email=${EMAIL}"
      - "--certificatesresolvers.le.acme.storage=/letsencrypt/acme.json"
    labels:
      traefik.enable: "true"
      traefik.docker.network: "proxy"
      # Globalny redirect - http na https
      traefik.http.routers.http-catchall.rule: "hostregexp(`{host:(www\\.)?.+}`)"
      traefik.http.routers.http-catchall.entrypoints: web
      traefik.http.routers.http-catchall.middlewares: redirect-to-https
      # Globalny redirect - https://www to https
      traefik.http.routers.https-catchall.rule: "hostregexp(`{host:(www\\.).+}`)"
      traefik.http.routers.https-catchall.entrypoints: websecure
      traefik.http.routers.https-catchall.tls: "true"
      # Middleware redirect
      traefik.http.middlewares.redirect-to-https.redirectregex.regex: "^https?://(?:www\\.)?(.+)"
      traefik.http.middlewares.redirect-to-https.redirectregex.replacement: "https://$${1}"
      traefik.http.middlewares.redirect-to-https.redirectregex.permanent: "true"
    ports:
      - "80:80"
      - "443:443"
      - "8000:8000"
    networks:
      - "proxy"
    volumes:
      - "./letsencrypt:/letsencrypt"
      - "/var/run/docker.sock:/var/run/docker.sock:ro"

######## Tester przekierowania i certyfikatów ######################
#  whoami:
#   image: traefik/whoami
#   container_name: "whoami_checker"
#   labels:
#      - "traefik.enable=true"
#      - "traefik.http.routers.whoami.rule=Host(`${DOMAIN}`)"
#      - "traefik.http.routers.whoami.rule=Host(`www.${DOMAIN}`)"
#      - "traefik.http.routers.whoami.entrypoints=websecure"
#      #- "traefik.http.routers.whoami.tls.certresolver=le"
#   env_file:
#     - ".env"
#   networks:
#     - "proxy"
####################################################################
  web-nginx:
    image: nginx:alpine # Wybrałem lekką wersję alpine ponieważ bazowy obraz nginx'a jest o wiele większy (alpine 22,3MB a pełny 133MB), a podstawowe funkcjonalności w tym wypadku w zupełności wystarczą
    restart: always
    container_name: "web-nginx"
    volumes:
       - "./nginx-data:/usr/share/nginx:ro"
    networks:
      - "proxy"
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.nginx.rule=Host(`${DOMAIN}`)"
      - "traefik.http.routers.nginx.tls=true"
      - "traefik.http.routers.nginx.tls.certresolver=le"
      - "traefik.http.services.nginx.loadbalancer.server.port=80"
      - "traefik.docker.network=proxy"
    env_file:
      - ".env"

  python: 
    image: python:3.8-slim-buster ## 113MB - > pełny obraz : 886MB
    restart: always
    container_name: "python-slim-buster"
    command: python -m http.server 8000
    volumes:
      - "./python-data:/python-data"
    labels:
      - "traefik.enable=true"
      - "traefik.tcp.routers.python.rule=HostSNI(`*`)"
      - "traefik.tcp.routers.python.entrypoints=python"
      - "traefik.tcp.routers.python.service=python"
      - "traefik.tcp.services.python.loadbalancer.server.port=8000"
      - "traefik.docker.network=proxy"
    networks:
      - "proxy"
    env_file:
      - ".env"
    depends_on:
      - postgres-database

  postgres-database: # Wybrałem lekką wersję alpine ponieważ bazowy obraz postgresql'a jest o wiele większy (alpine 160MB a pełny 314MB), a podstawowe funkcjonalności w tym wypadku w zupełności wystarczą
   image: postgres:alpine
   restart: always
   container_name: "postgres-database"
   volumes:
      - "./postgres-data/var/lib/postgresql:/var/lib/postgresql"
   environment:
     POSTGRES_USER: test_user
     POSTGRES_PASSWORD: test123
     POSTGRES_DB: test_database
   networks:
     - "proxy"

  redis-database: # -||- analogicznie jak przy wcześniejszych: wersja alpine 32,2MB , pełna 104 MB
      image: redis:alpine
      restart: always
      container_name: "redis-database"
      command: redis-server --appendonly yes
      volumes:
        - "./redis-data:/data"
      networks:
        - "proxy"
```        

### create internal network between the nginx proxy and the service
```sh
docker network create chaching
```

### run nginx and bind to 8080. traffic is internally hit on 8090 port. 8090 port has custom nginx `location` directive
```sh
docker run --name my-custom-nginx-container --rm -v /tmp/nginx.conf:/etc/nginx/conf.d/server.conf:ro --network chaching -p 8080:8090 nginx
```

### file serving python server. acts as the nginx backend service to which nginx proxies requests to
```sh
docker run --rm -ti -p 8000:8000 -v /tmp/test:/app --network chaching python:3.9 sh -c "cd /app && python3 -m http.server"
```

### test from you localhost
curl -vvv localhost:8080/11.10-15.10.20.pdf -o bla.pdf

### video of testing (see the X-Cache header changes over time)
https://asciinema.org/a/kWh0HUN61Olg8nsPi1VamTr9T

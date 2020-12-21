# create internal network between the nginx proxy and the service
docker network create chaching

# run nginx and bind to 8080. traffic is internally hit on 8090 port. 8090 port has custom nginx `location` directive
docker run --name my-custom-nginx-container --rm -v /tmp/nginx.conf:/etc/nginx/conf.d/server.conf:ro --network chaching -p 8080:8090 nginx

# file serving python server. acts as the nginx backend service to which nginx proxies requests to
docker run --rm -ti -p 8000:8000 -v /tmp/test:/app --network chaching python:3.9 sh -c cd /app && python3 -m http.server

# test from you localhost
curl -vvv localhost:8080/11.10-15.10.20.pdf -o bla.pdf

# video of testing (see the X-Cache header changes over time)
https://asciinema.org/a/kWh0HUN61Olg8nsPi1VamTr9T

# nginx.conf (note that proxy_pass is directed to the named container http:///ppp:8000) (note that the cache is evicted after 20s)
```
	# Set cache dir
	proxy_cache_path /var/cache/nginx levels=1:2 keys_zone=microcache:5m max_size=1000m;

	# Virtualhost/server configuration
	server {
	    listen   8090;
	
	    # Define cached location (may not be whole site)
	    location / {
	        # Setup var defaults
	        set $no_cache "";
	        # If non GET/HEAD, don't cache & mark user as uncacheable for 1 second via cookie
	        if ($request_method !~ ^(GET|HEAD)$) {
	            set $no_cache "1";
	        }
	        # Drop no cache cookie if need be
	        # (for some reason, add_header fails if included in prior if-block)
	        if ($no_cache = "1") {
	            add_header Set-Cookie "_mcnc=1; Max-Age=2; Path=/";            
	            add_header X-Microcachable "0";
	        }
	        # Bypass cache if no-cache cookie is set
	        if ($http_cookie ~* "_mcnc") {
	            set $no_cache "1";
	        }
	        # Bypass cache if flag is set
	        proxy_no_cache $no_cache;
	        proxy_cache_bypass $no_cache;
	        # Point nginx to the real app/web server
	        proxy_pass http://ppp:8000;
	        # Set cache zone
	        proxy_cache microcache;
	        # Set cache key to include identifying components
	        proxy_cache_key $scheme$host$request_method$request_uri;
	        # Only cache valid HTTP 200 responses for 20 second
	        proxy_cache_valid 200 20s;
	        # Serve from cache if currently refreshing
	        proxy_cache_use_stale updating;
	        # Send appropriate headers through
	        proxy_set_header X-Real-IP $remote_addr;
	        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
	        # Set files larger than 1M to stream rather than cache
	        proxy_max_temp_file_size 100M;
                # Set header for cache status 
                add_header X-Cache-Status $upstream_cache_status;
	    }
	}
```

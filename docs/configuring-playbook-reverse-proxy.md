# Configuring the reverse-proxy

By default, this playbook installs its own [Traefik](https://traefik.io/) webserver (in a Docker container) which listens on ports 80 and 443.
If that's alright, you can skip this.


## Introduction

The default flow is like this: (Traefik -> `nextcloud-reverse-proxy-companion` -> `nextcloud-apache`).

If you don't want this playbook's Traefik webserver to take over your server's 80/443 ports like that,
and you'd like to use your own webserver (be it your own Traefik installed in another way, nginx, Apache, Varnish Cache, etc.), you can.

Whichever reverse-proxy you decide on using, we recommend that you keep the last part the same (`nextcloud-reverse-proxy-companion` -> `nextcloud-apache`) the same.


## Using your own Traefik server (installed separately)

If you'd like to avoid the playbook installing its own Traefik server and instead use your own, use this configuration:

```yaml
nextcloud_playbook_traefik_installation_enabled: false

nextcloud_container_network: 'YOUR_TRAEFIK_NETWORK'
```

The `nextcloud-reverse-proxy-companion` container has container labels attached, so that a Traefik instance can reverse-proxy to it. See `roles/custom/nextcloud_reverse_proxy_companion/templates/labels.j2`.


# Disabling Traefik

To disable Traefik, use:

```yaml
# Disable Traefik
nextcloud_playbook_traefik_installation_enabled: false

# If you're not using Traefik, you can also avoid attaching container labels
# to the `nextcloud-reverse-proxy-companion` container, although attaching labels won't hurt.
nextcloud_reverse_proxy_companion_container_labels_traefik_enabled: false
```

This disables traffic, but leaves `nextcloud-reverse-proxy-companion` installed. It doesn't expose `nextcloud-reverse-proxy-companion`'s ports though.


## Using nginx


### Using nextcloud-nginx-proxy for SSL termination

To disable Traefik and use `nextcloud-nginx-proxy` instead, use the following configuration:

```yaml
nextcloud_playbook_traefik_installation_enabled: false

# If you're not using Traefik, you can also avoid attaching container labels
# to the `nextcloud-reverse-proxy-companion` container, although attaching labels won't hurt.
nextcloud_reverse_proxy_companion_container_labels_traefik_enabled: false

nextcloud_playbook_nginx_proxy_installation_enabled: true
```

This will install `nextcloud-nginx-proxy` and make it terminate SSL automatically.

The flow would be: (`nextcloud-nginx-proxy` -> `nextcloud-reverse-proxy-companion` -> `nextcloud-apache`).

If you'd like to provide your own certifiates, consider tweaking `nextcloud_nginx_proxy_ssl_retrieval_method`, etc.


#### Using nextcloud-nginx-proxy to generate configuration for your own, other, nginx proxy

You can also make use of the the `nextcloud-nginx-proxy` role to retrieve SSL certificates and generate *some* configuration, which you later plug into your own nginx server running on the host.

All it takes is:

1) making sure your web server user (something like `http`, `apache`, `www-data`, `nginx`) is part of the `nextcloud` group. You should run something like this: `usermod -a -G nextcloud nginx`

2) editing your configuration file (`inventory/nextcloud.<your-domain>/vars.yml`):

To do this, use the following configuration:

```yaml
nextcloud_playbook_traefik_installation_enabled: false

# If you're not using Traefik, you can also avoid attaching container labels
# to the `nextcloud-reverse-proxy-companion` container, although attaching labels won't hurt.
nextcloud_reverse_proxy_companion_container_labels_traefik_enabled: false

nextcloud_playbook_nginx_proxy_installation_enabled: true

# Make nextcloud-nginx-proxy generate a configuration that can be imported into a locally running nginx.
# Such a locally running nginx instance will also be configured to talk to nextcloud-reverse-proxy-companion locally.
nextcloud_nginx_proxy_service_reach_mode: local

# Make nextcloud-nginx-proxy's configuration point to nextcloud-reverse-proxy-companion's local port.
# Also see nextcloud_reverse_proxy_companion_http_bind_port below.
nextcloud_nginx_proxy_service_reach_local_port: 37150

# Expose nextcloud-reverse-proxy-companion locally, on port 37150
nextcloud_reverse_proxy_companion_http_bind_port: 127.0.0.1:37150
```

**Note**: even if you do this, in order [to install](installing.md), this playbook still expects port 80 to be available. **Please manually stop your other webserver while installing**. You can start it back again afterwards.

**If your own webserver is nginx**, you can most likely directly use the config files installed by this playbook at: `/nextcloud/nginx-proxy/conf.d`. Just include them in your `nginx.conf` like this: `include /nextcloud/nginx-proxy/conf.d/*.conf;`

**If your own webserver is not nginx**, you can still take a look at the sample files in `/nextcloud/nginx-proxy/conf.d`, and:

- ensure you set up a vhost that proxies to Nextcloud (`localhost:37150`)

- ensure that the `/.well-known/acme-challenge` location for the "port=80 vhost" gets proxied to `http://localhost:2403` (controlled by `nextcloud_nginx_proxy_ssl_certbot_standalone_http_port`) for automated SSL renewal to work

- ensure that you restart/reload your webserver once in a while, so that renewed SSL certificates would take effect (once a month should be enough)


## Using another reverse-proxy

This method is about first [disabling Traefik](#disabling-traefik), which then allows you to reverse-proxy by yourself.

```yaml
nextcloud_playbook_traefik_installation_enabled: false

# If you're not using Traefik, you can also avoid attaching container labels
# to the `nextcloud-reverse-proxy-companion` container, although attaching labels won't hurt.
nextcloud_reverse_proxy_companion_container_labels_traefik_enabled: false

# Expose the nextcloud-reverse-proxy-companion container's webserver to port 37150 on the loopback network interface only.
# You can reverse-proxy to it using a locally running webserver.
nextcloud_reverse_proxy_companion_http_bind_port: '127.0.0.1:37150'

# Or:
# Expose the nextcloud-reverse-proxy-companion container's webserver to port 37150 on all network interfaces.
# You can reverse-proxy to it from another machine on the public or private network.
# nextcloud_reverse_proxy_companion_http_bind_port: '37150'
```

You can then reverse-proxy from your own webserver to the `nextcloud-reverse-proxy-companion` container on port `37150`.
# ovpn-admin

Simple web UI to manage OpenVPN users, their certificates & routes in Linux. While backend is written in Go, frontend is based on Vue.js.

***DISCLAIMER!** This project was created for experienced users (system administrators) and private (e.g., protected by network policies) environments only. Thus, it is not implemented with security in mind (e.g., it doesn't strictly check all parameters passed by users, etc.). It also relies heavily on files and fails if required files aren't available.*

## Features

* Adding, deleting OpenVPN users (generating certificates for them);
* Revoking/restoring/rotating users certificates;
* Generating ready-to-user config files;
* Providing metrics for Prometheus, including certificates expiration date, number of (connected/total) users, information about connected users;
* (optionally) Specifying CCD (`client-config-dir`) for each user;
* (optionally) Operating in a master/slave mode (syncing certs & CCD with other server);
* (optionally) Specifying/changing password for additional authorization in OpenVPN;
* (optionally) Specifying the Kubernetes LoadBalancer if it's used in front of the OpenVPN server (to get an automatically defined `remote` in the `client.conf.tpl` template).
* (optionally) Storing certificates and other files in Kubernetes Secrets (**Attention, this feature is experimental!**).

## This fork

This fork ([weekends/ovpn-admin](https://github.com/weekends/ovpn-admin)) adds a few
things on top of upstream [palark/ovpn-admin](https://github.com/palark/ovpn-admin),
mostly driven by running it on a small (1 vCPU / ~1GB RAM), 32-bit (i586) production
server behind a reverse proxy:

* **Fixed a crash** in CCD parsing (`parseCcd()`): the original code indexed into
  `push "..."` directive fields without checking the field count first, so any
  `push` line that wasn't exactly `push "route ADDR MASK"` (e.g. a hand-added
  `push "redirect-gateway def1"`) would panic the whole request handler with
  `index out of range`.
* **Per-user full-tunnel toggle.** A `Full tunnel` checkbox in the "Edit routes"
  modal writes `push "redirect-gateway def1"` into that user's CCD file, instead of
  needing to hand-edit CCD files to get this (previously-crashing) behavior.
* **Live traffic stats.** The "Active Connections" column now shows, per connected
  session, the virtual + real IP and live RX/TX byte counts (`↓ x MB / ↑ y MB`),
  plus a "Total live traffic" summary line above the table. This is live/current-session
  data only (from the OpenVPN management interface) — it resets whenever a client
  reconnects, there's no historical/cumulative storage across sessions.
* **Light/dark theme toggle**, persisted in the browser's `localStorage`, defaulting
  to the OS/browser's `prefers-color-scheme` on first load.
* **PEM extraction safety.** Cert/key/CA/tls-auth content is now extracted via regex
  (`-----BEGIN ...----- ... -----END ...-----`) before being embedded into generated
  client `.ovpn` files, instead of embedding the raw easy-rsa output file verbatim —
  defensive against easy-rsa versions that prepend non-PEM metadata to those files.
* **Modern ciphers in `client.conf.tpl`** (`data-ciphers`/`data-ciphers-fallback`/
  `auth SHA256`) instead of the legacy single `cipher AES-128-CBC` directive, for
  OpenVPN 2.6+ servers.
* A generic fallback renderer in the custom `table-row` Vue slot, so that adding a
  custom per-column cell template (like the ones above) doesn't blank out every
  *other* column that isn't explicitly handled — this is a real sharp edge in
  `vue-good-table`'s slot API, not specific to this fork's additions, so it's
  documented here in case it trips up further customization.

None of the above touches the server-side VPN/cert logic beyond what's listed;
everything else (CCD handling, easy-rsa shelling-out, master/slave sync, Prometheus
metrics, etc.) is unchanged from upstream.

### Screenshots

Managing users in ovpn-admin:
![ovpn-admin UI](https://raw.githubusercontent.com/palark/ovpn-admin/master/img/ovpn-admin-users.png)

An example of dashboard made using ovpn-admin metrics:
![ovpn-admin metrics](https://raw.githubusercontent.com/palark/ovpn-admin/master/img/ovpn-admin-metrics.png)

## Installation

### 1. Docker

There is a ready-to-use [docker-compose.yaml](https://github.com/palark/ovpn-admin/blob/master/docker-compose.yaml), so you can just change/add values you need and start it with [start.sh](https://github.com/palark/ovpn-admin/blob/master/start.sh).

Requirements:
You need [Docker](https://docs.docker.com/get-docker/) and [docker-compose](https://docs.docker.com/compose/install/) installed.

Commands to execute:

```bash
git clone https://github.com/palark/ovpn-admin.git
cd ovpn-admin
./start.sh
```
#### 1.1
Ready docker images available on [Docker Hub](https://hub.docker.com/r/flant/ovpn-admin/tags) 
. Tags are simple: `$VERSION` or `latest` for ovpn-admin and `openvpn-$VERSION` or `openvpn-latest` for openvpn-server

### 2. Building from source

Requirements. You need Linux with the following components installed:
- [golang](https://golang.org/doc/install)
- [packr2](https://github.com/gobuffalo/packr#installation)
- [nodejs/npm](https://nodejs.org/en/download/package-manager/)

Commands to execute:

```bash
git clone https://github.com/palark/ovpn-admin.git
cd ovpn-admin
./bootstrap.sh
./build.sh
./ovpn-admin 
```

(Please don't forget to configure all needed params in advance.)

### 3. Prebuilt binary

You can also download and use prebuilt binaries from the [releases](https://github.com/palark/ovpn-admin/releases/latest) page — just choose a relevant tar.gz file.


## Securing the web UI (reverse proxy + Basic Auth)

**ovpn-admin has no login screen of its own.** `--auth.password`/`--auth.db` only
gate a *separate* feature — OpenVPN client username/password auth, checked by
`userCreateHandler`/`userChangePasswordHandler` — they do **not** protect the admin
web UI itself. Per the disclaimer at the top of this README, access control to the
UI is entirely your responsibility; the upstream assumption is a private,
network-restricted environment. If you expose it through any public-facing
address, put it behind a reverse proxy with its own authentication.

### Example: Apache (`httpd`/`httpd2`) reverse proxy with Basic Auth

```apache
<IfModule ssl_module>
  <VirtualHost YOUR_PUBLIC_IP:8080>
      ServerName              your.domain.example

      SSLEngine on
      SSLCertificateFile      /path/to/fullchain.pem
      SSLCertificateKeyFile   /path/to/privkey.pem

      # see "Caching gotcha" below — without this, browsers can end up running
      # a stale cached bundle.min.js against a freshly-deployed index.html
      Header always set Cache-Control "no-cache, must-revalidate"

      <Location />
          AuthType Basic
          AuthName "ovpn-admin"
          AuthUserFile /etc/httpd2/conf/ovpn-admin.htpasswd
          Require valid-user
      </Location>

      ProxyPass        / http://127.0.0.1:8090/
      ProxyPassReverse / http://127.0.0.1:8090/
  </VirtualHost>
</IfModule>
```

Create the password file once (`htpasswd -c` only on the *first* user, drop `-c`
for subsequent ones or you'll overwrite the file):

```bash
htpasswd -cb /etc/httpd2/conf/ovpn-admin.htpasswd someusername 'some-strong-password'
```

Requires `mod_proxy`, `mod_proxy_http`, `mod_ssl` and `mod_headers` enabled, and
bind ovpn-admin itself to localhost only so it's not reachable except through the
proxy: `--listen.host=127.0.0.1`.

### Caching gotcha

Webpack builds `bundle.min.js`/`style.min.js` with **fixed filenames, no
content hash** (see `frontend/webpack.config.js`), and `frontend/static/index.html`
is a static file (not generated by a template plugin that would rewrite those
filenames per build). That means after you deploy a new build, a browser that
already cached the old `bundle.min.js` under that same URL has no signal that the
content changed — it can keep serving the stale JS alongside the new `index.html`.

In practice this showed up as: desktop browsers worked fine (dev tools open with
"Disable cache" is a common habit), while a phone's regular browsing profile kept
loading a blank page after an update, while the *same URL in a private/incognito
tab* (no cache at all) worked correctly — a strong tell that it's a caching
mismatch, not a real bug, if you see this pattern.

Two ways to deal with it:
1. **Server-side (recommended, done once):** send `Cache-Control: no-cache,
   must-revalidate` for the whole proxied vhost, as in the example config above.
   This doesn't stop caching outright — it forces the browser to revalidate with
   the server (via ETag/Last-Modified, a cheap `304 Not Modified` round-trip) before
   reusing a cached copy, so a changed file is always picked up without a manual
   cache clear.
2. **Client-side (one-time, only needed for caches poisoned *before* you added the
   header above):** a full browser cache clear (not just "site settings" — on
   mobile Chrome specifically, Settings → Privacy → Clear browsing data → check
   "Cached images and files", range "All time"). A private/incognito tab also
   works as a quick one-off test since it never reads the existing cache.

If you'd rather fix this at the build level instead: wire `frontend/static/index.html`
through something like `html-webpack-plugin` so hashed output filenames
(`bundle.[contenthash].min.js`) get injected automatically on every build — this
fork didn't do that to keep the diff against upstream's build setup minimal, but
it's the more thorough fix if you're doing frequent deploys.

## Notes
* This tool uses external calls for `bash`, `coreutils` and `easy-rsa`, thus **Linux systems only are supported** at the moment.
* To enable additional password authentication, provide `--auth` and `--auth.db="/etc/easyrsa/pki/users.db`" flags and install [openvpn-user](https://github.com/pashcovich/openvpn-user/releases/latest). This tool should be available in your `$PATH` and its binary should be executable (`+x`).
* If you use `--ccd` and `--ccd.path="/etc/openvpn/ccd"` and plan to use static address setup for users, do not forget to provide `--ovpn.network="172.16.100.0/24"` with valid openvpn-server network.
* If you want to pass all the traffic generated by the user, you need to edit `ovpn-admin/templates/client.conf.tpl` and uncomment `redirect-gateway def1`.
* Tested with openvpn-server versions 2.4 and 2.5 and with tls-auth mode only.
* Not tested with Easy-RSA version > 3.0.8.
* Status of user connections update every 28 seconds.
* Master-replica synchronization and additional password authentication do not work with `--storage.backend=kubernetes.secrets` - **WIP**

## Usage

```
usage: ovpn-admin [<flags>]

Flags:
  --help                       show context-sensitive help (try also --help-long and --help-man)

  --listen.host="0.0.0.0"      host for ovpn-admin
  (or OVPN_LISTEN_HOST)

  --listen.port="8080"         port for ovpn-admin
  (or OVPN_LISTEN_PORT)

  --listen.base-url="/"        base URL for ovpn-admin web files
  (or $OVPN_LISTEN_BASE_URL)

  --role="master"              server role, master or slave
  (or OVPN_ROLE)

  --master.host="http://127.0.0.1"  
  (or OVPN_MASTER_HOST)       URL for the master server

  --master.basic-auth.user=""  user for master server's Basic Auth
  (or OVPN_MASTER_USER)
 
  --master.basic-auth.password=""  
  (or OVPN_MASTER_PASSWORD)   password for master server's Basic Auth

  --master.sync-frequency=600  master host data sync frequency in seconds
  (or OVPN_MASTER_SYNC_FREQUENCY)

  --master.sync-token=TOKEN    master host data sync security token
  (or OVPN_MASTER_TOKEN)

  --ovpn.network="172.16.100.0/24"  
  (or OVPN_NETWORK)           NETWORK/MASK_PREFIX for OpenVPN server

  --ovpn.server=HOST:PORT:PROTOCOL ...  
  (or OVPN_SERVER)            HOST:PORT:PROTOCOL for OpenVPN server
                               can have multiple values

  --ovpn.server.behindLB       enable if your OpenVPN server is behind Kubernetes
  (or OVPN_LB)                Service having the LoadBalancer type

  --ovpn.service="openvpn-external"  
  (or OVPN_LB_SERVICE)        the name of Kubernetes Service having the LoadBalancer
                               type if your OpenVPN server is behind it

  --mgmt=main=127.0.0.1:8989 ...  
  (or OVPN_MGMT)              ALIAS=HOST:PORT for OpenVPN server mgmt interface;
                               can have multiple values

  --metrics.path="/metrics"    URL path for exposing collected metrics
  (or OVPN_METRICS_PATH)

  --easyrsa.path="./easyrsa/"  path to easyrsa dir
  (or EASYRSA_PATH)

  --easyrsa.index-path="./easyrsa/pki/index.txt"  
  (or OVPN_INDEX_PATH)        path to easyrsa index file

  --ccd                        enable client-config-dir
  (or OVPN_CCD)

  --ccd.path="./ccd"           path to client-config-dir
  (or OVPN_CCD_PATH)

  --templates.clientconfig-path=""  
  (or OVPN_TEMPLATES_CC_PATH) path to custom client.conf.tpl

  --templates.ccd-path=""      path to custom ccd.tpl
  (or OVPN_TEMPLATES_CCD_PATH)

  --auth.password              enable additional password authorization
  (or OVPN_AUTH)

  --auth.db="./easyrsa/pki/users.db"
  (or OVPN_AUTH_DB_PATH)      database path for password authorization

  --auth.db-init
  (or OVPN_AUTH_DB_INIT)      enable database init if user db not exists or size is 0
   
  --log.level                  set log level: trace, debug, info, warn, error (default info)
  (or LOG_LEVEL)
  
  --log.format                 set log format: text, json (default text)
  (or LOG_FORMAT)
  
  --storage.backend            storage backend: filesystem, kubernetes.secrets (default filesystem)
  (or STORAGE_BACKEND)
 
  --version                    show application version
```

## Authors

ovpn-admin was originally created in [Flant](https://github.com/flant/) and used internally for years.

In March 2021, it [went public](https://medium.com/flant-com/introducing-ovpn-admin-a-web-interface-to-manage-openvpn-users-d81705ad8f23) and was still developed in Flant.
Namely, [@vitaliy-sn](https://github.com/vitaliy-sn) created its first version in Python, and [@pashcovich](https://github.com/pashcovich) rewrote it in Go.

In November 2024, this project was moved to [Palark](https://github.com/palark/), which is currently responsible for its maintenance and development.

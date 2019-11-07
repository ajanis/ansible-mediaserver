# Ansible-Foreman

**This role is part of the [Ansible-Mediaserver Project][]**

Deploy a full media-server / media-services environment with download agents, library metrics and media request service.

 - Plex Media Server (Docker)
 - SabNZBD Download Agent (Docker)
 - Sonarr TV-Show download agent (Docker)
 - Radarr Movie download agent (Docker)
 - Bazarr Subtitle download agent (Docker)
 - Tautulli - Plex Media Library Stats (Docker)

## Dependencies

[Ansible-Docker][] role is needed to set up media services

The following ansible groups must be defined:
 - mediaservices
    - plexservers
    - nzbservices

## Requirements
### Docker Role variables for media server / service containers
```
docker_containers:
  plex:
    description: "Plex Media Server"
    image: plexinc/pms-docker:plexpass
    network_mode: host
    ports: []
    volumes:
      - '/etc/localtime:/etc/localtime:ro'
      - '{{ media_root }}/configs/plex:/config'
      - '{{ media_root }}/ssl:/ssl'
      - '{{ media_root }}/tv:/tvshows'
      - '{{ media_root }}/movies:/movies'
      - '{{ media_root }}/images:/photos'
      - '/transcode:/transcode'
    env:
      TZ: '{{ timezone }}'
      PLEX_UID: '{{ media_user_uid }}'
      PLEX_GID: '{{ media_user_gid }}'
      ALLOWED_NETWORKS: '10.0.0.0/8,172.16.0.0/16,192.168.0.0/16'
      CHANGE_CONFIG_DIR_OWNERSHIP: 'false'
  tautulli:
    description: "Tautulli - Plex Media Server Usage Metrics"
    image: tautulli/tautulli
    network_mode: bridge
    pull: yes
    ports:
      - 8181:8181
    volumes:
      - '/etc/localtime:/etc/localtime:ro'
      - '{{ media_root }}/configs/tautulli:/config'
      - '{{ media_root }}/configs/plex/Library/Application Support/Plex Media Server/Logs:/plex_logs:ro'
    env:
      TZ: '{{ timezone }}'
      PUID: '{{ media_user_uid }}'
      PGID: '{{ media_user_gid }}'
      ADVANCED_GIT_BRANCH: 'nightly'
  varken:
    description: "Varken Metrics Collector"
    image: boerderij/varken:develop
    network_mode: host
    pull: yes
    volumes:
      - '{{ media_root }}/configs/varken:/config'
    env:
      TZ: '{{ timezone }}'
      PUID: '{{ media_user_uid }}'
      PGID: '{{ media_user_gid }}'

  sabnzbd:
    description: "NZB Download Service"
    image: linuxserver/sabnzbd
    network_mode: bridge
    pull: yes
    ports:
      - 8181:8080
      - 9090:9090
    volumes:
      - '/etc/localtime:/etc/localtime:ro'
      - '{{ media_root }}/configs/sabnzbd:/config'
      - '{{ usenet_incomplete_download_directory}}:/incomplete-downloads'
      - '{{ usenet_complete_download_directory}}:/downloads'
      - '{{ usenet_watch_directory}}:/watch'
    env:
      PUID: '{{ media_user_uid }}'
      PGID: '{{ media_user_gid }}'
  sonarr:
    description: "NZB Search Engine and Library Manager for TV Shows"
    image: linuxserver/sonarr:preview
    network_mode: bridge
    ports:
      - 8180:8989
    pull: yes
    volumes:
      - '/etc/localtime:/etc/localtime:ro'
      - '/dev/rtc:/dev/rtc:ro'
      - '{{ media_root }}/configs/sonarr:/config'
      - '{{ usenet_complete_download_directory}}:/downloads'
      - '{{ media_root }}/tv:/tv'
    env:
      PUID: '{{ media_user_uid }}'
      PGID: '{{ media_user_gid }}'
      TZ: '{{ timezone }}'
  radarr:
    description: "NZB Search Engine and Library Manager for Movies"
    image: linuxserver/radarr
    network_mode: bridge
    pull: yes
    ports:
      - 8183:7878
    volumes:
      - '/etc/localtime:/etc/localtime:ro'
      - '{{ media_root }}/configs/radarr:/config'
      - '{{ usenet_complete_download_directory}}/:/downloads'
      - '{{ media_root }}/movies:/movies'
    env:
      PUID: '{{ media_user_uid }}'
      PGID: '{{ media_user_gid }}'
      TZ: '{{ timezone }}'
  ombi:
    description: "User Requests for Media Server"
    image: linuxserver/ombi
    network_mode: bridge
    pull: yes
    ports:
      - 3579:3579
    volumes:
      - '/etc/localtime:/etc/localtime:ro'
      - '/dev/rtc:/dev/rtc:ro'
      - '{{ media_root }}/configs/ombi:/config'
    env:
      PUID: '{{ media_user_uid }}'
      PGID: '{{ media_user_gid }}'
      TZ: '{{ timezone }}'
  bazarr:
    description: "Subtitle Companion App for Sonarr/Radarr"
    image: linuxserver/bazarr
    network_mode: bridge
    pull: yes
    ports:
      - 6767:6767
    volumes:
      - '{{ media_root }}/configs/bazarr:/config'
      - '{{ media_root }}/tv:/tv'
      - '{{ media_root }}/movies:/movies'
    env:
      PUID: '{{ media_user_uid }}'
      PGID: '{{ media_user_gid }}'
      TZ: '{{ timezone }}'

```

### Additional Deployment Option Requirements

**The following additional services can be configured with the appropriate Ansible roles**


Webserver / Reverse Proxy:

 - NGINX (for reverse proxy) [Ansible-NGINX][]

Stats and Metrics:

 - Varken (For media server / download service performance metrics and stats)

**Requires the following**

  - [Ansible-InfluxDB][]
  - [Ansible-Grafana][]
  - [Ansible-Telegraf][]

Support for shared storage backends:

*Note: Configuring the actual storage backends is beyond the scope of this project*

 - CephFS [Ansible-Ceph-fs][]
 - NFS [Ansible-NFS][]

[Ansible-OpenLDAP][] can be used to set up an OpenLDAP Server, in addition to several client configurations and addon services:
 - Authentication backend for SSH and supported applications such as Grafana.
 - AutoFS (automount) configurations for both shared-storage roles listed above.
 - Apple File Protocol and Samba file sharing services can be configured to share out your media libraries


## Role Variables

### defaults/main.yml
```yaml
# Shared Storage Configs
# currently cephfs and nfs storage backends supported
shared_storage: False
#storage_backend: "cephfs"
#storage_backend: "nfs"

timezone: "US/Mountain"

# Directories
data_mount_root: "/data"
media_directory: "media"
usenet_incomplete_download_directory: "/downloads/usenet/incomplete"
usenet_complete_download_directory: "/downloads/usenet/complete"
usenet_watch_directory: "/downloads/usenet/watch"
media_root: "{{ data_mount_root }}"
media_user_uid:
media_user_gid:

# Plex Server Configs
plex_server_ip:
plex_username:
plex_password:
plex_token:
plex_ramfs_transcode: False
ramfs_transcode_path: /transcode
ramfs_mode: 01777
ramfs_size_mb: 8192

### FILESERVER CONFIGS
afp_ldap_auth: False

```

### vars/debian.yml
```yaml
mediaservice_pkgs:
  - sqlite3
  - unzip
  - mkvtoolnix
  - ffmpeg

```



## Example Group Variables for Supporting Roles
```yaml
# currently cephfs and nfs storage backends supported
shared_storage: True
storage_backend: "cephfs"
use_ldap_automount: True

### STORAGE DIRECTORY CONFIG
data_mount_root: "/data"
media_directory: ""
www_directory: "public_html"
backup_directory: "backups"
ldap_user_home_directory: "homedirs"
git_directory: "git"
configs_directory: "configs"

### FILESERVER CONFIGS
afp_ldap_auth: True

### LDAP SERVER CONFIG
#The ldif file
openldap_server_domain_name: "ldap.home.{{ www_domain }}"
openldap_server_ip: "10.0.10.15"
openldap_server_rootpw: "{{ vault_openldap_server_rootpw }}"
openldap_server_enable_ssl: false

ssl_privkey: "{{ vault_{{ www_domain }}_ssl_private_key }}"
ssl_keypath: "/etc/ssl/private/{{ www_domain }}.key"
ssl_certchain: "{{ vault_{{ www_domain }}_ssl_certificate }}"
ssl_certpath: "/etc/ssl/certs/{{ www_domain }}.crt"

### MEDIA SERVER CONFIGS
media_root: "{{ data_mount_root }}"
media_user_uid: "{{ ssh_users.media.uid }}"
media_user_gid: "{{ ssh_users.media.gid }}"
usenet_incomplete_download_directory: /downloads/usenet/incomplete
usenet_complete_download_directory: /downloads/usenet/complete
usenet_watch_directory: /downloads/usenet/watch
plex_ramfs_transcode: True
ramfs_transcode_path: /transcode
ramfs_size_mb: 8192
plex_server_ip: 10.0.10.175
plex_username: devicenull
plex_password: "{{ vault_plex_password }}"
plex_token: "{{ vault_plex_token }}"


## SSH CONFIGS

sshd_permit_root_login: 'yes'

ssh_users:
  ajanis:
    password: "{{vault_ajanis_pw}}"
    cn: "Alan Janis"
    givenname: "Alan"
    sn: "Janis"
    mail: "alan.janis@gmail.com"
    shell: /bin/zsh
    gecos: ajanis
    uid: 1043
    gid: 1042
    pubkey: "{{ vault_ssh_pubkey }}"
    state: present
  media:
    password: "{{vault_media_pw}}"
    cn: "Media User"
    givenname: "Media"
    sn: "User"
    mail: "alan.janis@gmail.com"
    gecos: media
    uid: 2000
    gid: 2000
    pubkey: "{{ vault_ssh_pubkey }}"
    state: present

ssh_groups:
  admin:
    description: Administrators Group
    gid: 1042
    members:
      - ajanis
      - tglenn
  users:
    description: SSH Users Group
    gid: 2002
    members:
      - ajanis
      - tglenn
      - svela
      - jeasey
      - bkoebler
  media:
    description: Media Services Group
    gid: 2000
    members:
      - ajanis
      - tglenn
      - media

# InfluxDB
influxdb_server_ip: 10.0.10.152
influxdb_install_python_client: True
prometheus_scrape_configs:
  - job_name: "prometheus"
    scrape_interval: 5s
    scrape_timeout: 5s
    metrics_path: "{{ prometheus_metrics_path }}"
    static_configs:
      - targets:
        - "{{ ansible_fqdn | default(ansible_host) | default('localhost') }}:9090"
  - job_name: "node"
    scrape_interval: 5s
    scrape_timeout: 5s
    static_configs:
      - targets:
        - "{{ ansible_fqdn | default(ansible_host) | default('localhost') }}:9100"
  - job_name: ceph_exporter
    scrape_interval: 5s
    scrape_timeout: 5s
    static_configs:
      - targets:
        - 'localhost:9128'
        labels:
          alias: "ceph-exporter"
  - job_name: "unifi_exporter"
    scrape_interval: 5s
    scrape_timeout: 5s
    static_configs:
      - targets:
        - "10.0.10.142:9130"
        labels:
          alias: "unifi-exporter"
  - job_name: maas_stats
    scrape_interval: 5s
    scrape_timeout: 5s
    metrics_path: "/MAAS/metrics"
    static_configs:
      - targets:
        - "10.0.10.118:5240"
  - job_name: maas
    scrape_interval: 5s
    scrape_timeout: 5s
    static_configs:
      - targets:
        - "10.0.10.118:5239"
        - "10.0.10.118:5249"

# Telegraf
telegraf_influxdb_urls:
  - "http://{{ influxdb_server_ip }}:8086"

telegraf_install_url: "https://dl.influxdata.com/telegraf/nightlies/telegraf_nightly_amd64.deb"

telegraf_runas_user: root
telegraf_runas_group: root

telegraf_agent_interval: 5s
telegraf_round_interval: "true"
telegraf_metric_batch_size: "1000"
telegraf_metric_buffer_limit: "10000"
percpu:
telegraf_collection_jitter: 2s
telegraf_flush_interval: 10s
telegraf_flush_jitter: 5s
telegraf_debug: "true"
telegraf_quiet: "false"
telegraf_influxdb_database: telegraf
telegraf_influxdb_precision: ms

# Grafana
grafana_api_token: "{{ vault_grafana_api_token }}"
grafana_url: "https://stats.{{ www_domain }}"
grafana_enable_auth_ldap: True
grafana_enable_auth_anonymous: True

grafana_enable_smtp: True
grafana_smtp_host: "smtp.gmail.com"
grafana_smtp_port: 587
grafana_smtp_user: "{{ vault_gmail_address }}"
grafana_smtp_password: "{{ vault_gmail_password }}"
grafana_smtp_from_address: "{{ vault_gmail_address }}"
grafana_smtp_from_name: "Grafana Alerts"

### WEB SERVER CONFIGS
www_domain: 'prettybaked.com'

nginx_user: "media"
nginx_group: "media"
nginx_default_servername: "home.{{ www_domain }} www.{{ www_domain }} {{ www_domain }}"
nginx_default_docroot: "{{ data_mount_root }}/{{ www_directory }}"

fail2ban_enable: True

fail2ban_bantime: "-1" #Permanent ban
fail2ban_findtime: "10m"
fail2ban_maxretry: "2"
fail2ban_usedns: "raw"
fail2ban_logtarget: /var/log/fail2ban.log
fail2ban_banaction: urltable
fail2ban_pfsense_ip: "10.0.10.1"
fail2ban_pfsense_user: fail2ban
fail2ban_urltable_file: "{{ data_mount_root }}/{{ www_directory }}/fail2ban.txt"
fail2ban_ssh_private_key: "{{ vault_fail2ban_ssh_private_key }}"

fail2ban_services:
  - name: nginx-http-auth
    enabled: True
    filter: nginx-http-auth
    port: http,https
    logpath: /var/log/nginx/home*.log
  - name: nginx-noscript
    enabled: True
    filter: nginx-noscript
    port: http,https
    logpath: /var/log/nginx/home*.log
  - name: nginx-badbots
    enabled: True
    filter: nginx-badbots
    port: http,https
    logpath: /var/log/nginx/home*.log
  - name: nginx-botsearch
    enabled: True
    filter: nginx-botsearch
    port: http,https
    logpath: /var/log/nginx/home*.log
  - name: nginx-nohome
    enabled: True
    filter: nginx-nohome
    port: http,https
    logpath: /var/log/nginx/home*.log
  - name: nginx-404
    enabled: True
    filter: nginx-404
    port: http,https
    logpath: /var/log/nginx/home*.log
  - name: nginx-401
    enabled: True
    filter: nginx-401
    port: http,https
    findtime: '1m'
    bantime: '5m'
    maxretry: 3
    logpath:
      - /var/log/nginx/ombi*.log
      - /var/log/nginx/plex*.log
      - /var/log/nginx/tautulli*.log
      - /var/log/nginx/stats*.log

nginx_proxy_cache_regex: '~* (^/(photo|media|image|images|mediacover|pms_image_proxy)|\.(?:jpg|jpeg|gif|png|ico|cur|gz|svg|svgz|mp4|ogg|ogv|webm|htc|css|js)$)'

nginx_backends:

  - service: automation
    servers:
      - 10.0.11.174:8123

  - service: sabnzbd
    servers:
      - 10.0.10.177:8181

  - service: ombi
    servers:
      - 10.0.10.177:3579

  - service: sonarr
    servers:
      - 10.0.10.177:8180

  - service: radarr
    servers:
      - 10.0.10.177:8183

  - service: grafana
    servers:
      - 10.0.10.152:3000

  - service: tautulli
    servers:
      - 10.0.10.175:8181

  - service: bazarr
    servers:
      - 10.0.10.177:6767

  - service: plex
    servers:
      - 10.0.10.175:32400

nginx_vhosts:
  - servername: "home.{{ www_domain }}"
    serveralias: "{{ www_domain }} www.{{ www_domain }} {{ ansible_eth0.ipv4.address }}"
    serverlisten: "80 default_server"
    locations:
      - name: /
        docroot: "{{ data_mount_root }}/{{ www_directory }}"
        extra_parameters: |
          fancyindex on;
          fancyindex_localtime on; #on for local time zone. off for GMT
          fancyindex_exact_size off; #off for human-readable. on for exact size in bytes
          fancyindex_header "/fancyindex/header.html";
          fancyindex_footer "/fancyindex/footer.html";
          fancyindex_ignore "fancyindex"; #ignore this directory when showing list
          fancyindex_ignore "iot_firewall_allow.txt";
          fancyindex_ignore "fail2ban.txt";
          fancyindex_ignore "robots.txt";

  - servername: "automation.home.{{ www_domain }}"
    serveralias: automation
    serverlisten: 80
    locations:
      - name: /
        proxy: True
        backend: automation
      - name: /api/websocket
        proxy: True
        backend: automation/api/websocket

  - servername: "sabnzbd.home.{{ www_domain }}"
    serveralias: sabnzbd
    serverlisten: 80
    locations:
      - name: /
        proxy: True
        backend: sabnzbd
        custom_css: sabnzbd_dark.css

  - servername: "sonarr.home.{{ www_domain }}"
    serveralias: sonarr
    serverlisten: 80
    locations:
      - name: /
        proxy: True
        proxy_cache: True
        backend: sonarr
        custom_css: sonarr/dark.css
      - name: "{{ nginx_proxy_cache_regex }}"
        proxy: True
        proxy_cache: True
        backend: sonarr


  - servername: "radarr.home.{{ www_domain }}"
    serveralias: radarr
    serverlisten: 80
    locations:
      - name: /
        proxy: True
        backend: radarr
        proxy_cache: True
        custom_css: radarr/dark.css
      - name: "{{ nginx_proxy_cache_regex }}"
        proxy: True
        proxy_cache: True
        backend: radarr

  - servername: "bazarr.home.{{ www_domain }}"
    serveralias: bazarr
    serverlisten: 80
    locations:
      - name: /
        proxy: True
        backend: bazarr
      - name: "{{ nginx_proxy_cache_regex }}"
        proxy: True
        backend: bazarr
        proxy_cache: True

  - servername: "ombi.home.{{ www_domain }}"
    serveralias: ombi
    serverlisten: 80
    locations:
      - name: /
        proxy: True
        backend: ombi
      - name: "{{ nginx_proxy_cache_regex }}"
        proxy: True
        proxy_cache: True
        backend: ombi

  - servername: "tautulli.home.{{ www_domain }}"
    serveralias: tautulli
    serverlisten: 80
    locations:
      - name: /
        proxy: True
        backend: tautulli
        proxy_cache: True
      - name: "{{ nginx_proxy_cache_regex }}"
        proxy: True
        proxy_cache: True
        backend: tautulli

  - servername: "plex.home.{{ www_domain }}"
    serveralias: plex
    serverlisten: 80
    locations:
      - name: /
        proxy: True
        backend: plex
        custom_css: plex/dark.css
      - name: "{{ nginx_proxy_cache_regex }}"
        proxy: True
        proxy_cache: True
        backend: plex

  - servername: "stats.home.{{ www_domain }}"
    serveralias: stats
    serverlisten: 80
    locations:
      - name: /
        proxy: True
        backend: grafana
        custom_css: grafana/graforg.css
      - name: "{{ nginx_proxy_cache_regex }}"
        proxy: True
        backend: grafana
        proxy_cache: True
      - name: favicon.ico
        docroot: "{{ nginx_default_docroot }}"
        proxy: True
        backend: grafana
        proxy_cache: True

  - servername: "home.{{ www_domain }}"
    serveralias: "{{ www_domain }} www.{{ www_domain }} {{ ansible_eth0.ipv4.address }}"
    serverlisten: "8080 default_server"
    locations:
      - name: /
        docroot: "{{ data_mount_root }}/{{ www_directory }}"
        extra_parameters: |
          fancyindex on;
          fancyindex_localtime on; #on for local time zone. off for GMT
          fancyindex_exact_size off; #off for human-readable. on for exact size in bytes
          fancyindex_header "/fancyindex/header.html";
          fancyindex_footer "/fancyindex/footer.html";
          fancyindex_ignore "fancyindex"; #ignore this directory when showing list
          fancyindex_ignore "iot_firewall_allow.txt";
          fancyindex_ignore "fail2ban.txt";
          fancyindex_ignore "robots.txt";

  - servername: "automation.{{ www_domain }}"
    serveralias: automation
    serverlisten: 8080
    locations:
      - name: /
        proxy: True
        backend: automation
      - name: /api/websocket
        proxy: True
        backend: automation/api/websocket

  - servername: "ombi.{{ www_domain }}"
    serveralias: ombi
    serverlisten: 8080
    locations:
      - name: /
        proxy: True
        backend: ombi
      - name: "{{ nginx_proxy_cache_regex }}"
        proxy: True
        proxy_cache: True
        backend: ombi

  - servername: "tautulli.{{ www_domain }}"
    serveralias: tautulli
    serverlisten: 8080
    locations:
      - name: /
        proxy: True
        backend: tautulli
      - name: "{{ nginx_proxy_cache_regex }}"
        proxy: True
        proxy_cache: True
        backend: tautulli

  - servername: "plex.{{ www_domain }}"
    serveralias: plex
    serverlisten: 8080
    locations:
      - name: /
        proxy: True
        proxy_cache: True
        backend: plex
        custom_css: plex/dark.css
      - name: "{{ nginx_proxy_cache_regex }}"
        proxy: True
        proxy_cache: True
        backend: plex

  - servername: "stats.{{ www_domain }}"
    serveralias: stats
    serverlisten: 8080
    locations:
      - name: /
        proxy: True
        backend: grafana
        custom_css: grafana/graforg.css
      - name: "{{ nginx_proxy_cache_regex }}"
        proxy: True
        backend: grafana
        proxy_cache: True
      - name: favicon.ico
        docroot: "{{ nginx_default_docroot }}"
        proxy: True
        backend: grafana
        proxy_cache: True


nginx_vhosts_ssl:
  - servername: "home.{{ www_domain }}"
    serveralias: "{{ www_domain }} www.{{ www_domain }}"
    serverlisten: "443 default_server"
    ssl_certchain: "{{ ssl_certchain }}"
    ssl_privkey: "{{ ssl_privkey }}"
    ssl_certpath: "{{ ssl_certpath }}"
    ssl_keypath: "{{ ssl_keypath }}"
    locations:
      - name: /
        docroot: "{{ data_mount_root }}/{{ www_directory }}"
        extra_parameters: |
          fancyindex on;
          fancyindex_localtime on; #on for local time zone. off for GMT
          fancyindex_exact_size off; #off for human-readable. on for exact size in bytes
          fancyindex_header "/fancyindex/header.html";
          fancyindex_footer "/fancyindex/footer.html";
          fancyindex_ignore "fancyindex"; #ignore this directory when showing list
          fancyindex_ignore "iot_firewall_allow.txt";
          fancyindex_ignore "fail2ban.txt";
          fancyindex_ignore "robots.txt";

  - servername: "automation.home.{{ www_domain }}"
    serverlisten: "443"
    ssl_certchain: "{{ ssl_certchain }}"
    ssl_privkey: "{{ ssl_privkey }}"
    ssl_certpath: "{{ ssl_certpath }}"
    ssl_keypath: "{{ ssl_keypath }}"
    locations:
      - name: /
        proxy: True
        backend: automation
      - name: /api/websocket
        proxy: True
        backend: automation/api/websocket

  - servername: "sabnzbd.home.{{ www_domain }}"
    serverlisten: "443"
    ssl_certchain: "{{ ssl_certchain }}"
    ssl_privkey: "{{ ssl_privkey }}"
    ssl_certpath: "{{ ssl_certpath }}"
    ssl_keypath: "{{ ssl_keypath }}"
    locations:
      - name: /
        proxy: True
        backend: sabnzbd
        custom_css: sabnzbd_dark.css

  - servername: "sonarr.home.{{ www_domain }}"
    serverlisten: "443"
    ssl_certchain: "{{ ssl_certchain }}"
    ssl_privkey: "{{ ssl_privkey }}"
    ssl_certpath: "{{ ssl_certpath }}"
    ssl_keypath: "{{ ssl_keypath }}"
    locations:
      - name: /
        proxy: True
        proxy_cache: True
        backend: sonarr
        custom_css: sonarr/dark.css
      - name: "{{ nginx_proxy_cache_regex }}"
        proxy: True
        proxy_cache: True
        backend: sonarr


  - servername: "radarr.home.{{ www_domain }}"
    serverlisten: "443"
    ssl_certchain: "{{ ssl_certchain }}"
    ssl_privkey: "{{ ssl_privkey }}"
    ssl_certpath: "{{ ssl_certpath }}"
    ssl_keypath: "{{ ssl_keypath }}"
    locations:
      - name: /
        proxy: True
        proxy_cache: True
        backend: radarr
        custom_css: radarr/dark.css
      - name: "{{ nginx_proxy_cache_regex }}"
        proxy: True
        proxy_cache: True
        backend: radarr

  - servername: "bazarr.home.{{ www_domain }}"
    serverlisten: "443"
    ssl_certchain: "{{ ssl_certchain }}"
    ssl_privkey: "{{ ssl_privkey }}"
    ssl_certpath: "{{ ssl_certpath }}"
    ssl_keypath: "{{ ssl_keypath }}"
    locations:
      - name: /
        proxy: True
        backend: bazarr
      - name: "{{ nginx_proxy_cache_regex }}"
        proxy: True
        backend: bazarr
        proxy_cache: True

  - servername: "ombi.home.{{ www_domain }}"
    serverlisten: "443"
    ssl_certchain: "{{ ssl_certchain }}"
    ssl_privkey: "{{ ssl_privkey }}"
    ssl_certpath: "{{ ssl_certpath }}"
    ssl_keypath: "{{ ssl_keypath }}"
    locations:
      - name: /
        proxy: True
        backend: ombi
      - name: "{{ nginx_proxy_cache_regex }}"
        proxy: True
        proxy_cache: True
        backend: ombi

  - servername: "tautulli.home.{{ www_domain }}"
    serverlisten: "443"
    ssl_certchain: "{{ ssl_certchain }}"
    ssl_privkey: "{{ ssl_privkey }}"
    ssl_certpath: "{{ ssl_certpath }}"
    ssl_keypath: "{{ ssl_keypath }}"
    locations:
      - name: /
        proxy: True
        proxy_cache: True
        backend: tautulli
      - name: "{{ nginx_proxy_cache_regex }}"
        proxy: True
        proxy_cache: True
        backend: tautulli

  - servername: "plex.home.{{ www_domain }}"
    serverlisten: "443"
    ssl_certchain: "{{ ssl_certchain }}"
    ssl_privkey: "{{ ssl_privkey }}"
    ssl_certpath: "{{ ssl_certpath }}"
    ssl_keypath: "{{ ssl_keypath }}"
    locations:
      - name: /
        proxy: True
        proxy_cache: True
        backend: plex
        custom_css: plex/dark.css
      - name: "{{ nginx_proxy_cache_regex }}"
        proxy: True
        proxy_cache: True
        backend: plex

  - servername: "stats.home.{{ www_domain }}"
    serverlisten: "443"
    ssl_certchain: "{{ ssl_certchain }}"
    ssl_privkey: "{{ ssl_privkey }}"
    ssl_certpath: "{{ ssl_certpath }}"
    ssl_keypath: "{{ ssl_keypath }}"
    locations:
      - name: /
        proxy: True
        backend: grafana
        custom_css: grafana/graforg.css
      - name: "{{ nginx_proxy_cache_regex }}"
        proxy: True
        backend: grafana
        proxy_cache: True
      - name: favicon.ico
        docroot: "{{ nginx_default_docroot }}"
        proxy: True
        backend: grafana
        proxy_cache: True

  - servername: "home.{{ www_domain }}"
    serveralias: "{{ www_domain }} www.{{ www_domain }} {{ ansible_eth0.ipv4.address }}"
    serverlisten: "8443 default_server"
    ssl_certchain: "{{ ssl_certchain }}"
    ssl_privkey: "{{ ssl_privkey }}"
    ssl_certpath: "{{ ssl_certpath }}"
    ssl_keypath: "{{ ssl_keypath }}"
    locations:
      - name: /
        docroot: "{{ data_mount_root }}/{{ www_directory }}"
        extra_parameters: |
          fancyindex on;
          fancyindex_localtime on; #on for local time zone. off for GMT
          fancyindex_exact_size off; #off for human-readable. on for exact size in bytes
          fancyindex_header "/fancyindex/header.html";
          fancyindex_footer "/fancyindex/footer.html";
          fancyindex_ignore "fancyindex"; #ignore this directory when showing list
          fancyindex_ignore "iot_firewall_allow.txt";
          fancyindex_ignore "fail2ban.txt";
          fancyindex_ignore "robots.txt";

  - servername: "automation.{{ www_domain }}"
    serverlisten: "8443"
    ssl_certchain: "{{ ssl_certchain }}"
    ssl_privkey: "{{ ssl_privkey }}"
    ssl_certpath: "{{ ssl_certpath }}"
    ssl_keypath: "{{ ssl_keypath }}"
    locations:
      - name: /
        proxy: True
        backend: automation
      - name: /api/websocket
        proxy: True
        backend: automation/api/websocket

  - servername: "ombi.{{ www_domain }}"
    serverlisten: "8443"
    ssl_certchain: "{{ ssl_certchain }}"
    ssl_privkey: "{{ ssl_privkey }}"
    ssl_certpath: "{{ ssl_certpath }}"
    ssl_keypath: "{{ ssl_keypath }}"
    locations:
      - name: /
        proxy: True
        backend: ombi
      - name: "{{ nginx_proxy_cache_regex }}"
        proxy: True
        proxy_cache: True
        backend: ombi

  - servername: "tautulli.{{ www_domain }}"
    serverlisten: "8443"
    ssl_certchain: "{{ ssl_certchain }}"
    ssl_privkey: "{{ ssl_privkey }}"
    ssl_certpath: "{{ ssl_certpath }}"
    ssl_keypath: "{{ ssl_keypath }}"
    locations:
      - name: /
        proxy: True
        proxy_cache: True
        backend: tautulli
      - name: "{{ nginx_proxy_cache_regex }}"
        proxy: True
        proxy_cache: True
        backend: tautulli

  - servername: "plex.{{ www_domain }}"
    serverlisten: "8443"
    ssl_certchain: "{{ ssl_certchain }}"
    ssl_privkey: "{{ ssl_privkey }}"
    ssl_certpath: "{{ ssl_certpath }}"
    ssl_keypath: "{{ ssl_keypath }}"
    locations:
      - name: /
        proxy: True
        proxy_cache: True
        backend: plex
        custom_css: plex/dark.css
      - name: "{{ nginx_proxy_cache_regex }}"
        proxy: True
        proxy_cache: True
        backend: plex

  - servername: "stats.{{ www_domain }}"
    serverlisten: "8443"
    ssl_certchain: "{{ ssl_certchain }}"
    ssl_privkey: "{{ ssl_privkey }}"
    ssl_certpath: "{{ ssl_certpath }}"
    ssl_keypath: "{{ ssl_keypath }}"
    locations:
      - name: /
        proxy: True
        backend: grafana
        custom_css: grafana/graforg.css
      - name: "{{ nginx_proxy_cache_regex }}"
        proxy: True
        backend: grafana
        proxy_cache: True
      - name: favicon.ico
        docroot: "{{ nginx_default_docroot }}"
        proxy: True
        backend: grafana
        proxy_cache: True
```

## Example Playbook
```yaml
- name: "[Media Server] :: Deploy Plex Media Server and supporting media services"
  hosts: all
  remote_user: root

  tasks:
    - include_role:
        name: common
    - include_role:
        name: mediaserver
      when: "'mediaservices' in group_names"
    - include_role:
        name: docker
      when: "'mediaservices' in group_names"
    - include_role:
        name: nginx
      when: "'webservices' in group_names"

```

## License

MIT

## Author Information

Created by Alan Janis

[ansible-openldap]: https://github.com/ajanis/ansible-openldap.git
[ansible-telegraf]: https://github.com/ajanis/ansible-telegraf.git
[ansible-nfs]: https://github.com/ajanis/ansible-nfs.git
[ansible-ceph-fs]: https://github.com/ajanis/ansible-cephfs.git
[ansible-nginx]: https://github.com/ajanis/ansible-nginx.git
[ansible-docker]: https://github.com/ajanis/ansible-docker.git
[ansible-grafana]: https://github.com/ajanis/ansible-grafana.git
[ansible-mediaserver project]: https://github.com/ajanis/ansible-mediaserver-project.git
[ansible-influxdb]: https://github.com/ajanis/ansible-influxdb.git

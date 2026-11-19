## Criação da máquina virtual

```bash
eduardo@eduardo:~$ sudo snap install lxd
lxd (6/stable) 6.10.1-f5c750d from Canonical✓ installed
eduardo@eduardo:~$ sudo /snap/bin/lxd init --auto
eduardo@eduardo:~$ lxc launch ubuntu:24.04 pluvial-gs --vm
eduardo@eduardo:~$ lxc exec pluvial-gs -- bash     
```

## Configuração de hostname

O cloud-init define o hostname a partir do nome da VM. Para fixá-lo de forma explícita e impedir que seja sobrescrito:

```bash
root@pluvial-gs:~# hostnamectl set-hostname pluvial-gs
root@pluvial-gs:~# echo "preserve_hostname: true" > /etc/cloud/cloud.cfg.d/99-pluvial-hostname.cfg
root@pluvial-gs:~# hostnamectl --static
pluvial-gs
```

## Configuração de DNS e resolução de nomes

### Resolução local (/etc/hosts)

```bash
root@pluvial-gs:~# sed -i "/^127.0.0.1 localhost/a 127.0.1.1 pluvial-gs.local pluvial-gs\n10.181.228.37 pluvial-gs.local pluvial-gs" /etc/hosts
root@pluvial-gs:~# grep pluvial /etc/hosts
127.0.1.1 pluvial-gs.local pluvial-gs
10.181.228.37 pluvial-gs.local pluvial-gs
```

### Servidores DNS (systemd-resolved)

O DNS principal é o do gateway da rede LXD (`10.181.228.1`), que resolve os nomes internos (domínio `lxd`). Se ele falhar, o sistema usa o Cloudflare (`1.1.1.1`) e o Google (`8.8.8.8`).

```bash
root@pluvial-gs:~# mkdir -p /etc/systemd/resolved.conf.d
root@pluvial-gs:~# cat > /etc/systemd/resolved.conf.d/pluvial.conf <<"EOF"
[Resolve]
DNS=10.181.228.1
FallbackDNS=1.1.1.1 8.8.8.8
Domains=lxd
EOF
root@pluvial-gs:~# systemctl restart systemd-resolved
root@pluvial-gs:~# resolvectl status | grep -E "DNS Servers|Fallback|DNS Domain"
         DNS Servers: 10.181.228.1
Fallback DNS Servers: 1.1.1.1 8.8.8.8
          DNS Domain: lxd
root@pluvial-gs:~# getent hosts github.com
4.228.31.150    github.com
```

## Instalação do Nginx

```bash
root@pluvial-gs:~# sudo apt install -y nginx
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
nginx is already the newest version (1.24.0-2ubuntu7.20).
0 upgraded, 0 newly installed, 0 to remove and 51 not upgraded.
```

## Clone do projeto no servidor

```bash
root@pluvial-gs:~# sudo apt install -y git
sudo mkdir -p /opt/pluvial
sudo git clone https://github.com/vitordpacifico/pluvia-mobile.git /opt/pluvial/pluvia-mobile
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
The following additional packages will be installed:
  git-man
Suggested packages:
  git-daemon-run | git-daemon-sysvinit git-doc git-email git-gui gitk gitweb
  git-cvs git-mediawiki git-svn
The following packages will be upgraded:
  git git-man
2 upgraded, 0 newly installed, 0 to remove and 49 not upgraded.
Need to get 4781 kB of archives.
After this operation, 5120 B of additional disk space will be used.
Get:1 http://archive.ubuntu.com/ubuntu noble-updates/main amd64 git-man all 1:2.43.0-1ubuntu7.4 [1101 kB]
Get:2 http://archive.ubuntu.com/ubuntu noble-updates/main amd64 git amd64 1:2.43.0-1ubuntu7.4 [3681 kB]
Fetched 4781 kB in 1s (5834 kB/s)
(Reading database ... 75120 files and directories currently installed.)
Preparing to unpack .../git-man_1%3a2.43.0-1ubuntu7.4_all.deb ...
Unpacking git-man (1:2.43.0-1ubuntu7.4) over (1:2.43.0-1ubuntu7.3) ...
Preparing to unpack .../git_1%3a2.43.0-1ubuntu7.4_amd64.deb ...
Unpacking git (1:2.43.0-1ubuntu7.4) over (1:2.43.0-1ubuntu7.3) ...
Setting up git-man (1:2.43.0-1ubuntu7.4) ...
Setting up git (1:2.43.0-1ubuntu7.4) ...
Processing triggers for man-db (2.12.0-4build2) ...
Scanning processes...                                                           
Scanning linux images...                                                        

Running kernel seems to be up-to-date.

No services need to be restarted.

No containers need to be restarted.

No user sessions are running outdated binaries.

No VM guests are running outdated hypervisor (qemu) binaries on this host.
Cloning into '/opt/pluvial/pluvia-mobile'...
remote: Enumerating objects: 787, done.
remote: Counting objects: 100% (787/787), done.
remote: Compressing objects: 100% (470/470), done.
remote: Total 787 (delta 311), reused 757 (delta 281), pack-reused 0 (from 0)
Receiving objects: 100% (787/787), 9.62 MiB | 257.00 KiB/s, done.
Resolving deltas: 100% (311/311), done.
```

## Serviço do app (Expo)

```bash
root@pluvial-gs:~# systemctl status pluvial-app
● pluvial-app.service - Pluvia Mobile (Expo)
     Loaded: loaded (/etc/systemd/system/pluvial-app.service; enabled; preset: enabled)
     Active: active (running) since Thu 2026-11-19 16:57:54 UTC; 1min 42s ago
   Main PID: 1171 (node)
      Tasks: 11 (limit: 2234)
     Memory: 468.4M (peak: 472.8M)
        CPU: 4.735s
     CGroup: /system.slice/pluvial-app.service
             └─1171 node /opt/pluvial/pluvia-mobile/node_modules/.bin/expo start --host lan --port 8081

Nov 19 16:57:54 pluvial-gs systemd[1]: Started pluvial-app.service - Pluvia Mobile (Expo).
Nov 19 16:57:54 pluvial-gs expo[1171]: Starting project at /opt/pluvial/pluvia-mobile
Nov 19 16:57:54 pluvial-gs expo[1171]: Using src/app as the root directory for Expo Router.
Nov 19 16:57:54 pluvial-gs expo[1171]: Metro is running in CI mode, reloads are disabled. Remove CI=true to enable watch mode.
Nov 19 16:57:54 pluvial-gs expo[1171]: Starting Metro Bundler
Nov 19 16:57:54 pluvial-gs expo[1171]: Waiting on http://localhost:8081
Nov 19 16:57:54 pluvial-gs expo[1171]: An update for expo is available: 58.0.23 → ~58.0.26
Nov 19 16:57:54 pluvial-gs expo[1171]: 12 other packages may need updating. Run npx expo install --check for details.
Nov 19 16:57:54 pluvial-gs expo[1171]: Logs for your project will appear below.
```

## Monitoramento de CPU, RAM e Disco

### Criação do script monitor.sh

```bash
root@pluvial-gs:~# mkdir -p /opt/pluvial/scripts /var/log/pluvial
root@pluvial-gs:~# cat > /opt/pluvial/scripts/monitor.sh <<"EOF"
#!/usr/bin/env bash
set -uo pipefail     # sem -e: o monitor não deve morrer por uma checagem que falhou

LIMITE_CPU=80; LIMITE_MEM=80; LIMITE_DISCO=85
INTERVALO=30
LOG="/var/log/pluvial/monitor.log"

log() { echo "$(date "+%F %T") [$1] $2" >> "$LOG"; }

cpu_uso()   { vmstat 1 2 | tail -1 | awk "{print 100 - \$15}"; }
mem_uso()   { free | awk "/^Mem:/ {printf \"%d\", (\$2-\$7)/\$2*100}"; }
disco_uso() { df --output=pcent / | tail -1 | tr -dc "0-9"; }

verificar() {
  local nome="$1" valor="$2" limite="$3"
  if (( valor >= limite )); then
    log ALERTA "$nome em ${valor}% (limite ${limite}%)"
  else
    log OK "$nome em ${valor}%"
  fi
}

log INFO "Monitor iniciado (PID $$)"
while true; do
  verificar CPU   "$(cpu_uso)"   "$LIMITE_CPU"
  verificar MEM   "$(mem_uso)"   "$LIMITE_MEM"
  verificar DISCO "$(disco_uso)" "$LIMITE_DISCO"
  sleep "$INTERVALO"
done
EOF
root@pluvial-gs:~# chmod +x /opt/pluvial/scripts/monitor.sh
root@pluvial-gs:~# bash -n /opt/pluvial/scripts/monitor.sh && echo sintaxe-ok
sintaxe-ok
```

### Transformando o monitor em serviço

```bash
root@pluvial-gs:~# cat > /etc/systemd/system/pluvial-monitor.service <<"EOF"
[Unit]
Description=Monitor de recursos pluvial
After=network.target

[Service]
Type=simple
ExecStart=/opt/pluvial/scripts/monitor.sh
Restart=always
RestartSec=5
User=root

[Install]
WantedBy=multi-user.target
EOF
root@pluvial-gs:~# systemctl daemon-reload
root@pluvial-gs:~# systemctl enable --now pluvial-monitor
Created symlink /etc/systemd/system/multi-user.target.wants/pluvial-monitor.service → /etc/systemd/system/pluvial-monitor.service.
root@pluvial-gs:~# systemctl status pluvial-monitor --no-pager
● pluvial-monitor.service - Monitor de recursos pluvial
     Loaded: loaded (/etc/systemd/system/pluvial-monitor.service; enabled; preset: enabled)
     Active: active (running) since Thu 2026-11-19 17:00:58 UTC; 8s ago
   Main PID: 1307 (bash)
      Tasks: 2 (limit: 2234)
```

### Verificação do log

```bash
root@pluvial-gs:~# cat /var/log/pluvial/monitor.log
2026-11-19 17:00:58 [INFO] Monitor iniciado (PID 1307)
2026-11-19 17:00:59 [OK] CPU em 0%
2026-11-19 17:00:59 [OK] MEM em 23%
2026-11-19 17:00:59 [OK] DISCO em 80%
```

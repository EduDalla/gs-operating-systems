# GS Operating Systems (Linux): Pluvia

FIAP · Engenharia de Software · 3º ano · Global Solution 2026.2

## Integrantes

| Nome | RM |
|---|---|
| Eduardo Dallabella | 556803 |
| Guilherme Abe | 554743 |
| Gustavo Ruiz Vieira | 554779 |
| Heloísa Real | 554535 |
| Victos Pacífico Dias | 558017 |

## Projeto

O **Pluvia** é um app em React Native (Expo), tema CitizenTech, que mostra alagamentos perto da pessoa, avisa quando surge uma ocorrência perto do carro estacionado ou de um lugar favorito e recebe relatos com foto da comunidade, conferidos pela equipe antes de aparecerem no mapa. A área atendida é São Paulo, SP (Cambuci, Ipiranga, Mooca, Vila Prudente e bairros vizinhos).

Nesta disciplina, a equipe preparou o servidor Linux que hospeda e executa o app, com monitoramento dos recursos e registro em logs.

- Repositório do app: https://github.com/vitordpacifico/pluvia-mobile
- ODS conectados: 9, 11 e 13

## Implementação

| Componente | Como foi feito | Situação |
|---|---|---|
| Máquina virtual | VM Ubuntu 24.04 criada com LXD | Feito |
| Hostname | `pluvial-gs`, fixado com `hostnamectl` e `preserve_hostname: true` no cloud-init | Feito |
| DNS e nomes | systemd-resolved com DNS do gateway (`10.181.228.1`), fallback `1.1.1.1` e `8.8.8.8`; entradas em `/etc/hosts` | Feito |
| Aplicação | Projeto em `/opt/pluvial/pluvia-mobile`, executado como serviço systemd `pluvial-app` (Expo, porta 8081) | Feito |
| Servidor web | Nginx como proxy reverso para a porta 8081, em HTTP (80) e HTTPS (443, certificado autoassinado) | Feito |
| Monitoramento | `monitor.sh` mede CPU, memória e disco a cada 30 s, com limites de 80%, 80% e 85%, e grava alertas em `/var/log/pluvial/monitor.log` | Feito |
| Serviço de monitoramento | Unit `pluvial-monitor.service`: inicia com o sistema e reinicia sozinha (`Restart=always`) | Feito |
| IP e rede | Configuração de IP estático e parâmetros de rede (netplan) | Pendente |
| Usuário de serviço e permissões | Hoje os serviços rodam como `root` | Pendente |
| Script de provisionamento | Script único e reproduzível que prepare todo o ambiente | Pendente |
| Testes de falha e evidências | Parar o serviço, simular CPU e memória altas, reiniciar a VM e registrar os resultados | Pendente |

## Requisitos

- Host Linux com o [LXD](https://canonical.com/lxd) (instalado via snap) e acesso a `sudo`.
- Acesso à internet (pacotes do Ubuntu, clone do GitHub e dependências do npm).
- Um navegador no host para abrir o app.

## Instalação e reprodução do ambiente

Os passos abaixo reproduzem o ambiente atual. Os comandos com a saída completa estão em [comandos.md](comandos.md). Todos os comandos depois do passo 1 rodam **dentro da VM**, como `root`.

### 1. Criar a VM (no host)

```bash
sudo snap install lxd
sudo /snap/bin/lxd init --auto
lxc launch ubuntu:24.04 pluvial-gs --vm
lxc exec pluvial-gs -- bash
```

### 2. Hostname

```bash
hostnamectl set-hostname pluvial-gs
echo "preserve_hostname: true" > /etc/cloud/cloud.cfg.d/99-pluvial-hostname.cfg
```

### 3. DNS e resolução de nomes

Troque `10.181.228.37` pelo IP da VM (`ip -4 addr show`).

```bash
sed -i "/^127.0.0.1 localhost/a 127.0.1.1 pluvial-gs.local pluvial-gs\n10.181.228.37 pluvial-gs.local pluvial-gs" /etc/hosts

mkdir -p /etc/systemd/resolved.conf.d
cat > /etc/systemd/resolved.conf.d/pluvial.conf <<"EOF"
[Resolve]
DNS=10.181.228.1
FallbackDNS=1.1.1.1 8.8.8.8
Domains=lxd
EOF
systemctl restart systemd-resolved
```

### 4. Pacotes e aplicação

O app foi executado com Node.js 22 e npm 10.

```bash
apt install -y nginx git
mkdir -p /opt/pluvial
git clone https://github.com/vitordpacifico/pluvia-mobile.git /opt/pluvial/pluvia-mobile
cd /opt/pluvial/pluvia-mobile && npm ci
```

### 5. Serviço do app

```bash
cat > /etc/systemd/system/pluvial-app.service <<"EOF"
[Unit]
Description=Pluvia Mobile (Expo)
After=network.target

[Service]
Type=simple
WorkingDirectory=/opt/pluvial/pluvia-mobile
Environment=CI=1
ExecStart=/opt/pluvial/pluvia-mobile/node_modules/.bin/expo start --host lan --port 8081
Restart=always
RestartSec=5
User=root

[Install]
WantedBy=multi-user.target
EOF
systemctl daemon-reload
systemctl enable --now pluvial-app
```

### 6. Nginx com HTTPS

O app usa `crypto.subtle` no cadastro e no login, e o navegador só o libera em contexto seguro (HTTPS ou `localhost`). Por isso o nginx serve o app em HTTPS.

```bash
mkdir -p /etc/nginx/ssl
openssl req -x509 -nodes -newkey rsa:2048 -days 825 \
  -keyout /etc/nginx/ssl/pluvial.key -out /etc/nginx/ssl/pluvial.crt \
  -subj "/CN=pluvial-gs" -addext "subjectAltName=IP:10.181.228.37,DNS:pluvial-gs"
chmod 600 /etc/nginx/ssl/pluvial.key

cat > /etc/nginx/sites-available/pluvia <<"EOF"
map $http_upgrade $connection_upgrade {
    default upgrade;
    ""      close;
}

server {
    listen 80 default_server;
    listen [::]:80 default_server;
    listen 443 ssl default_server;
    listen [::]:443 ssl default_server;
    ssl_certificate /etc/nginx/ssl/pluvial.crt;
    ssl_certificate_key /etc/nginx/ssl/pluvial.key;
    server_name _;

    location / {
        proxy_pass http://127.0.0.1:8081;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;
        proxy_read_timeout 300s;
    }
}
EOF
rm -f /etc/nginx/sites-enabled/default
ln -sf /etc/nginx/sites-available/pluvia /etc/nginx/sites-enabled/pluvia
nginx -t && systemctl reload nginx
```

### 7. Monitoramento

O script `monitor.sh` e a unit `pluvial-monitor.service` estão em [comandos.md](comandos.md), na seção "Monitoramento de CPU, RAM e Disco". Depois de criá-los:

```bash
mkdir -p /opt/pluvial/scripts /var/log/pluvial
chmod +x /opt/pluvial/scripts/monitor.sh
systemctl daemon-reload
systemctl enable --now pluvial-monitor
```

## Execução e verificação

Abra **https://IP-DA-VM/** no navegador (por exemplo `https://10.181.228.37/`) e aceite o aviso do certificado autoassinado. O app traz duas contas de demonstração:

| Perfil | E-mail | Senha |
|---|---|---|
| Cidadão | `cidadao@pluvia.demo` | `Cidadao123` |
| Agente | `agente@pluvia.demo` | `Agente123` |

Para cadastrar uma conta nova, a senha precisa ter pelo menos 8 caracteres, com letras e números.

```bash
# estado dos serviços
systemctl status pluvial-app pluvial-monitor nginx --no-pager

# log do monitoramento (INFO, OK e ALERTA)
tail -f /var/log/pluvial/monitor.log

# log do app
journalctl -u pluvial-app -f

# controle do serviço de monitoramento
systemctl stop pluvial-monitor
systemctl start pluvial-monitor
systemctl restart pluvial-monitor
```

Os limites de alerta ficam no início do `monitor.sh`: CPU 80%, memória 80% e disco 85%, com verificação a cada 30 s.

## Documentação

- [comandos.md](comandos.md): registro dos comandos executados na VM, com as saídas

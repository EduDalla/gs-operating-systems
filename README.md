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
| Máquina virtual | VM Ubuntu 24.04 criada com LXD (`lxc launch ubuntu:24.04 pluvial-gs --vm`) | Feito |
| Hostname | `pluvial-gs`, fixado com `hostnamectl` e `preserve_hostname: true` no cloud-init | Feito |
| DNS e nomes | systemd-resolved com DNS do gateway (`10.181.228.1`), fallback `1.1.1.1` e `8.8.8.8`; entradas em `/etc/hosts` | Feito |
| Aplicação | Projeto clonado em `/opt/pluvial/pluvia-mobile` e executado como serviço systemd `pluvial-app` (Expo, porta 8081) | Feito |
| Servidor web | Nginx como proxy reverso para a porta 8081, em HTTP (80) e HTTPS (443, certificado autoassinado) | Feito |
| Monitoramento | `monitor.sh` mede CPU, memória e disco a cada 30 s, com limites de 80%, 80% e 85%, e grava alertas em `/var/log/pluvial/monitor.log` | Feito |
| Serviço de monitoramento | Unit `pluvial-monitor.service`: inicia com o sistema e reinicia sozinha (`Restart=always`) | Feito |
| IP e rede | Configuração de IP estático e parâmetros de rede (netplan) | Pendente |
| Usuário de serviço e permissões | Hoje os serviços rodam como `root` | Pendente |
| Script de provisionamento | Script único e reproduzível que prepare todo o ambiente | Pendente |
| Testes de falha e evidências | Parar o serviço, simular CPU e memória altas, reiniciar a VM e registrar os resultados | Pendente |

## Documentação

- [comandos.md](comandos.md): registro dos comandos executados na VM

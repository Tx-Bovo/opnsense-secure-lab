# 🛡️ Projeto de Infraestrutura e Segurança (NOC)

![Status](https://img.shields.io/badge/Status-Conclu%C3%ADdo-success?style=for-the-badge&logo=appveyor)
![OPNsense](https://img.shields.io/badge/Firewall-OPNsense-orange?style=for-the-badge)
![Zabbix](https://img.shields.io/badge/Monitoring-Zabbix_7.0_LTS-red?style=for-the-badge&logo=zabbix)
![Docker](https://img.shields.io/badge/Orchestration-Docker_Compose-blue?style=for-the-badge&logo=docker)
![GNS3](https://img.shields.io/badge/Lab-GNS3-lightgrey?style=for-the-badge)

> Um laboratório prático de implementação de segurança de rede, segmentação, VPN com MFA e monitoramento contínuo (NOC).

---

## 📂 Índice

- [1. Descrição e Cenário](#1-descrição-e-cenário)
- [2. Arquitetura e Topologia](#2-arquitetura-e-topologia)
- [3. Ferramentas Utilizadas](#3-ferramentas-utilizadas)
- [4. Implementação e Hardening](#4-implementação-e-hardening)
    - [4.1 Segmentação de Rede (VLANs)](#41-segmentação-de-rede-vlans)
    - [4.2 Configuração do Switch](#42-configuração-do-switch)
    - [4.3 Configuração do Firewall (OPNsense)](#43-configuração-e-regras-de-firewall-hardening)
    - [4.4 Acesso Remoto Seguro (VPN + MFA)](#44-acesso-remoto-seguro-vpn--mfa)
    - [4.5 Monitoramento e Orquestração](#45-monitoramento-e-orquestração-docker)
- [5. Testes e Evidências](#5-testes-e-evidências)
- [6. Desafios e Troubleshooting](#6-desafios-e-troubleshooting)
- [7. Conclusão](#7-conclusão)

---

## 1. Descrição e Cenário

O objetivo deste projeto foi simular a infraestrutura de uma pequena empresa que necessita elevar seu nível de maturidade em segurança. O ambiente precisava sair de uma rede "flat" (sem segregação) para uma rede segmentada, conteinerizada e monitorada.

**Os requisitos do projeto foram:**
1.  Isolar serviços públicos (DMZ) da rede interna (LAN).
2.  Criar uma rede de gerenciamento (MGMT) restrita.
3.  Permitir que administradores acessem a rede remotamente de forma segura.
4.  Monitorar a disponibilidade, integridade e performance de todos os ativos.

---

## 2. Arquitetura e Topologia

O laboratório foi virtualizado inteiramente no **GNS3**, integrando máquinas virtuais e containers Docker.

![image](https://github.com/user-attachments/assets/e31be645-c895-45b3-b770-5fa6eddc3437)

A topologia segue o modelo de defesa em profundidade:
* **Edge:** OPNsense atuando como Firewall e Gateway.
* **Switch Core:** Gerenciamento de VLANs (802.1Q).
* **Endpoints:** Windows 10 (Usuário), Linux Mint (Admin), Docker Hosts (Serviços e Monitoramento).

---

## 3. Ferramentas Utilizadas

| Categoria | Ferramenta | Descrição |
| :--- | :--- | :--- |
| **Firewall** | OPNsense | Distribuição baseada em HardenedBSD com plugin nativo de monitoramento. |
| **Simulador** | GNS3 | Utilizado para emular o hardware de rede e conexões. |
| **Orquestração** | Docker Compose | Gerenciamento de stacks de serviços e agentes via código (IaC). |
| **Monitoramento** | Zabbix 7.0 + Grafana | Coleta de métricas (Active/Passive) e visualização de dados. |
| **Segurança** | OpenVPN + MFA | VPN SSL com autenticação de dois fatores (OTP). |
| **Alvo** | DVWA | *Damn Vulnerable Web App* simulando servidor de produção na DMZ. |

---

## 4. Implementação e Hardening

### 4.1 Segmentação de Rede (VLANs)
Para reduzir a superfície de ataque, a rede foi dividida em zonas lógicas:

| ID | Nome | Interface | Subrede | Função |
| :--- | :--- | :--- | :--- | :--- |
| **10** | `LAN` | `LAN` | `10.10.10.0/24` | Rede de estações de trabalho. Acesso à Internet permitido, acesso à DMZ bloqueado. |
| **20** | `DMZ` | `OPT1` | `10.10.20.0/24` | Rede de serviços expostos (DVWA). Isolada da LAN e MGMT (com exceção de monitoramento). |
| **30** | `MGMT` | `OPT2` | `10.10.30.0/24` | Rede crítica de gerenciamento. Contém o stack Zabbix/Grafana. Acessível apenas via VPN ou Admin autorizado. |


### 4.2 Configuração do Switch (Layer 2)


![Configuração do Switch no GNS3](https://github.com/user-attachments/assets/5830881a-6878-4da3-a51e-660b7f5e6b84)


A segmentação lógica foi materializada na camada de enlace através da configuração de um switch virtual no GNS3. A estratégia de portas foi definida para suportar o tráfego "Tagged" (Tronco) para o firewall e "Untagged" (Acesso) para os dispositivos finais.

**Mapeamento de Portas e VLANs:**

| Porta | Tipo | VLAN ID | Conectado a | Função |
| :--- | :--- | :--- | :--- | :--- |
| **0** | **dot1q (Trunk)** | 1 | **OPNsense (LAN Interface)** | Porta de tronco (Uplink). Trafega todas as VLANs (10, 20, 30) encapsuladas via 802.1Q para que o Firewall faça o roteamento inter-VLAN. |
| **1** | Access | **10** | Windows 10 | Entrega a rede de Usuários (LAN) sem tag. |
| **2** | Access | **20** | Servidor DVWA | Entrega a rede de Serviços (DMZ) sem tag. |
| **3** | Access | **30** | Linux Mint (Admin) | Entrega a rede de Gerência (MGMT) para a estação de trabalho do administrador. |
| **4** | Access | **30** | Servidor Zabbix | Entrega a rede de Gerência (MGMT) para o stack de monitoramento. |

> ⚙️ **Detalhe Técnico:** A porta 0 foi configurada explicitamente como **dot1q**. Isso permite que o OPNsense receba o tráfego marcado e atue como "Router on a Stick", sendo o gateway padrão para todas as sub-redes virtuais.

### 4.3 Configuração e Regras de Firewall (Hardening)

A política de segurança foi desenhada seguindo o princípio do "Least Privilege". Foi aplicada uma lógica rigorosa de **"First Match"** para permitir o monitoramento sem quebrar o isolamento da DMZ:

| Interface | Ação | Origem | Destino | Porta/Protocolo | Propósito |
| :--- | :---: | :--- | :--- | :--- | :--- |
| **DMZ** | ✅ ALLOW | DVWA Host | Zabbix Server | 10051 (TCP) | **Exceção de Monitoramento:** Permite apenas o envio de métricas do Agente (Active) para o Server. |
| **DMZ** | ✅ ALLOW | DMZ Net | This Firewall | 53 (TCP/UDP) | **Infraestrutura:** Garante resolução de DNS antes das regras de bloqueio. |
| **DMZ** | 🚫 BLOCK | DMZ Net | MGMT Net | Any | **Proteção Crítica:** Impede acesso lateral à rede de gerenciamento. |
| **DMZ** | 🚫 BLOCK | DMZ Net | This Firewall | Any | **Gerência Segura:** Bloqueia tentativas de acesso à GUI/SSH do Firewall. |
| **DMZ** | 🚫 BLOCK | DMZ Net | LAN Net | Any | **Anti-Pivoting:** Isola a DMZ da rede de usuários. |
| **DMZ** | ✅ ALLOW | DMZ Net | Any | 80, 443 (TCP) | **Saída Controlada:** Permite apenas tráfego web (updates) via Alias de portas, bloqueando portas altas/suspeitas. |

### 4.4 Acesso Remoto Seguro (VPN + MFA)

Para garantir a administração segura do ambiente fora do perímetro local, foi implementado um servidor **OpenVPN** no OPNsense. A configuração prioriza confidencialidade e integridade, utilizando criptografia forte e autenticação multifator.

**Especificações do Túnel:**
* **Protocolo:** UDP/1194 (Tun Layer 3).
* **Criptografia de Dados:** AES-256-CBC.
* **Algoritmo de Hash (Auth):** SHA512.
* **Autenticação:** Usuário Local + **Token OTP** (Time-based One-Time Password via Google Authenticator).

#### Política de Acesso (Zero Trust)
A VPN foi configurada estritamente como um canal de **Gerenciamento (Management Plane)**. Diferente de VPNs convencionais que dão acesso total à rede, este túnel implementa uma política de bloqueio padrão:

| Origem | Destino | Ação | Justificativa |
| :--- | :--- | :--- | :--- |
| **VPN Clients** | **MGMT Net (VLAN30)** | ✅ ALLOW | Permite acesso ao Zabbix, Dashboards e terminais de administração. |
| **VPN Clients** | **Firewall (Self)** | ✅ ALLOW | Permite acesso restrito a DNS (53), GUI (8443) e SSH (22). |
| **VPN Clients** | **LAN / DMZ** | 🚫 BLOCK | **Anti-Pivoting:** Impede que um administrador comprometido tenha acesso direto a estações de trabalho ou servidores de produção. |

> 🔒 **Estratégia de Segurança:** O tráfego de produção (acesso ao site DVWA) ocorre publicamente via WAN (NAT). A VPN é isolada e exclusiva para a equipe de operações (NOC/SecOps), reduzindo drasticamente a superfície de ataque interna.

![Regras de Firewall da VPN](https://github.com/user-attachments/assets/9f3e8645-b4d2-41ea-bb18-f34e1f205bff)

*(Configuração de regras no OPNsense demonstrando o acesso restrito)*



### 4.5 Monitoramento e Orquestração (Docker)

Todo o ambiente de monitoramento foi implantado utilizando **Docker Compose**, garantindo reprodutibilidade.

**1. Stack de Monitoramento (VLAN MGMT):**
* **Zabbix Server 7.0 LTS:** Backend de coleta com banco de dados MySQL.
* **Self-Monitoring:** Implementado container `zabbix-agent` (Alpine) dentro do stack para monitorar a saúde do próprio servidor.

**2. Monitoramento da DMZ (Active Agent Pattern):**
* O servidor web (DVWA) roda acompanhado de um container **Zabbix Agent 2** no mesmo arquivo `docker-compose.yml`.
* **Configuração Avançada**: Utilizado network_mode: "host" e mapeamento do docker.sock para permitir que o agente monitore o host real e os containers vizinhos.
* **Modo Active:** Devido ao bloqueio de firewall (MGMT não inicia conexões para DMZ), o agente foi configurado como **Active**, iniciando a conexão de fora para dentro na porta 10051.

**3. Monitoramento do Firewall:**
* Instalação do plugin nativo `os-zabbix-agent` (FreeBSD) no OPNsense, reportando métricas de hardware e tráfego diretamente para o servidor.

---

## 5. Testes e Evidências

Aqui estão as comprovações do funcionamento do laboratório.

### 📸 1. Conexão VPN com MFA
*Demonstração do pedido de Token OTP ao conectar na VPN:*

![Print da VPN pedindo token](https://github.com/user-attachments/assets/f55eebfd-5c04-4e7d-a648-fff1a26822ce)

### 📸 2. Regras de Firewall e Hardening
*Configuração de "First Match" garantindo funcionamento do Zabbix e bloqueio de movimentação lateral:*

![Print das regras de firewall](https://github.com/user-attachments/assets/8072bc4f-d8b8-4b2e-86c0-8906e6b2f174)

### 📸 3. Dashboards de Operação (NOC)
***A. Visão de Infraestrutura (OPNsense):** Foco em saúde do hardware (CPU/RAM) e fluxo de tráfego de rede (WAN/LAN/DMZ).*

![Dashboard Grafana](https://github.com/user-attachments/assets/686e2123-d5f3-4147-9c7e-c2b1729f2e2e)

***B. Visão de Serviço (DVWA):** Monitoramento focado na aplicação: Disponibilidade HTTP (Status 200), tempo online e saúde do container Docker.*

![Dashboard Grafana](https://github.com/user-attachments/assets/5a3b27a0-a08a-469b-99b0-199945a4c2ba)

---

## 6. Desafios e Troubleshooting

Durante a implementação do monitoramento na DMZ, um desafio técnico complexo foi encontrado e solucionado.

### 🔧 O Problema: Falha Silenciosa do Agente Ativo
O Zabbix Agent no container DVWA (DMZ) parou de enviar dados para o servidor, embora os testes de conectividade (ping/netcat) na porta 10051 estivessem funcionando e não houvesse erros explícitos de conexão nos logs.

### 🕵️ Diagnóstico
Após analisar os logs do servidor e comparar os ambientes, identificou-se um **Time Drift (Dessincronização de Relógio)** severo de 17 horas entre o container na DMZ e o Zabbix Server.
* **Causa Raiz:** As regras de *Hardening* do Firewall bloqueavam todo o tráfego de saída da DMZ, exceto HTTP/HTTPS. Isso impedia o container de consultar servidores NTP (Network Time Protocol) na porta **UDP 123** para ajustar a hora.
* **Impacto:** O Zabbix Server descartava silenciosamente os dados recebidos do agente, pois os considerava "dados do passado" (timestamp inválido).

### ✅ Solução Implementada
1.  **Firewall:** Criada uma regra de exceção na interface DMZ permitindo tráfego **UDP/123** com destino ao Gateway (OPNsense), que atua como servidor NTP local.
2.  **Docker:** Mapeamento dos volumes `/etc/localtime` e `/etc/timezone` no container para garantir que ele herde a hora correta do host sincronizado.

> **Lição Aprendida:** Em ambientes isolados e conteinerizados, a sincronização de tempo (NTP) é um vetor crítico para a integridade de logs e monitoramento distribuído.

---

## 7. Conclusão

Este projeto permitiu consolidar conhecimentos em **Defesa Cibernética**, **Docker** e **Redes**. O principal desafio foi orquestrar a comunicação entre containers em VLANs isoladas, exigindo configurações finas de Firewall (regras de exceção) e o uso estratégico de Zabbix Agents em modo Ativo vs Passivo. O resultado é um ambiente seguro, segmentado e com observabilidade total.

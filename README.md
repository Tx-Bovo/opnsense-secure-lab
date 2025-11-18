# 🛡️ Projeto de Infraestrutura de Segurança (SOC/NOC)

![Status](https://img.shields.io/badge/Status-Conclu%C3%ADdo-success?style=for-the-badge&logo=appveyor)
![OPNsense](https://img.shields.io/badge/Firewall-OPNsense-orange?style=for-the-badge)
![Zabbix](https://img.shields.io/badge/Monitoring-Zabbix-red?style=for-the-badge&logo=zabbix)
![Docker](https://img.shields.io/badge/Container-Docker-blue?style=for-the-badge&logo=docker)
![GNS3](https://img.shields.io/badge/Lab-GNS3-lightgrey?style=for-the-badge)

> Um laboratório prático de implementação de segurança de rede, segmentação, VPN com MFA e monitoramento contínuo.

---

## 📂 Índice

- [1. Descrição e Cenário](#1-descrição-e-cenário)
- [2. Arquitetura e Topologia](#2-arquitetura-e-topologia)
- [3. Ferramentas Utilizadas](#3-ferramentas-utilizadas)
- [4. Implementação e Hardening](#4-implementação-e-hardening)
    - [4.1 Segmentação de Rede (VLANs)](#41-segmentação-de-rede-vlans)
    - [4.2 Configuração do Firewall (OPNsense)](#42-configuração-do-firewall-opnsense)
    - [4.3 Acesso Remoto Seguro (VPN + MFA)](#43-acesso-remoto-seguro-vpn--mfa)
    - [4.4 Monitoramento (SOC/NOC)](#44-monitoramento-socnoc)
- [5. Testes e Evidências](#5-testes-e-evidências)
- [6. Conclusão](#6-conclusão)

---

## 1. Descrição e Cenário

O objetivo deste projeto foi simular a infraestrutura de uma pequena empresa que necessita elevar seu nível de maturidade em segurança. O ambiente precisava sair de uma rede "flat" (sem segregação) para uma rede segmentada e monitorada.

**Os requisitos do projeto foram:**
1.  Isolar serviços públicos (DMZ) da rede interna (LAN).
2.  Criar uma rede de gerenciamento (MGMT) restrita.
3.  Permitir que administradores acessem a rede remotamente de forma segura.
4.  Monitorar a disponibilidade e integridade dos serviços críticos.

---

## 2. Arquitetura e Topologia

O laboratório foi virtualizado inteiramente no **GNS3**.

![Topologia do Projeto](./topology.png)

A topologia segue o modelo de defesa em profundidade:
* **Edge:** OPNsense atuando como Firewall e Gateway.
* **Switch Core:** Gerenciamento de VLANs (802.1Q).
* **Endpoints:** Windows 10 (Usuário), Linux Mint (Admin), Docker (Serviços).

---

## 3. Ferramentas Utilizadas

| Categoria | Ferramenta | Descrição |
| :--- | :--- | :--- |
| **Firewall** | OPNsense | Distribuição baseada em HardenedBSD para roteamento e firewall. |
| **Simulador** | GNS3 | Utilizado para emular o hardware de rede e conexões. |
| **Container** | Docker | Hospedagem ágil dos serviços de aplicação e monitoramento. |
| **Monitoramento** | Zabbix + Grafana | Coleta de métricas via SNMP e visualização de dados. |
| **Segurança** | OpenVPN + Google Auth | VPN SSL com autenticação de dois fatores (OTP). |
| **Alvo** | DVWA | *Damn Vulnerable Web App* usado para simular um servidor web em produção. |

---

## 4. Implementação e Hardening

### 4.1 Segmentação de Rede (VLANs)
Para reduzir a superfície de ataque, a rede foi dividida em zonas lógicas:

| ID | Nome | Subrede | Função |
| :--- | :--- | :--- | :--- |
| **10** | `LAN` | `192.168.10.0/24` | Rede de estações de trabalho (Windows 10). Acesso à Internet permitido, acesso à DMZ bloqueado. |
| **20** | `DMZ` | `192.168.20.0/24` | Rede de serviços expostos (DVWA). Isolada da LAN e MGMT. |
| **30** | `MGMT` | `192.168.30.0/24` | Rede crítica de gerenciamento. Contém o servidor Zabbix/Grafana. Acessível apenas via VPN ou Console. |

### 4.2 Configuração do Firewall (OPNsense)
As regras de firewall foram aplicadas seguindo o princípio do **menor privilégio**:

* **Regra Default:** Bloqueio total (`Block All`) entre VLANs.
* **Exceção 1:** Permitido tráfego da `MGMT` para `LAN` e `DMZ` (para monitoramento e gestão).
* **Exceção 2:** Bloqueado tráfego da `DMZ` para iniciar conexões com a `LAN` (evita *lateral movement* em caso de comprometimento do DVWA).

### 4.3 Acesso Remoto Seguro (VPN + MFA)
Foi configurado um servidor **OpenVPN** dentro do OPNsense para acesso administrativo.

* **Protocolo:** UDP/1194.
* **Criptografia:** AES-256-CBC.
* **Autenticação:** Usuário Local + Token OTP (Time-based One-Time Password).

> 🔒 **Configuração de Segurança:** A VPN entrega uma rota estática apenas para a subrede `192.168.30.0/24` (MGMT), impedindo que o usuário da VPN acesse a LAN indevidamente.

### 4.4 Monitoramento (SOC/NOC)
O stack de observabilidade foi configurado via Docker na rede de Gerência.

**Zabbix Server:**
* Configurado host OPNsense via **SNMPv3** (mais seguro que v2).
* Configurado host DVWA via Zabbix Agent 2.

**Grafana:**
* Dashboard personalizado consumindo dados do Zabbix para visualização de tráfego de entrada/saída e uso de CPU do Firewall.

---

## 5. Testes e Evidências

Aqui estão as comprovações do funcionamento do laboratório.

### 📸 1. Conexão VPN com MFA
*Demonstração do pedido de Token OTP ao conectar na VPN:*

![Print da VPN pedindo token](./screenshots/vpn-mfa.png)

### 📸 2. Bloqueio de Firewall (LAN vs DMZ)
*Teste de ping falhando da DMZ para a LAN, provando o isolamento:*

![Print do bloqueio de ping](./screenshots/ping-block.png)

### 📸 3. Dashboard de Monitoramento
*Visão geral do Grafana monitorando o tráfego do OPNsense:*

![Dashboard Grafana](./screenshots/grafana-dash.png)

---

## 6. Conclusão

Este projeto permitiu consolidar conhecimentos em **Defesa Cibernética** e **Administração de Redes**. Foi possível demonstrar na prática como a segmentação correta e o uso de múltiplos fatores de autenticação (MFA) aumentam drasticamente a segurança de uma infraestrutura, dificultando a movimentação lateral de atacantes e garantindo visibilidade total através do monitoramento.

---

**Autor:** [Seu Nome]
*Conecte-se comigo no [LinkedIn](seu-link)*

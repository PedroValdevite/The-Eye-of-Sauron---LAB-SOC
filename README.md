# The-Eye-of-Sauron---LAB-SOC

>Este repositório é o registro da tentativa de construção das minhas próprias muralhas de Gondor.

O objetivo deste projeto foi erguer um ambiente corporativo completo do zero, focado em visibilidade total e defesa profunda.

Assim como o Olho que tudo vê em Mordor, implementei um ecossistema onde o Active Directory estabelece a ordem do reino, o Iptables guarda os portões negros e o combo SIEM + IDS + EDR atua como a pupila vigilante.

Aqui, documento no Obsidian cada configuração de telemetria, os desafios de integração e a análise de logs reais, transformando dados brutos em inteligência contra ameaças.


## Arquitetura do Ambiente

O laboratório é executado em um PC Host utilizando a plataforma de virtualização VMware. A rede foi projetada para garantir a segmentação de funções e o controle rigoroso do tráfego.

### Infraestrutura de Rede

Foram configuradas três interfaces virtuais para segmentação:

- Interface NAT: Provê acesso controlado à rede externa.
    
- Interfaces Host-Only (x2): Utilizadas para isolar o tráfego da rede interna (LAN) e o tráfego de gerenciamento/monitoramento.
    

### Ativos e Componentes

A arquitetura inicial está composta pelos seguintes elementos:

- **Gateway e Firewall:** VM Debian utilizando Iptables para gerenciar as três interfaces e aplicar regras de filtragem.
    
- **Controlador de Domínio:** Windows Server 2022 atuando como Active Directory (AD), DNS e DHCP.
    
- **Servidor Web:** Servidor Linux dedicado à hospedagem de serviços e aplicações.
    
- **Estações de Trabalho:** VMs Windows e Linux simulando o ambiente de usuário final (Workstations).
    
- **Camada de Monitoramento (SOC):** Servidores Linux dedicados à execução das seguintes ferramentas:
    
    - Wazuh (SIEM e EDR)
        
    - Suricata (IDS de Rede)
        
    - Grafana (Visualização de dados e métricas)
        

---

## Observações de Projeto

Esta é a configuração da **arquitetura inicial**. O projeto é dinâmico, e a topologia, assim como as ferramentas escolhidas, serão reavaliadas e adaptadas conforme o andamento das implementações e a necessidade de novos controles técnicos.

Após a consolidação da infraestrutura, o ambiente será utilizado para exercícios de hardening, simulação de ataques e testes de vulnerabilidades.

---

## Tecnologias Utilizadas

- Virtualização: VMware
    
- Sistemas Operacionais: Windows Server 2022, Debian, Distribuições Linux diversas
    
- Segurança: Iptables, Wazuh, Suricata
    
- Observabilidade: Grafana
    
- Documentação: Obsidian
    

**Autor:** Pedro Valdevite
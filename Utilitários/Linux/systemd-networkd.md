O **`systemd-networkd`** é um daemon (serviço em segundo plano) nativo do ecossistema systemd, projetado para gerenciar e configurar interfaces de rede de forma leve, previsível e puramente orientada a arquivos de configuração.

Ele se tornou o padrão ouro para ambientes minimalistas, como servidores sem interface gráfica, contêineres e sistemas embarcados, substituindo ferramentas legadas como o _ifupdown_ ou scripts pesados como o _[[NetworkManager]]_.

## 1. O que é e qual sua proposta?

Ao contrário do NetworkManager (que é focado em desktops e reage dinamicamente a mudanças abruptas como "conectar em um Wi-Fi novo na cafeteria"), o `systemd-networkd` assume que o ambiente é **estático ou controlado**. Ele foi feito para inicializar a rede o mais rápido possível durante o boot do sistema, consumindo o mínimo de memória RAM e CPU.

## 2. Como ele interage com o Kernel do Linux?

O `systemd-networkd` não manipula o hardware diretamente. Ele atua como um tradutor de alto nível para as APIs do Kernel do Linux.

- **Netlink (rtnetlink):** É o canal de comunicação. O `systemd-networkd` abre um [[socket]] Netlink com o kernel. Esse protocolo de comunicação bidirecional permite que o serviço envie comandos para o kernel (como "defina o IP X na interface Y") e receba eventos em tempo real (como "o cabo de rede da interface Z foi desconectado").
    
- **Udev:** O `systemd-networkd` trabalha em perfeita sintonia com o `systemd-udevd` (o gerenciador de dispositivos). Assim que o kernel detecta uma nova placa de rede (física ou virtual), o `udev` a nomeia de forma previsível (ex: `enp3s0`), e o `networkd` imediatamente assume o controle para configurá-la de acordo com suas regras.
    

## 3. Logística e Funcionamento Interno

A logística do `systemd-networkd` é baseada em **casamento de padrões (Match)** de forma linear e ordenada por prioridade alfabética/numérica.

Ele varre os diretórios de configuração em busca de arquivos de texto específicos. Os diretórios seguem a hierarquia padrão do systemd:

1. `/etc/systemd/network/` (Configurações locais do administrador — **maior prioridade**)
    
2. `/run/systemd/network/` (Configurações dinâmicas geradas em tempo de execução)
    
3. `/usr/lib/systemd/network/` (Configurações padrão distribuídas pelo sistema operacional)
    

O serviço lê os arquivos em ordem alfabética. Por isso, usa-se prefixos numéricos (ex: `10-vlan.network`, `20-eth0.network`). Quando uma interface de rede surge no sistema, o `networkd` percorre a lista de arquivos de cima para baixo. **O primeiro arquivo que der "Match" com as propriedades daquela interface será o único aplicado a ela.**

## 4. Tipos de Arquivos e Sintaxe

O `systemd-networkd` utiliza três tipos principais de arquivos no formato `.ini` (estruturados em seções como `[Seção]` e pares de `Chave=Valor`).

### A. Arquivos `.network` (Configurações de Links/IPs)

Configuram endereços IP, rotas, DNS e DHCP para interfaces existentes.

**Sintaxe Comum:**

Ini, TOML

```
[Match]
# Define a qual interface este arquivo se aplica. Aceita curingas (*).
Name=enp* 

[Network]
# Ativa o cliente DHCP (pode ser "yes", "no", "ipv4" ou "ipv6")
DHCP=ipv4
# Endereços DNS caso queira fixar
DNS=1.1.1.1

[Address]
# Usado para IPs estáticos (se DHCP estiver desativado)
Address=192.168.1.100/24

[Route]
# Configuração de rotas estáticas e Gateway
Gateway=192.168.1.1
```

### B. Arquivos `.netdev` (Redes Virtuais)

Usados para **criar** interfaces de rede virtuais (como Bridges, VLANs, Bonds, e túneis VPN/WireGuard).

**Sintaxe Comum:**

Ini, TOML

```
[NetDev]
Name=br0
Kind=bridge
```

### C. Arquivos `.link` (Políticas de Baixo Nível)

Configuram parâmetros de hardware da placa de rede antes mesmo dela ser ativada (como mudar o endereço MAC ou alterar a política de nomenclatura).

**Sintaxe Comum:**

Ini, TOML

```
[Match]
MACAddress=00:a1:b2:c3:d4:e5

[Link]
Name=minha-placa-customizada
MTUBytes=9000
```

## 5. Como interagir com o systemd-networkd

A interação direta é dividida em duas frentes: gerenciamento do daemon e inspeção de estado.

### Gerenciando o Serviço (Via `systemctl`)

Bash

```
# Habilitar no boot e iniciar o serviço
sudo systemctl enable --now systemd-networkd

# Aplicar alterações feitas nos arquivos de configuração
sudo systemctl restart systemd-networkd
# ou recarregar as configurações de forma segura (sem derrubar links estáveis):
sudo networkctl reload
```

### Inspecionando a Rede (Via `networkctl`)

O `networkctl` é a ferramenta de linha de comando oficial para interagir e interrogar o `systemd-networkd`.

|**Comando**|**Função**|
|---|---|
|`networkctl`|Lista todas as interfaces de rede, se estão configuradas e se o link físico está ativo (online/configuring/degraded).|
|`networkctl status`|Mostra um resumo geral da rede do sistema, incluindo IPs obtidos, servidores DNS em uso e rotas principais.|
|`networkctl status enp3s0`|Exibe detalhes profundos de uma única interface (Logs, IPs associados, arquivo de configuração exato que ela deu "Match").|
|`networkctl lldp`|Mostra os vizinhos de rede descobertos via protocolo LLDP (útil para descobrir em qual porta do switch físico o servidor está plugado).|


## 1. Arquitetura de Diretórios e Regras de Ouro

O `systemd-networkd` é puramente orientado a arquivos de configuração `.ini`. Para configurações locais do administrador, utilizaremos estritamente o diretório:

📂 `/etc/systemd/network/`

### As Três Regras de Ouro:

1. **Prefixos Numéricos:** Os arquivos são lidos em ordem alfabética (ex: `10-eth0.network` é lido antes de `20-eth1.network`).
    
2. **A Regra do Primeiro "Match":** Quando uma interface surge no sistema, o `networkd` percorre a lista de arquivos de cima para baixo. **O primeiro arquivo que der "Match" com o nome ou MAC da interface será o único aplicado a ela.** Os demais serão ignorados para aquela interface.
    
3. **Extensões Corretas:**
    
    - `.network` $\rightarrow$ Configura IPs, Rotas, DHCP e DNS em interfaces existentes.
        
    - `.netdev` $\rightarrow$ Cria interfaces virtuais (Bridges, VLANs, Bonds).
        

## 2. Consulta e Inspeção (O comando `networkctl`)

Antes de alterar qualquer arquivo, você precisa mapear o estado atual do sistema usando o `networkctl`.

Bash

```
# Listar todas as interfaces, seus estados de link e qual arquivo de configuração foi aplicado
networkctl list

# Mostrar um resumo geral da rede (IPs ativos, DNS em uso e rotas principais)
networkctl status

# Exibir detalhes profundos de uma interface específica (ex: enp3s0)
networkctl status enp3s0
```

## 3. Casos Práticos de Configuração (Arquivos `.network`)

### Caso A: Configuração de IP Dinâmico (Cliente DHCPv4 e DHCPv6)

Ideal para interfaces conectadas à WAN ou redes com atribuição automática.

Crie o arquivo `/etc/systemd/network/10-dhcp.network`:

Ini, TOML

```
[Match]
# Aplica-se a qualquer interface física que comece com "en" (ex: eth0, enp3s0)
Name=en*

[Network]
# Ativa DHCP para IPv4 e IPv6 simultaneamente
DHCP=yes
# Habilita suporte a IPv6 Privacy Extensions (muda o IP IPv6 temporariamente para privacidade)
IPv6PrivacyExtensions=yes
```

### Caso B: Configuração de IP Estático (Manual)

Ideal para servidores, gateways ou redes locais controladas (LAN).

Crie o arquivo `/etc/systemd/network/20-static.network`:

Ini, TOML

```
[Match]
Name=enp3s0

[Network]
# Desativa explicitamente o DHCP para esta interface
DHCP=no

[Address]
# Define o endereço IP e a máscara de rede em notação CIDR
Address=192.168.10.50/24

[Route]
# Define o Gateway padrão (Rota para a Internet)
Gateway=192.168.10.1

[Network]
# Define os servidores DNS (separados por espaço se houver mais de um)
DNS=1.1.1.1 8.8.8.8
```

## 4. Criação de Redes Virtuais (Arquivos `.netdev` + `.network`)

Para criar estruturas complexas de laboratório, como uma **Bridge** (Ponte) para interligar máquinas virtuais ou contêineres à rede física, você precisa de dois arquivos: um para criar a interface virtual e outro para associar as placas físicas a ela.

### Passo 1: Criar a interface Bridge virtual

Crie o arquivo `/etc/systemd/network/25-br0.netdev`:

Ini, TOML

```
[NetDev]
Name=br0
Kind=bridge
```

### Passo 2: Associar uma placa física (ex: eth1) à Bridge

Crie o arquivo `/etc/systemd/network/30-bridge-member.network`:

Ini, TOML

```
[Match]
Name=eth1

[Network]
# Transforma a eth1 em um membro escravo da bridge br0
Bridge=br0
```

### Passo 3: Configurar o IP na própria Bridge

_Nota: A interface física associada perde o IP próprio; quem recebe o IP agora é a interface virtual criada._

Crie o arquivo `/etc/systemd/network/40-br0-interface.network`:

Ini, TOML

```
[Match]
Name=br0

[Network]
Address=10.0.0.1/24
```

## 5. Ciclo de Vida: Aplicando e Testando as Configurações

O `systemd-networkd` permite aplicar alterações sem a necessidade de reiniciar o sistema operacional.

Bash

```
# Habilitar o serviço para iniciar junto com o sistema (se já não estiver)
sudo systemctl enable systemd-networkd

# Recarregar os arquivos de configuração do disco (Modo Seguro)
# Atualiza as regras sem derrubar conexões ativas que não foram modificadas
sudo networkctl reload

# Forçar a reconfiguração de uma interface específica imediatamente
sudo networkctl reconfigure enp3s0

# Reinicialização completa do serviço de rede (Derruba e levanta tudo)
# Use com cautela se estiver conectado via SSH
sudo systemctl restart systemd-networkd
```

### Troubleshooting (Resolução de Problemas)

Se a interface ficar em estado `configuring` ou `degraded` no comando `networkctl list`, inspecione os logs do daemon em tempo real para identificar erros de sintaxe nos arquivos `.network`:

Bash

```
sudo journalctl -u systemd-networkd -f
```
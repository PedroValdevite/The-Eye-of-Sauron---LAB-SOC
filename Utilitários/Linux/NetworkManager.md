O **`NetworkManager`** é um daemon (serviço em segundo plano) projetado para gerenciar e configurar redes de forma dinâmica, automática e centralizada. Ele se tornou o gerenciador de rede padrão da maioria das distribuições Linux voltadas para desktops e notebooks, mas também é amplamente utilizado em servidores modernos.

Sua principal proposta é abstrair a complexidade da configuração de rede, oferecendo uma camada inteligente entre o usuário, os aplicativos e o Kernel Linux.

---

# 1. O que é e qual sua proposta?

Ao contrário do `systemd-networkd`, que assume ambientes previsíveis e estáticos, o `NetworkManager` foi projetado para ambientes que mudam constantemente.

Ele é capaz de:

- Detectar automaticamente novas redes.
- Alternar entre Wi-Fi e Ethernet.
- Gerenciar VPNs.
- Controlar múltiplas interfaces simultaneamente.
- Integrar-se a interfaces gráficas.
- Reagir dinamicamente a eventos de rede.

Por exemplo:

- Você desconecta o cabo Ethernet → ele ativa o Wi-Fi automaticamente.
- Você chega em casa → conecta-se à rede conhecida.
- Você inicia uma VPN → ele ajusta rotas e DNS automaticamente.

Sua filosofia é:

> "A rede deve funcionar sem que o usuário precise pensar nela."

---

# 2. Como ele interage com o Kernel Linux?

Assim como o `systemd-networkd`, o NetworkManager não conversa diretamente com o hardware.

Ele atua como uma camada de orquestração acima das APIs do Kernel.

## Netlink

O principal canal de comunicação com o kernel.

Por meio do protocolo Netlink, o NetworkManager:

- Configura IPs.
- Cria rotas.
- Ativa interfaces.
- Recebe notificações de mudanças de link.

Exemplo:

Quando um cabo é conectado:

```
Kernel → Evento Netlink → NetworkManager
```

O daemon recebe o evento e executa as ações configuradas.

---

## Udev

O NetworkManager utiliza o `udev` para detectar dispositivos.

Fluxo:

```
Nova placa de rede        ↓Kernel detecta        ↓udev nomeia        ↓NetworkManager assume controle
```

Interfaces como:

```
enp0s3ens33wlp2s0
```

são descobertas automaticamente e passam a ser gerenciadas.

---

## D-Bus

Este é o componente que diferencia o NetworkManager do `systemd-networkd`.

O NetworkManager expõe uma API completa via D-Bus.

Isso permite que programas gráficos ou scripts controlem a rede sem editar arquivos diretamente.

Exemplos:

- GNOME Network Settings
- KDE Plasma Network Settings
- VPN Clients
- Applets de Wi-Fi

Todos se comunicam com o NetworkManager através do D-Bus.

---

# 3. Logística e Funcionamento Interno

A unidade lógica do NetworkManager não é a interface.

A unidade lógica é a **Conexão (Connection Profile)**.

---

## Interface Física

Representa o hardware:

```
enp0s3wlp2s0eth0
```

---

## Perfil de Conexão

Representa a configuração.

Exemplos:

```
CasaEmpresaLaboratórioVPN-Corporativa
```

Um mesmo adaptador Wi-Fi pode possuir dezenas de perfis.

Exemplo:

```
wlp2s0 ├─ Casa ├─ Faculdade ├─ Empresa └─ Hotspot
```

Quando uma rede aparece, o NetworkManager procura um perfil compatível e o ativa automaticamente.

---

# 4. Arquitetura de Arquivos

Os perfis são armazenados principalmente em:

```
/etc/NetworkManager/system-connections/
```

Cada perfil é salvo como um arquivo `.nmconnection`.

Exemplo:

```
Casa.nmconnection
```

---

## Hierarquia de Diretórios

### Configurações globais

```
/etc/NetworkManager/
```

---

### Perfis de conexão

```
/etc/NetworkManager/system-connections/
```

---

### Configurações distribuídas pelo sistema

```
/usr/lib/NetworkManager/
```

---

### Estado temporário

```
/run/NetworkManager/
```

---

# 5. Arquivos `.nmconnection`

Os arquivos utilizam formato semelhante ao INI.

Exemplo:

```
[connection]id=LANuuid=12345678-abcdtype=ethernetinterface-name=enp0s3[ipv4]method=manualaddress1=192.168.1.100/24,192.168.1.1dns=1.1.1.1;8.8.8.8;[ipv6]method=ignore
```

---

## Estrutura comum

### `[connection]`

Identificação do perfil.

```
[connection]id=Casatype=wifi
```

---

### `[ipv4]`

Configurações IPv4.

```
[ipv4]method=auto
```

ou

```
[ipv4]method=manual
```

---

### `[ipv6]`

Configurações IPv6.

```
[ipv6]method=auto
```

---

### `[wifi]`

Configurações Wi-Fi.

```
[wifi]ssid=MinhaRede
```

---

### `[wifi-security]`

Segurança Wi-Fi.

```
[wifi-security]key-mgmt=wpa-pskpsk=minhaSenha
```

---

# 6. Ferramentas de Administração

O NetworkManager pode ser administrado de três formas:

---

## A. nmcli (CLI)

Ferramenta oficial de linha de comando.

É o equivalente do `networkctl`.

---

### Estado geral

```
nmcli general status
```

---

### Interfaces

```
nmcli device status
```

---

### Conexões

```
nmcli connection show
```

---

### Detalhes de uma conexão

```
nmcli connection show "Casa"
```

---

### Redes Wi-Fi disponíveis

```
nmcli device wifi list
```

---

### Conectar em uma rede

```
nmcli device wifi connect "MinhaRede"
```

---

## B. nmtui (TUI)

Interface textual baseada em ncurses.

Executar:

```
nmtui
```

Permite:

- Criar conexões
- Editar IPs
- Configurar DNS
- Ativar/desativar interfaces

Sem editar arquivos manualmente.

---

## C. Interfaces Gráficas

Exemplos:

- GNOME Settings
- KDE Plasma
- Cinnamon
- XFCE

Todos utilizam D-Bus para conversar com o NetworkManager.

---

# 7. Casos Práticos

## Caso A: DHCP

Criar conexão Ethernet com DHCP:

```
nmcli connection add \type ethernet \ifname enp0s3 \con-name LAN-DHCP
```

Configurar DHCP:

```
nmcli connection modify LAN-DHCP ipv4.method auto
```

Ativar:

```
nmcli connection up LAN-DHCP
```

---

## Caso B: IP Estático

Criar perfil:

```
nmcli connection add \type ethernet \ifname enp0s3 \con-name LAN-STATIC
```

Configurar:

```
nmcli connection modify LAN-STATIC \ipv4.addresses 192.168.10.50/24 \ipv4.gateway 192.168.10.1 \ipv4.dns "1.1.1.1 8.8.8.8" \ipv4.method manual
```

Ativar:

```
nmcli connection up LAN-STATIC
```

---

# 8. Redes Virtuais

O NetworkManager também gerencia:

- Bridges
- VLANs
- Bonds
- Teaming
- Túnel GRE
- VXLAN
- WireGuard
- VPNs OpenVPN/IPsec

---

## Criando uma Bridge

### Criar a bridge

```
nmcli connection add \type bridge \ifname br0 \con-name br0
```

---

### Adicionar interface física

```
nmcli connection add \type bridge-slave \ifname enp0s3 \master br0
```

---

### Configurar IP

```
nmcli connection modify br0 \ipv4.addresses 10.0.0.1/24 \ipv4.method manual
```

---

# 9. Ciclo de Vida

## Gerenciando o serviço

```
sudo systemctl enable NetworkManager
```

Iniciar:

```
sudo systemctl start NetworkManager
```

Reiniciar:

```
sudo systemctl restart NetworkManager
```

Verificar status:

```
sudo systemctl status NetworkManager
```

---

## Recarregar configurações

```
nmcli connection reload
```

---

## Reaplicar configuração em uma interface

```
nmcli device reapply enp0s3
```

---

# 10. Troubleshooting

## Logs

Visualizar logs em tempo real:

```
sudo journalctl -u NetworkManager -f
```

---

## Verificar dispositivos

```
nmcli device status
```

---

## Verificar conectividade

```
nmcli networking connectivity
```

Resultado típico:

```
fulllimitedportalnone
```

---

# Comparação Filosófica: NetworkManager vs systemd-networkd

|Aspecto|NetworkManager|systemd-networkd|
|---|---|---|
|Filosofia|Rede dinâmica|Rede estática|
|Público-alvo|Desktop, notebook e servidores modernos|Servidores minimalistas|
|Interface gráfica|Sim|Não|
|D-Bus|Sim|Não|
|Consumo de recursos|Maior|Menor|
|Wi-Fi|Excelente|Limitado (precisa de wpa_supplicant/iwd)|
|VPN|Integrado|Externo|
|Facilidade de uso|Alta|Média|
|Automação de mudanças|Muito alta|Baixa|
|Arquitetura|Baseada em perfis de conexão|Baseada em arquivos de configuração|

Uma forma intuitiva de enxergar a diferença é:

- **`systemd-networkd`** trata a rede como uma infraestrutura previsível: "esta interface deve sempre ter esta configuração".
- **`NetworkManager`** trata a rede como um ambiente vivo: "descubra o que está disponível e conecte-se da melhor forma possível".

Por isso, em laboratórios, desktops, notebooks e ambientes corporativos com Wi-Fi, VPN e mobilidade, o NetworkManager costuma ser a escolha natural. Já em servidores minimalistas, appliances e contêineres, o `systemd-networkd` frequentemente oferece uma solução mais simples, leve e determinística.
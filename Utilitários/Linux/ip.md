O comando `ip` é a ferramenta central para configuração de rede no Linux moderno, substituindo os antigos `ifconfig`, `route` e `arp`. Ele faz parte do pacote **iproute2** e está presente em praticamente todas as distribuições atuais.

---

**Estrutura geral**

```
ip [opções] OBJETO AÇÃO
```

- **OBJETO** é o que você quer gerenciar (`link`, `addr`, `route`, `neigh`...)
- **AÇÃO** é o que fazer com ele (`show`, `add`, `del`, `set`...)

---

**Os principais subcomandos**

`ip link` — gerencia interfaces de rede (ativar, desativar, listar, alterar MTU, renomear).

`ip addr` — gerencia endereços IP associados às interfaces. Substitui o `ifconfig eth0 192.168.1.5`.

`ip route` — manipula a tabela de roteamento. Você define por onde o tráfego deve sair, incluindo o gateway padrão.

`ip neigh` — exibe e manipula a tabela ARP (IPv4) e NDP (IPv6), mostrando quais endereços MAC correspondem a quais IPs na rede local.

`ip netns` — gerencia namespaces de rede, muito usado com containers e ambientes isolados.

---

**Dicas práticas**

Os subcomandos aceitam abreviações: `ip a` funciona igual a `ip addr show`, e `ip r` igual a `ip route show`.

Para tornar as configurações permanentes, use ferramentas como `nmcli`, `netplan` ou edite os arquivos de configuração da sua distribuição — as alterações feitas com `ip` são perdidas ao reiniciar.


![[imagens/Pasted image 20260522105343.png]]
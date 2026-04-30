- **Netfilter (O Motor no Kernel):** É a infraestrutura real que vive dentro do kernel do Linux. O Netfilter insere cinco "ganchos" (hooks) na pilha de rede do sistema operacional. Toda vez que um pacote de rede entra, sai ou atravessa a máquina, ele obrigatoriamente passa por esses ganchos. O Netfilter é quem efetivamente segura o pacote e decide o seu destino.

- **Iptables (O Volante no Espaço do Usuário):** É a ferramenta de linha de comando (user-space utility) que você utiliza para conversar com o Netfilter. O kernel não entende comandos em texto puro; o `iptables` traduz as suas regras para que o Netfilter saiba o que fazer com os pacotes interceptados nos ganchos.

# **As 5 Tabelas (O "Por Quê" da manipulação):**
												
 As regras no iptables são organizadas em coleções chamadas Tabelas, cada uma com um propósito específico:

1. **Filter:** É a tabela padrão. Usada exclusivamente para decidir se um pacote deve passar ou ser bloqueado (ações de firewall).
2. **NAT (Network Address Translation):** Usada para traduzir endereços de rede (modificar IP de origem/destino ou portas), permitindo redirecionamentos ou compartilhamento de internet. Apenas o primeiro pacote de uma conexão é avaliado aqui.
3. **Mangle:** Usada para alterações especializadas no cabeçalho do pacote, como modificar o TTL (Time To Live), marcas de TOS (Type of Service) ou definir marcações internas (MARK) no pacote.
4. **Raw:** Uma tabela processada antes de qualquer outra. Seu principal uso é isentar certos pacotes do pesado sistema de rastreamento de conexões (usando o target `NOTRACK`).
5. **Security:** Usada para regras de rede de Controle de Acesso Obrigatório (MAC), aplicando marcas de contexto de segurança como as utilizadas pelo SELinux.

# **As 5 Chains (O "Onde" o pacote está):** As tabelas contêm _Chains_ (Cadeias), que correspondem diretamente aos 5 ganchos do Netfilter. O caminho que um pacote percorre depende do seu destino:

- **PREROUTING:** O pacote acabou de chegar na interface de rede, antes de qualquer decisão de roteamento.
- **INPUT:** Após o roteamento, o sistema percebe que o pacote é destinado a um processo local (nesta mesma máquina).
- **FORWARD:** Após o roteamento, o sistema percebe que o pacote não é para esta máquina e precisa ser encaminhado para outra (a máquina atuando como roteador).
- **OUTPUT:** O pacote foi gerado por um processo local (nesta máquina) e está prestes a sair.
- **POSTROUTING:** O pacote está prestes a sair da interface de rede, após todas as decisões de roteamento terem sido tomadas.

**O Fluxo Resumido:**

- _Tráfego de entrada para o firewall:_ PREROUTING -> INPUT -> Processo Local.
- _Tráfego de saída do firewall:_ Processo Local -> OUTPUT -> POSTROUTING.
- _Tráfego atravessando o firewall:_ PREROUTING -> FORWARD -> POSTROUTING.



# **Sintaxe e Comandos**

A sintaxe básica para visualização é: `iptables -t [tabela] -L`

_Exemplos:_

- Visualizar as chains e regras da tabela padrão (filter): `iptables -L` (Se você omitir o `-t`, o iptables assume a tabela `filter`).
- Visualizar as chains da tabela NAT: `iptables -t nat -L`.
- Visualizar as chains da tabela Mangle: `iptables -t mangle -L`.

**Comandos de Gerenciamento das Chains:**

- `-A` (_Append_): Adiciona a regra ao final da _chain_. Será a última a ser avaliada.
- `-I` (_Insert_): Insere a regra em uma posição numérica específica (o índice começa em 1). Ex: `-I INPUT 1` coloca a regra no topo absoluto da lista.
- `-D` (_Delete_): Remove uma regra, seja especificando a regra inteira novamente ou pelo seu número na lista (ex: `-D INPUT 1`).
- `-L` (_List_): Lista as regras da _chain_. Frequentemente usado com `-n` (numérico, não resolve nomes de DNS) e `-v` (verbose, para ver contadores de pacotes).
- `-F` (_Flush_): Apaga ("limpa") todas as regras da _chain_ ou da tabela inteira, operando como um reset.
- `-P` (_Policy_): Define a política padrão de uma _chain_. É a ação tomada se o pacote não der _match_ em nenhuma regra da lista. Ex: `-P INPUT DROP`.

**Condições de Filtragem (Matches):**

- **Protocolo:** `-p tcp`, `-p udp`, `-p icmp` ou `-p all`.
- **IP de Origem:** `-s` (source). Pode ser um IP (ex: `192.168.1.1`) ou uma rede (ex: `192.168.0.0/24`). O prefixo `!` inverte a condição (ex: `! -s 192.168.1.1` significa "qualquer IP, exceto este").
- **IP de Destino:** `-d` (destination). Mesma lógica da origem.
- **Interface de Entrada:** `-i` (in-interface). Ex: `-i eth0`. Só é válido nas chains INPUT, FORWARD e PREROUTING.
- **Interface de Saída:** `-o` (out-interface). Ex: `-o eth1`. Só é válido nas chains OUTPUT, FORWARD e POSTROUTING.
- **Portas:** `--sport` (source-port) e `--dport` (destination-port). **Atenção:** você obrigatoriamente precisa declarar o protocolo (`-p tcp` ou `-p udp`) antes de usar as portas.

Cenário Prático 

Mentalize o seguinte cenário: Você administra um servidor web na Internet. A política padrão do seu firewall para pacotes que entram (`INPUT`) acaba de ser definida como `DROP`. Ou seja, tudo está bloqueado por padrão.

**Seu Desafio (pense nos comandos exatos):**

1. Escreva a regra para permitir que tráfego TCP destinado à porta 80 (HTTP) consiga entrar através da sua interface conectada à Internet, que é a `eth0`.

iptables -I INPUT 1 -p tcp -i eth0 --dport 80 -j ACCEPT

2. Você identificou que o IP malicioso `203.0.113.50` está atacando seu servidor. Você quer descartar silenciosamente todo o tráfego que vem exclusivamente desse IP, mas **essa regra deve ter prioridade máxima**, ou seja, ser avaliada antes da regra que libera a porta 80.

iptables -I INPUT 1 -s 203.0.113.50 -j DROP

# Hello Word, primeiros testes práticos

Para consolidar os conhecimentos estudados até então, fiz as primeiras interações com os comandos para criar regras que liberam trafego SSH


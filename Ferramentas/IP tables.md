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

Para consolidar os conhecimentos estudados até então, fiz as primeiras interações com os comandos para criar regras que liberam trafego SSH (port 20) e bloqueio todo resto.

Regra de liberação para porta 22:
iptables -t FILTER -I INPUT 1 -s 192.168.114.1 -i ens34 --dport 22 -j ACCEPT

Coloquei também uma regra que aceitaria trafego loopback para comunicação entre serviços/processos:

iptables -I INPUT 1 -i lo -j ACCEPT

Com essa configuração inicial, eu criei a barreira que permite a entrada apenas de trafego SSH (TCP/22). Sendo assim, testes usando outras portas seriam dropados pelo firewall.

![[Pasted image 20260430095717.png]]


# A Rota de Ida e a Rota de Volta

As chains tratam do sentido e do local do pacote naquele exato milissegundo:

- **O pacote de entrada (Requisição):** Chega do cabo de rede, bate no `PREROUTING`, o kernel percebe que é para a máquina local e o envia para a chain **INPUT**. A sua regra (`-j ACCEPT` na porta 80) o libera, e ele é entregue ao servidor web (Apache/Nginx).
- **O pacote de saída (Resposta):** O servidor web processa o pedido e gera um _novo pacote_ para enviar a página web de volta ao cliente. Como este pacote está sendo gerado por um processo local e vai sair da sua máquina, o kernel o envia para a chain **OUTPUT** e, em seguida, para o `POSTROUTING`.

**O grande problema:** Se a política padrão (`-P`) da sua chain `OUTPUT` for `DROP`, a resposta do seu servidor web ficará presa dentro do firewall e nunca chegará ao cliente, mesmo que o `INPUT` tenha permitido a entrada!


### Conntrack (Stateful Firewall)

Antigamente (em firewalls chamados _stateless_), para a comunicação funcionar, você teria que criar regras manuais de `OUTPUT` liberando portas altas aleatórias para que seu servidor pudesse responder. Isso era um pesadelo de segurança.

Hoje, o iptables usa um módulo interno chamado `conntrack` (Connection Tracking). Ele dá uma "memória" ao seu firewall. Toda vez que uma requisição entra ou sai, o Netfilter anota essa conexão em uma tabela na memória. Ele classifica os pacotes em 4 estados principais:

1. **NEW:** É o primeiro pacote de uma nova conexão (ex: o cliente pedindo acesso na porta 80).
2. **ESTABLISHED:** O firewall reconhece que o pacote faz parte de uma conexão que já viu tráfego em ambas as direções. A resposta do seu servidor web se enquadrará exatamente aqui.
3. **RELATED:** O pacote inicia uma nova conexão, mas está diretamente associado a uma conexão já existente (como dados de um FTP ou mensagens de erro ICMP).
4. **INVALID:** O pacote não pertence a nenhuma conexão conhecida e deve ser dropado.


### Sintaxe e Comandos 
Para que o tráfego de saída que foi iniciado no `INPUT` consiga sair sem que você precise criar dezenas de regras de `OUTPUT`, nós usamos a inteligência do estado.

Em vez de liberar tudo na saída, você diz ao iptables: _"Permita a saída de qualquer pacote, desde que ele seja uma resposta a uma comunicação que eu já conheço e autorizei"_.

O comando para fazer isso na sua chain de saída é: `iptables -A OUTPUT -m state --state ESTABLISHED,RELATED -j ACCEPT`

_Explicação do comando:_

- `-m state`: Carrega explicitamente o módulo de extensão de verificação de estados.
- `--state ESTABLISHED,RELATED`: Verifica se o pacote pertence aos estados conhecidos e seguros.
  
  Vamos à logística de como cada estado se manifesta no mundo real e como o firewall reage:

1. Estado: NEW (Novo)

- **Logística (Cenário):** Um visitante na Internet digita o endereço do seu site no navegador. O navegador dele cria o mesmíssimo primeiro pacote de cumprimento (um TCP SYN) e o dispara em direção à porta 80 do seu servidor.
- **Comportamento do Firewall:** O pacote atinge o gancho do firewall. O módulo `conntrack` consulta sua tabela na memória e percebe que não há nenhum registro prévio daquele endereço de origem conversando com o seu servidor. Ele então cria uma nova entrada na tabela e rotula esse pacote com o estado **NEW**. Se houver uma regra permitindo pacotes NEW para a porta 80, ele passa.

2. Estado: ESTABLISHED (Estabelecido)

- **Logística (Cenário):** O seu servidor web (Nginx/Apache) recebe o pacote inicial do visitante, processa e diz: "Olá, estou aqui, vamos conversar". Ele gera um pacote de resposta (TCP SYN/ACK) e o envia de volta ao visitante. A partir daí, ambos começam a trocar dados da página web.
- **Comportamento do Firewall:** Quando esse pacote de resposta (ou qualquer pacote subsequente de ida e volta) atinge o firewall, o `conntrack` olha para o pacote, reconhece que ele pertence a uma conexão que já viu tráfego fluindo em **ambas as direções** e muda o estado dessa comunicação na memória para **ESTABLISHED**. O firewall permite que o tráfego flua livremente de forma bidirecional e rápida.

3. Estado: RELATED (Relacionado)

- **Logística (Cenário):** O estado _RELATED_ é uma conexão nova, mas que nasce "filha" de uma conexão _ESTABLISHED_ existente. Existem dois cenários clássicos aqui:
    - _Protocolos Complexos (FTP):_ Você se conecta a um servidor FTP na porta 21 (conexão de controle). Quando você pede para baixar um arquivo, o protocolo FTP tenta abrir uma _nova_ conexão paralela (como a porta 20 para dados).
    - _Erros de Rede (ICMP):_ Um usuário da sua rede local tenta acessar um site externo, mas o roteador do provedor de internet de destino caiu. Esse roteador envia de volta uma mensagem de erro ICMP ("Network Unreachable").
- **Comportamento do Firewall:** Em firewalls burros, a porta 20 do FTP ou o ICMP de erro seriam bloqueados pois pareceriam tráfego não solicitado. O `conntrack` (muitas vezes usando "helpers", como o módulo específico para FTP) examina as entranhas da conexão e entende que esta nova tentativa de conexão está intimamente **relacionada** à conexão principal que você já autorizou. O firewall marca o pacote como **RELATED** e, se você tiver uma regra aceitando esse estado, o pacote entra perfeitamente.

4. Estado: INVALID (Inválido)

- **Logística (Cenário):** Um invasor na internet gera um pacote deformado propositalmente (com combinações de flags TCP erradas, como SYN e FIN ao mesmo tempo) para tentar confundir o seu servidor. Ou então, um pacote de erro ICMP chega, mas seu firewall percebe que ninguém na sua rede sequer conversou com o IP que gerou esse erro.
- **Comportamento do Firewall:** O pacote não inicia uma conexão válida, não pertence a uma conexão estabelecida e não está relacionado a nada conhecido na tabela do `conntrack`. O firewall o classifica como **INVALID**. A atitude correta e rigorosa de um Mestre é simplesmente descartá-lo no esquecimento (DROP).
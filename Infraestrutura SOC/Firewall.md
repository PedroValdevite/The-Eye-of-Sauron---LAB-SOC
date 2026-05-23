> Aqui irei documentar todo processo de configuração do iptables do laboratório.

 O primeiro passo será o levantamento de dados e construção da arquitetura esperada para o ambiente. 
 
 --- 
# 1. Topologia da rede

A construção da topologia de rede traz mais eficiência e agilidade para o restante do processo de configuração.

![[Drawing 2026-05-06 06.50.42.excalidraw]]


O ambiente, a principio, terá 2 redes isoladas entre si:

- Rede de servidores

- Rede de workstations
  
  Essa topologia permite visualizar a logística dos pacotes em alto nível. A única VM utilizada no desenho foi o Debian com IPTABLES. 
  
  Em um ambiente coorporativo real, a saída da interface ens34 iria para WAN, por vezes por mais de um link.
   
---
# 2. Logística dos pacotes
  
### 1. Comunicação LAN para WAN (Acesso à Internet)

Quando uma workstation ($192.168.0.33/27$) tenta acessar um site na internet:

1. **Gateway:** O pacote é enviado para o gateway padrão da rede, que é a interface `ens37` ($192.168.0.33$;
    
2. **Encaminhamento (Forward):** O IPTABLES recebe o pacote na `ens37` e o direciona para a interface de saída `ens34`.
    
3. **NAT (Masquerade):** Aqui ira ocorrer o **SNAT** (Source NAT). O endereço de origem privado da workstation é substituído pelo endereço da `ens34` ($192.168.114.129$).
    
4. **Saída:** O pacote sai para o PC Host, que realiza um segundo NAT (como indicado no desenho) antes de chegar ao roteador e, finalmente, à WAN.
    
5. **Retorno:** O caminho inverso ocorre, onde o Debian desfaz o NAT e entrega o pacote especificamente para a workstation que o solicitou.
   
   
   
   ### 2. Comunicação entre Redes (Inter-VLAN/Inter-Rede)

Se um administrador na "Rede de Workstations" tentar acessar um banco de dados na "Rede de Servidores":

1. **Roteamento Interno:** O pacote chega na `ens37`. O Debian consulta sua tabela de roteamento e percebe que o destino está na rede conectada à `ens33`.
    
2. **Filtragem (Firewall):** O tráfego passa pela chain `FORWARD` do IPTABLES.
        
3. **Sem NAT:** Diferente da WAN, aqui **não costuma haver NAT**. O servidor vê o IP real da workstation, o que é ideal para auditoria de logs.
   
   
# Configuração da VM

Recursos:

CPU: 1 processador e 2 cores
RAM: 2048MB
I/Os: LSI Logic
Disk type: SCSI
Disk: 8GB

Sistema operacional: Debian 13

> A fim de tornar a VM eficaz em recursos e segurança, será feita a instalação mínima do sistema.

# Configuração de instalação do sistema operacional

> O documento a seguir tem como objetivo ser um passo a passo de instalação. Será registrado apenas as partes mais relevantes do processo.


1. A principio a configuração se iniciou com opções relacionadas a linguagem e identificação do hardware. Após, houve a criação do usuário root e usuário com menos privilégios.
2. A partir do particionamento do disco será feita uma configuração mais robusta.
![[Pasted image 20260507080527.png]]
Foi selecionado o método manual.
3. Foi selecionado o disco (sda);
> Será criada 2 partições, a primeira para o sistema e 1Gb para SWAP, por segurança.
4. Selecionado Espaço livre;
5. Criar nova partição;
6. Como a VM tem 8Gb e tem 6.4Gb disponível, foi alocado 5.4Gb.
7. Sistema de arquivos ext4
8. Tipo primário;
9. Local inicio;
10. Ponto de montagem: /
11. Finalizar
>SWAP
12. Selecionado o espaço livre que sobrou;
13. Criar nova partição;
14. Tipo primário
15. Usar como: swap
16. Finalizar
> Agora vem a parte mais importante para instalação miníma
![[Pasted image 20260507083040.png]]

Foi selecionado apenas servidor SSH (para gerenciamento da VM). Após será instalado os utilitários necessários manualmente.


# Configuração inicial da VM

> Após subir o sistema operacional, é necessário fazer configurações iniciais no ambiente, pois não há utilitários de interação ao kernel necessários para criar as regras de comunicação. 

Conforme a topologia descrita nos começo dessa nota, há 3 interfaces nessa VM.
Para validar se todas estão sendo detectadas pela VM, foi executado o comando nativo `ip`
, mesmo em versão mínimas.
![[Pasted image 20260513104212.png]]
 Comando: `ip a`
 > Para mais informações sobre esse comando, acesse: [[ip]]


Essa será a primeira interação com a configuração de rede do ambiente. O iptables não está instalado ainda, e mesmo que tivesse, ele não é capaz de interagir com as interfaces de rede. Na distribuição debian, e seus derivados, geralmente há 2 gerenciadores de configurações de rede, o `systemd-networkd` e o `NetworkManager`. No nosso ambiente, vamos usar o deamon [[systemd-networkd]].

> Para mais informações do **NetworkManager**, acesse [[NetworkManager]].





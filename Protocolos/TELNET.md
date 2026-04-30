
## 1. Como funciona o Telnet?

O Telnet opera na **Camada de Aplicação** do modelo TCP/IP e utiliza a **porta 23** por padrão. Ele estabelece uma conexão cliente-servidor baseada em texto simples.

- **Comunicação Bidirecional:** O cliente envia comandos de teclado e o servidor devolve as saídas de texto.
    
- **Texto Puro (Clear Text):** Toda a troca de informações ocorre sem qualquer tipo de criptografia.
    
- **Negociação de Opções:** Antes de começar, o cliente e o servidor negociam parâmetros, como o tamanho da janela do terminal e o tipo de codificação.
    

> **O grande problema:** Por ser texto puro, se alguém interceptar os pacotes na rede (ataque de _Man-in-the-Middle_), conseguirá ler usuários e **senhas** com facilidade. Por isso, ele é hoje restrito a testes de conectividade ou equipamentos legados em redes isoladas.

---

## 2. Por qual protocolo foi substituído?

O Telnet foi substituído pelo **SSH (Secure Shell)**.

Lançado em 1995, o SSH resolveu a vulnerabilidade de segurança do Telnet ao introduzir uma camada robusta de proteção. Ele utiliza a **porta 22** por padrão e se tornou o padrão absoluto para gerenciamento remoto de servidores Linux, equipamentos de rede (switches e roteadores) e infraestrutura de nuvem.
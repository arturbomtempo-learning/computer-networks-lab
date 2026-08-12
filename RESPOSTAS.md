# Respostas

## Parte A - TCP

### Questão 1

Acontece um erro de conexão recusada (ConnectionRefusedError). Isso ocorre porque o TCP é orientado a conexão: o cliente precisa completar o handshake inicial com o servidor antes de trocar qualquer dado. Como não há nenhum processo escutando na porta (fazendo bind e listen), o sistema operacional do servidor recusa o SYN de imediato com um pacote RST, e o cliente recebe o erro na hora, sem chegar a enviar mensagem alguma.

### Questão 2

O mecanismo é o uso de números de sequência em cada segmento TCP. Cada byte enviado recebe um número de sequência, e o receptor usa esses números para reordenar os segmentos que chegam fora de ordem e confirmar (ACK) o que já foi recebido, antes de entregar os dados à aplicação na ordem correta.

### Questão 3

O código atual não suporta múltiplos clientes. Tanto no servidor Java quanto no Python, o accept() é chamado uma única vez, fora de qualquer laço, e o programa fica preso tratando só essa conexão dentro do while de leitura. Se um segundo cliente tentasse conectar nesse meio tempo, o pedido ficaria apenas na fila de espera do sistema operacional (explícita no listen(1) do Python), já que o servidor nunca chama accept() de novo; na prática, esse cliente fica travado aguardando ou acaba recebendo conexão recusada quando o servidor encerra.

## Parte B - UDP

### Questão 1

Com o servidor desligado, o cliente enviou a mensagem normalmente e ficou travado esperando uma resposta que nunca chegou, sem nenhum erro imediato. Isso acontece porque o UDP não estabelece conexão: o sendto() apenas coloca o datagrama na rede, sem checar se existe alguém do outro lado escutando. No TCP (Parte A), o simples fato de tentar conectar já teria gerado um erro imediato de conexão recusada, pois o TCP depende de um handshake prévio para confirmar que o servidor está disponível. No UDP essa falha é silenciosa: só percebemos o problema porque a aplicação esperava uma resposta que não veio.

### Questão 2

Um exemplo é o DNS, usado para resolver nomes de domínio em endereços IP: cada consulta é curta e independente, então o custo de estabelecer uma conexão TCP para cada pergunta seria maior que o próprio benefício, e se a resposta se perder o cliente simplesmente refaz a consulta. Outro exemplo é streaming de áudio e vídeo em tempo real (chamadas de voz, videochamadas): nesses casos um pacote perdido ou atrasado é menos prejudicial do que a retransmissão do TCP, que atrasaria todos os pacotes seguintes esperando o pacote perdido chegar, gerando travamentos perceptíveis na chamada.

### Questão 3

Seria possível implementar, mas não de graça como no TCP. O servidor precisaria manter, na aplicação, uma estrutura própria (por exemplo um dicionário ou mapa) associando cada cliente ao par IP e porta observado em cada recvfrom(), já que o UDP não cria nenhum estado de conexão automaticamente. Como não existe um evento equivalente ao fechamento de conexão do TCP para avisar que um cliente saiu, seria necessário implementar também mensagens de entrada e saída definidas pela própria aplicação, além de um mecanismo de timeout para remover clientes que pararam de responder. Isso muda bastante a arquitetura: o controle de quem está conectado, que no TCP vem pronto da camada de transporte, passa a ser responsabilidade inteira do código da aplicação.

## Parte C - Multicast

### Questão 1

No unicast repetido, o servidor monta e envia 3 pacotes idênticos, um para cada endereço de cliente, então o mesmo dado trafega várias vezes pela rede, inclusive em trechos de link compartilhados por mais de um cliente. No multicast, o servidor envia o pacote uma única vez para o endereço de grupo, e são os roteadores da rede que se encarregam de duplicar o pacote apenas nos pontos onde os caminhos até cada cliente se separam. Isso reduz bastante o tráfego total gerado pelo servidor, principalmente quando há muitos destinatários.

### Questão 2

O TTL (time to live) é um contador colocado no cabeçalho do pacote que é decrementado em uma unidade a cada roteador que ele atravessa; quando chega a zero, o pacote é descartado. No multicast ele é usado para limitar o alcance dos avisos na rede: com um TTL baixo (como o valor 2 usado no servidor Python deste laboratório), o pacote consegue passar por poucos roteadores e fica restrito a redes próximas, evitando que os avisos se espalhem por redes distantes sem necessidade, o que economiza banda e evita ruído em outras partes da rede.

### Questão 3

Não recebe. Nos nossos testes, os clientes só receberam os avisos porque já estavam abertos e inscritos no grupo antes do servidor começar a enviar (por isso o roteiro pede para abrir o cliente primeiro). O multicast é construído sobre UDP, então não guarda histórico nem reenvia o que já foi transmitido: cada aviso só chega a quem está com o socket aberto e escutando aquele grupo no exato momento do envio. Se um cliente ficar offline e voltar depois, ele simplesmente perdeu os avisos enviados nesse intervalo, da mesma forma que perderia qualquer datagrama UDP comum.

## Parte D - WebSocket

### Questão 1

Depois do handshake, a conexão TCP que começou como uma requisição HTTP comum deixa de seguir o modelo de requisição e resposta e passa a funcionar como um canal aberto e bidirecional: tanto o cliente quanto o servidor podem enviar mensagens a qualquer momento, sem precisar abrir uma nova conexão nem esperar ser "perguntados". Os dados trafegam em frames do próprio protocolo WebSocket, não mais em requisições HTTP, e a mesma conexão TCP que fez o handshake continua aberta e é reaproveitada durante toda a sessão, como se vê no nosso código: o servidor (MuralServidor em Java, mural_servidor.py em Python) consegue enviar mensagens para o cliente por conta própria, algo que o HTTP tradicional não permite.

### Questão 2

A diferença está em quem descobre e alcança os destinatários. No multicast, o servidor nem sabe quem são os clientes: ele manda o pacote uma vez para o endereço do grupo, e a própria rede (roteadores) se encarrega de entregar cópias a quem estiver inscrito, sem nenhum registro centralizado. No mural WebSocket, é o próprio servidor da aplicação que mantém uma lista explícita de conexões abertas (getConnections() no Java, o conjunto clientes_conectados no Python) e, ao receber uma mensagem, percorre essa lista enviando uma cópia para cada cliente pela sua conexão TCP individual. Ou seja, no multicast a distribuição acontece na camada de rede; no WebSocket, ela é feita manualmente pela aplicação, em cima de várias conexões unicast.

### Questão 3

Porque o TCP "cru" da Parte A só entende uma troca simples de linhas de texto entre um único cliente e o servidor: o código atual nem aceita mais de uma conexão ao mesmo tempo, e não existe nenhum jeito padronizado de o servidor enviar uma mensagem para vários clientes ou de o cliente saber onde uma mensagem termina e outra começa. O WebSocket já resolve esses dois problemas prontos: define frames com início e fim bem marcados, e a biblioteca oferece, de fábrica, a lista de conexões ativas e callbacks como onOpen, onMessage e onClose, o que torna o broadcast para vários alunos simples de implementar. Além disso, por nascer de um handshake HTTP, o WebSocket atravessa com mais facilidade a infraestrutura já existente da web (proxies, firewalls, navegadores), algo que um protocolo TCP proprietário como o da Parte A não tem.

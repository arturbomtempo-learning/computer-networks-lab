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

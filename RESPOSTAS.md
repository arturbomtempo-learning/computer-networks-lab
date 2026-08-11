# Respostas

## Parte A - TCP

### Questão 1

Acontece um erro de conexão recusada (ConnectionRefusedError). Isso ocorre porque o TCP é orientado a conexão: o cliente precisa completar o handshake inicial com o servidor antes de trocar qualquer dado. Como não há nenhum processo escutando na porta (fazendo bind e listen), o sistema operacional do servidor recusa o SYN de imediato com um pacote RST, e o cliente recebe o erro na hora, sem chegar a enviar mensagem alguma.

### Questão 2

O mecanismo é o uso de números de sequência em cada segmento TCP. Cada byte enviado recebe um número de sequência, e o receptor usa esses números para reordenar os segmentos que chegam fora de ordem e confirmar (ACK) o que já foi recebido, antes de entregar os dados à aplicação na ordem correta.

### Questão 3

O código atual não suporta múltiplos clientes. Tanto no servidor Java quanto no Python, o accept() é chamado uma única vez, fora de qualquer laço, e o programa fica preso tratando só essa conexão dentro do while de leitura. Se um segundo cliente tentasse conectar nesse meio tempo, o pedido ficaria apenas na fila de espera do sistema operacional (explícita no listen(1) do Python), já que o servidor nunca chama accept() de novo; na prática, esse cliente fica travado aguardando ou acaba recebendo conexão recusada quando o servidor encerra.

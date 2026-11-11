# ShopNow — Balanceador de Carga utilizando PHP

## 1. Apresentação do projeto

O **ShopNow** é um projeto acadêmico desenvolvido na disciplina de Sistemas Distribuídos, com o objetivo de implementar um mecanismo de balanceamento de carga para uma aplicação fictícia de comércio eletrônico.

A solução utiliza PHP para distribuir as requisições recebidas entre três servidores independentes. Para realizar essa distribuição, foram implementados três algoritmos diferentes, além de um mecanismo de monitoramento responsável por verificar se as instâncias estão disponíveis para atendimento.

O projeto permite estudar, na prática, conceitos importantes de sistemas distribuídos, como concorrência, distribuição de requisições, monitoramento de serviços e tolerância a falhas.

## 2. Ferramentas e tecnologias utilizadas

Para o desenvolvimento e execução da aplicação, foram utilizadas as seguintes tecnologias:

- PHP 8.3;
- Apache;
- Docker e Docker Compose;
- Python 3;
- Apache Benchmark (`ab`).

**Observação:** SQLite não faz parte da implementação deste projeto.

## 3. Organização da arquitetura

A infraestrutura foi dividida em quatro containers Docker, sendo um responsável pelo balanceamento e os outros três responsáveis pelo processamento das requisições.

A distribuição dos serviços está organizada da seguinte maneira:

- **Balanceador:** recebe as requisições dos clientes através da porta 8080 e determina qual servidor deverá atendê-las.
- **Servidor 1:** primeira instância da aplicação, disponível na porta 8081.
- **Servidor 2:** segunda instância, acessível pela porta 8082.
- **Servidor 3:** terceira instância, acessível pela porta 8083.

Antes de encaminhar uma requisição, o balanceador verifica quais servidores estão funcionando e considera somente as instâncias disponíveis.

### 3.1 Simulação de diferentes capacidades

Para representar servidores com desempenhos distintos, foram adicionados tempos de espera artificiais em cada instância.

| Instância | Atraso simulado |
|---|---:|
| Servidor 1 | 30 ms |
| Servidor 2 | 150 ms |
| Servidor 3 | 300 ms |

Essa configuração possibilita comparar o comportamento dos algoritmos quando os servidores apresentam velocidades de processamento diferentes.

## 4. Inicialização do ambiente

Para iniciar toda a infraestrutura, é necessário acessar o diretório principal do projeto e executar o comando:

```bash
docker compose up --build -d
```

Após a inicialização, os serviços poderão ser acessados pelos seguintes endereços:

| Serviço | Endereço |
|---|---|
| Balanceador | http://localhost:8080 |
| Servidor 1 | http://localhost:8081 |
| Servidor 2 | http://localhost:8082 |
| Servidor 3 | http://localhost:8083 |

Para consultar a situação dos containers e confirmar se estão em execução, utilize:

```bash
docker compose ps
```

## 5. Métodos de distribuição de requisições

Foram desenvolvidas três estratégias de balanceamento, cada uma utilizando um critério diferente para escolher o servidor responsável pelo atendimento.

### 5.1 Round Robin

O Round Robin realiza uma distribuição sequencial das requisições entre os servidores que estão disponíveis.

O balanceador mantém um índice de controle que indica qual instância deverá receber a próxima solicitação. Depois de selecionar um servidor, o índice avança para o próximo.

Dessa maneira, a tendência é que todos os servidores recebam quantidades semelhantes de requisições.

**Endereço para execução:**

```text
http://localhost:8080/?algorithm=round_robin
```

### 5.2 Least Connections

Nesse método, a escolha é baseada na quantidade de requisições que cada servidor está processando naquele momento.

O balanceador identifica a instância com o menor número de conexões ativas e encaminha a nova solicitação para ela.

O contador é atualizado antes do encaminhamento e reduzido quando o processamento termina. Quando dois ou mais servidores apresentam a mesma quantidade de conexões, é utilizado um mecanismo de alternância para desempatar a seleção.

**Endereço para execução:**

```text
http://localhost:8080/?algorithm=least_connections
```

### 5.3 Least Response Time

O Least Response Time utiliza o histórico de desempenho das instâncias para definir o destino das requisições.

Nesse algoritmo, o balanceador seleciona o servidor que apresenta o menor tempo médio de resposta registrado.

Para atualizar os valores, é aplicada uma média móvel exponencial, que permite considerar as medições mais recentes sem descartar completamente o histórico anterior.

Quando uma instância ainda não possui registros de desempenho, são encaminhadas requisições iniciais para obter as primeiras medições e alimentar o mecanismo de seleção.

**Endereço para execução:**

```text
http://localhost:8080/?algorithm=least_response_time
```

## 6. Monitoramento e disponibilidade dos servidores

O balanceador conta com um mecanismo de Health Check para acompanhar continuamente a disponibilidade das instâncias.

Essa verificação é realizada por meio de requisições HTTP direcionadas a um endpoint específico de cada servidor. O endpoint informa a condição da instância sem considerar os atrasos artificiais configurados para os testes.

Quando um servidor deixa de responder adequadamente, ele é retirado temporariamente do conjunto de instâncias disponíveis e deixa de receber novas requisições.

Caso todas as instâncias estejam indisponíveis, o balanceador informa a situação ao cliente através do código HTTP 503 (Service Unavailable).

## 7. Experimento de tolerância a falhas

Para avaliar a capacidade de continuidade do sistema diante da falha de uma instância, foi realizado um teste interrompendo o terceiro servidor.

Primeiramente, o container foi desligado com o comando:

```bash
docker compose stop servidor3
```

Em seguida, foi executado o script responsável por realizar as requisições de teste:

```bash
python3 testes/simular_balanceamento.py
```

Durante esse experimento, o balanceador identificou a indisponibilidade do Servidor 3 e continuou encaminhando as solicitações para as outras duas instâncias.

Conforme observado no teste, os servidores restantes continuaram respondendo normalmente, sem ocorrência de erros nas requisições realizadas.

Para disponibilizar novamente a terceira instância, basta executar:

```bash
docker compose start servidor3
```

## 8. Validação por testes automatizados

Além das simulações práticas, foram implementados testes automatizados utilizando PHP puro para verificar o funcionamento dos algoritmos.

A execução da suíte de testes é realizada através do comando:

```bash
make test
```

**Resultado alcançado:**

- 11 testes executados com sucesso;
- Nenhum teste apresentou falha.

Os testes contemplam diferentes situações de funcionamento, incluindo:

- Alternância de servidores no Round Robin;
- Seleção baseada no número de conexões ativas;
- Tratamento de situações de empate;
- Comportamento do Least Response Time sem histórico de medições;
- Atualização das médias de tempo de resposta.

## 9. Avaliação dos algoritmos por simulação

Para analisar as estratégias implementadas, foram desenvolvidos dois scripts em Python. Um realiza solicitações sequenciais e o outro trabalha com múltiplas requisições simultâneas.

### 9.1 Experimento com requisições sequenciais

O primeiro teste está disponível no arquivo:

```text
testes/simular_balanceamento.py
```

Para executá-lo:

```bash
python3 testes/simular_balanceamento.py
```

Nesse cenário, cada algoritmo recebeu um conjunto de 12 requisições realizadas sequencialmente.

**Distribuição observada:**

| Algoritmo | Servidor 1 | Servidor 2 | Servidor 3 |
|---|---:|---:|---:|
| Round Robin | 4 | 4 | 4 |
| Least Connections | 4 | 4 | 4 |
| Least Response Time | 10 | 1 | 1 |

**Interpretação dos resultados:**

- **Round Robin:** apresentou uma divisão igualitária das solicitações entre as três instâncias, conforme sua lógica de funcionamento.
- **Least Connections:** também distribuiu as requisições igualmente. Isso ocorreu porque, sendo o processamento sequencial, cada solicitação terminava antes do início da próxima, deixando todos os contadores de conexões zerados no momento da seleção.
- **Least Response Time:** concentrou a maior parte das solicitações no Servidor 1, pois ele apresentou o menor tempo de resposta entre as instâncias.

### 9.2 Experimento com requisições concorrentes

O segundo cenário foi criado para avaliar o comportamento dos algoritmos quando existem várias solicitações sendo processadas ao mesmo tempo.

O código está localizado em:

```text
testes/simular_concorrencia.py
```

Para iniciar o experimento:

```bash
python3 testes/simular_concorrencia.py
```

Foram disparadas 18 requisições, utilizando até 6 trabalhadores simultâneos.

**Resultados registrados:**

| Algoritmo | Servidor 1 | Servidor 2 | Servidor 3 |
|---|---:|---:|---:|
| Round Robin | 6 | 6 | 6 |
| Least Connections | 12 | 4 | 2 |
| Least Response Time | 9 | 3 | 6 |

**Análise do comportamento:**

- **Round Robin:** manteve a distribuição uniforme, independentemente das diferenças de velocidade entre os servidores.
- **Least Connections:** direcionou uma quantidade maior de solicitações ao Servidor 1, que possui menor tempo de processamento e, consequentemente, libera suas conexões mais rapidamente.
- **Least Response Time:** distribuiu as requisições considerando os tempos médios de resposta armazenados pelo balanceador.

Ao final da simulação, todas as solicitações receberam resposta HTTP 200, sem registro de erros.

## 10. Avaliação de desempenho com Apache Benchmark

Para complementar os experimentos, também foram realizados testes de carga utilizando o Apache Benchmark (`ab`).

A ferramenta foi executada dentro de um container Alpine, permitindo gerar um volume maior de requisições e avaliar o desempenho dos algoritmos sob concorrência.

### 10.1 Configurações do teste

Os parâmetros utilizados foram:

- Quantidade total: 1.000 requisições;
- Concorrência: 50 solicitações simultâneas;
- Infraestrutura: três servidores;
- Ferramenta de medição: Apache Benchmark.

### 10.2 Execução do Round Robin

```bash
docker run --rm --network shopnow-load-balancer_default alpine:3.20 sh -c "apk add --no-cache apache2-utils >/dev/null && ab -n 1000 -c 50 'http://balanceador/?algorithm=round_robin'"
```

### 10.3 Execução do Least Connections

```bash
docker run --rm --network shopnow-load-balancer_default alpine:3.20 sh -c "apk add --no-cache apache2-utils >/dev/null && ab -n 1000 -c 50 'http://balanceador/?algorithm=least_connections'"
```

### 10.4 Comparação dos resultados

Os valores obtidos durante os experimentos foram:

| Indicador | Round Robin | Least Connections |
|---|---:|---:|
| Requisições concluídas | 1.000 | 1.000 |
| Falhas reais (conexão, recebimento, exceções ou respostas não 2xx) | 0 | 0 |
| Failed requests reportadas pelo ab (Length) | 666 | 301 |
| Duração total | 7,691 s | 6,356 s |
| Requisições por segundo | 130,02 | 157,33 |
| Tempo médio por requisição | 384,550 ms | 317,800 ms |
| Mediana | 350 ms | 266 ms |
| Percentil 95 | 790 ms | 621 ms |
| Maior tempo registrado | 1.032 ms | 921 ms |

### 10.5 Entendendo as falhas reportadas pelo Apache Benchmark

Durante os testes, o Apache Benchmark apresentou uma quantidade de `Failed requests` diferente de zero. Entretanto, essas ocorrências não representaram falhas reais de comunicação ou processamento.

O motivo está relacionado à forma como a ferramenta verifica o tamanho das respostas HTTP. Ela utiliza o tamanho da primeira resposta recebida como referência, que nesse experimento foi de 161 bytes.

Como cada servidor retorna um valor diferente no campo `artificial_delay_ms`, existe uma diferença no tamanho do conteúdo:

- Servidor 1: `30`;
- Servidor 2: `150`;
- Servidor 3: `300`.

Assim, as respostas dos Servidores 2 e 3 apresentam um caractere adicional em relação à resposta do primeiro servidor.

Por conta dessa diferença, o Apache Benchmark classifica algumas respostas como falhas do tipo `Length`, mesmo quando o código HTTP retornado é 200.

Essas contagens também permitem observar a distribuição das requisições:

- **Round Robin:** 666 solicitações foram direcionadas aos Servidores 2 e 3, correspondendo a aproximadamente dois terços do total.
- **Least Connections:** somente 301 solicitações foram encaminhadas aos Servidores 2 e 3, indicando uma preferência maior pelo Servidor 1.

### 10.6 Discussão dos resultados de desempenho

Considerando os resultados obtidos nessa execução, o Least Connections apresentou desempenho superior ao Round Robin em todas as métricas de tempo e vazão analisadas.

A taxa de processamento passou de 130,02 para 157,33 requisições por segundo, representando um aumento aproximado de 21%.

Outro resultado importante foi a redução da mediana de resposta, que passou de 350 ms para 266 ms, uma diferença de 84 ms.

Esse comportamento pode ser explicado pela diferença de capacidade entre os servidores. Enquanto o Round Robin mantém uma distribuição uniforme e encaminha aproximadamente um terço das solicitações para a instância mais lenta, o Least Connections considera a quantidade de requisições em andamento em cada servidor.

Com isso, o Servidor 1, que apresenta menor atraso artificial, consegue finalizar suas solicitações mais rapidamente e receber novas requisições com maior frequência.

É importante destacar que os resultados podem variar de acordo com as condições de execução, a máquina utilizada e a carga do ambiente. Em execuções anteriores, por exemplo, os dois algoritmos apresentaram taxas próximas de 93 a 94 requisições por segundo.

Por esse motivo, a comparação realizada sob as mesmas condições de execução é a referência mais adequada para avaliar o comportamento observado.

## 11. Aspectos técnicos e limitações

Para permitir que diferentes processos Apache compartilhem informações durante a execução, o projeto utiliza um arquivo temporário como mecanismo de armazenamento de estado.

O acesso a esse arquivo é controlado através da função `flock`, que permite coordenar operações concorrentes e reduzir conflitos durante a atualização dos dados.

A implementação possui caráter acadêmico e foi desenvolvida para demonstrar conceitos fundamentais de Sistemas Distribuídos, principalmente:

- Distribuição de carga entre múltiplas instâncias;
- Gerenciamento de requisições concorrentes;
- Monitoramento de disponibilidade;
- Identificação e tratamento de falhas;
- Comparação de algoritmos de balanceamento;
- Avaliação de desempenho por testes de carga.

Apesar dos resultados positivos nos experimentos, a solução não foi projetada para substituir balanceadores profissionais utilizados em ambientes de produção.

## 12. Finalização da aplicação

Quando não houver mais necessidade de utilizar o ambiente, os containers e os recursos associados poderão ser encerrados com o seguinte comando:

```bash
docker compose down
```

Com isso, a infraestrutura do ShopNow será finalizada.
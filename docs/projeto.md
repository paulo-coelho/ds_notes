A área de computação distribuída é rica em aplicações e desenvolvê-los é topar de frente com vários problemas e decidir como resolvê-los ou contorná-los e, por isto, nada melhor que um projeto para experimentar em primeira mão as angústias e prazeres da área. 
Assim, proponho visitarmos o material destas notas à luz de uma aplicação genérica mas real, desenvolvida por vocês enquanto vemos a teoria.

O projeto consiste em uma com dois tipos de papeis, **clientes** e **administradores**. **Administradores** gerenciam cadastros de clientes e produtos. **Clientes** realizam pedidos de compra.

As funcionalidades são expostas para estes usuários via **dois tipos** de  aplicações distintas, o **portal de pedidos**  e o **portal administrativo**, mas ambos manipulam a **mesma base de dados**.

Múltiplas instâncias de cada portal podem existir e cada instância mantém um **cache** da base de dados em memória, com as entradas mais recentemente acessadas.

<!--
A base de dados é replicada em outros nós usando um protocolo de **difusão atômica**.
-->

A totalidade da base é particionada usando ***consistent hashing***.
Cada partição é replicada em outros nós usando um protocolo de **difusão atômica**.

![Projeto](drawings/projeto.drawio#0)

A arquitetura do sistema será **híbrida**, contendo um pouco de Cliente/Servidor e Peer-2-Peer, além de ser multicamadas.
Apesar de introduzir complexidade extra, também usaremos **múltiplos mecanismos para a comunicação** entre as partes, para que possam experimentar com diversas abordagens.

O sistemas tem duas **aplicações**, CLI ou GUI, para os dois tipos de papeis do sistema, clientes e administradores.
Estas aplicações se comunicarão com os portais para manipular os dados dos clientes e associados a cada cliente.
A aplicação do administrador manipula clientes e produtos, isto é, permite o CRUD de clientes e produtos.
A aplicação do cliente permite manipular os dados associados aos pedidos, ou seja, comprar produtos previamente cadastrados.

O cadastro do cliente inclui a provisão de um identificador único do cliente CID (*client id*).
Os dados dos clientes são mantidos em uma tabela CID -> Dados do Cliente, em memória (use uma tabela hash). 
O CID tem tipo String; **você** deve decidir o que compõe os dados do cliente, mas eles **devem** ser armazenados como uma string JSON.

O cadastro do produto inclui a provisão de um identificador único do produto PID (*product id*).
Os dados dos produtos são mantidos em uma tabela PID -> Dados do Produto, em memória (use uma tabela hash). 
O PID tem tipo String; **você** deve decidir o que compõe os dados do produto, mas devem ser suficientes para manter a descrição (String), quantidade (inteiro) e preço (ponto flutuante) do produto, e **devem** ser armazenados como uma string JSON.

A comunicação entre administradores e Portal Administrativo se dá obrigatoriamente via gRPC, conforme interface definida mais adiante.

Somente clientes devidamente cadastrados no sistema podem ter suas operações executadas.
O CID do cliente executando operações no portal de pedidos deve ser informado em cada operação para "autenticar" o cliente e autorizar a execução da operação.

A comunicação entre clientes e Portal de Pedidos se dá obrigatoriamente via gRPC, conforme interface definida mais adiante.

O cadastro do pedido inclui a geração de um identificador único do pedido OID (*order id*).
O OID tem tipo String; **você** deve decidir o que compõe os dados do pedido, mas devem ser suficientes para identificar o produto, a quantidade comprada e o preço para cada produto no pedido. **Devem** ser armazenados como uma string JSON.
A relação entre clientes e pedidos é mantida em uma tabela CID -> OID, em memória (use uma tabela hash). 

**Exemplo de armazenamento no portal de pedidos**: 
Cada pedido tem um identificador (Sring), e um corpo (JSON);
Os dados são mantidos em uma tabela hash e múltiplas entradas podem ser necessárias para armazenar e manter um pedido, isto é, algumas entradas podem ser de metadados, por exemplo, índice.
Por exemplo, para representar dois pedidos, `o1` e `o2`, com corpos `c1` e `c2`, associadas ao cliente `cliente1`, podemos ter as seguintes entradas.

* `cliente1 -> [o1,o2]`
* `cliente1:o1 -> c1`
* `cliente1:o2 -> c2`

Com este formato, podemos identificar os pedidos associados ao `cliente1` e, a partir desta lista, identificar o conteúdo associado a cada pedido.
Este formato também permite que múltiplos clientes tenham pedidos com o mesmo OID.


## Casos de Uso

###### Interfaces

**O formato exato em que os dados serão armazenados pode variar dentro do JSON**, mas a API apresentada deve ter a assinada definida a seguir:

```proto
syntax = "proto3";

option java_multiple_files = true;
option java_package = "br.ufu.facom.gbc074.project";

package project;

message Client {
  // Client ID
  string CID = 1;
  // JSON string representing client data: at least a name
  string data = 2;
}

message Product {
  // Produto ID
  string PID = 1;
  // JSON string representing produto data: at least product name, price, and
  // quantity
  string data = 2;
}

message Order {
  // Order ID
  string OID = 1;
  // CLient ID
  string CID = 2;
  // JSON string representing at least array of PIDs, prices, and quantities
  string data = 3;
}

message Reply {
  // Error code: 0 for success
  int32 error = 1;
  // Error message, if error > 0
  optional string description = 2;
}

message ID {
  // generic ID for CID, PID and OID
  string ID = 1;
}

service AdminPortal {
  rpc CreateClient(Client) returns (Reply) {}
  rpc RetrieveClient(ID) returns (Client) {}
  rpc UpdateClient(Client) returns (Reply) {}
  rpc DeleteClient(ID) returns (Reply) {}
  rpc CreateProduct(Product) returns (Reply) {}
  rpc RetrieveProduct(ID) returns (Product) {}
  rpc UpdateProduct(Product) returns (Reply) {}
  rpc DeleteProduct(ID) returns (Reply) {}
}

service OrderPortal {
  rpc CreateOrder(Order) returns (Reply) {}
  rpc RetrieveOrder(ID) returns (Order) {}
  rpc UpdateOrder(Order) returns (Reply) {}
  rpc DeleteOrder(ID) returns (Reply) {}
  rpc RetrieveClientOrders(ID) returns (stream Order) {}
}
```

O JSON do cliente deve ter o formato apresentado a seguir. Campos adicionais podem existir, mas não devem ser obrigatórios.

```json
{
  "CID": "123",
  "name": "Paulo",
  "non-mandatory-field": "xxxxxxx"
}
```

O JSON do produto deve ter o formato apresentado a seguir. Campos adicionais podem existir, mas não devem ser obrigatórios.

```json
{
  "PID": "123",
  "name": "caneta",
  "quantity": "40",
  "price": "2.50",
  "non-mandatory-field": "xxxxxxx"
}
```

O JSON do pedido deve ter o formato apresentado a seguir. Campos adicionais podem existir, mas não devem ser obrigatórios.

```json
{
  "OID": "123",
  "CID": "456",
  [
    {
      "name": "caneta",
      "quantity": "30",
      "price": "2.50",
      "non-mandatory-field": "xxxxxxx"
    },
    {
      "name": "lapis",
      "quantity": "10",
      "price": "1.50",
      "non-mandatory-field": "xxxxxxx"
    },
    {
      "name": "borracha",
      "quantity": "3",
      "price": "0.50",
      "non-mandatory-field": "xxxxxxx"
    }
  ],
   "non-mandatory-field": "xxxxxxx"
}
```

###### Manipulação Clientes
* Inserção de Cliente
    * Administrador
        * Gera um CID para cada cliente, baseado em seu nome ou outro atributo único.
        * Informa o CID para o cliente
    * Portal Administrador
        * Executa a operação e retorna código de erro/sucesso.
            * Se cliente existia, falha a operação.
            * Se cliente não existia, insere dados no banco e atualiza a cache.
    * Cliente
        * Recebe CID diretamente do administrador (já sabe o seu CID)
* Modificação de Cliente
    * Administrador
        * Determina CID de cliente a ser modificado.
    * Portal Administrador
        * Executa a operação e retorna código de erro/sucesso.
            * Se cliente existe, atualiza o cliente e atualiza a cache.
            * Se cliente não existe, retorna erro.
* Recuperação de Clientes
    * Administrador
        * Determina CID de cliente a ser recuperado
    * Portal Administrador
        * Executa a operação e retorna dados do cliente.
            * Se cliente não existe na cache, pesquisa banco de dados e atualiza a cache caso encontre.
            * Se cliente existe na cache, retorna informação.
            * Se cliente não existe na cache, retorna objeto vazio com CID igual a 0 (zero).
* Remoção de Cliente
    * Administrador
        * Determina CID de cliente a ser removido
    * Portal Administrador
        * Executa a operação e retorna código de erro/sucesso.
            * Apaga dados do banco.
            * Apaga dados da cache, se existe.

###### Manipulação Produtos
* Inserção de Produto
    * Administrador
        * Gera um PID para cada produto, baseado em seu nome ou outro atributo único.
    * Portal Administrador
        * Executa a operação e retorna código de erro/sucesso.
            * Se produto existia, falha a operação.
            * Se produto não existia, insere dados no banco e atualiza a cache.
    * Cliente
        * Recebe PID diretamente do administrador (já sabe o PID do produto que deseja pedir)
* Modificação de Produto
    * Administrador
        * Determina PID de produto a ser modificado.
    * Portal Administrador
        * Executa a operação e retorna código de erro/sucesso.
            * Se produto existe, atualiza o produto e atualiza a cache.
            * Se produto não existe, retorna erro.
* Recuperação de Produtos
    * Administrador
        * Determina PID de produto a ser recuperado
    * Portal Administrador
        * Executa a operação e retorna dados do produto.
            * Se produto não existe na cache, pesquisa banco de dados e atualiza a cache caso encontre.
            * Se produto existe na cache, retorna informação.
            * Se produto não existe na cache, retorna objeto vazio com PID igual a 0 (zero).
* Remoção de Produto
    * Administrador
        * Determina PID de produto a ser removido
    * Portal Administrador
        * Executa a operação e retorna código de erro/sucesso.
            * Apaga dados do banco.
            * Apaga dados da cache, se existe.

###### Manipulação de Tarefas dos Pedidos

Nesta descrição, a interação com a cache foi omitida, mas deverá ser implementada.

* Inserção de Pedido
    * Cliente
        * Usa o CID informado pelo administrador
    * Portal de Pedidos
        * Autentica o cliente pelo seu CID
        * Executa a operação e retorna OID para novo pedido vazio.
* Modificação de Pedido
    * Cliente
        * Usa o CID informado pelo administrador
        * Usa o OID informado retornado pelo Portal de Pedidos
    * Portal de Pedidos
        * Autentica o cliente pelo seu CID
        * Verifica disponibilidade de produto e quantidade.
        * Ajusta quantidade disponível de produto, se necessário.
        * A informação de quantidade zero (0) remove o produto do pedido.
        * Executa a operação e retorna código de erro/sucesso.
* Enumeração do pedido
    * Cliente
        * Usa o CID informado pelo administrador
        * Usa o OID informado retornado pelo Portal de Pedidos 
    * Portal de Pedidos
        * Autentica o cliente pelo seu CID
        * Caso o pedido exista, retorna informações dos produtos e valor total do pedido.
        * Caso contrário, retorna objeto vazio com OID igual a 0 (zero).
* Cancelamento do pedido
    * Cliente
        * Usa o CID informado pelo administrador
        * Usa o OID informado retornado pelo Portal de Pedidos 
    * Portal de Pedidos
        * Autentica o cliente pelo seu CID
        * Ajusta quantidade disponível de produto, se necessário.
        * Executa a operação e retorna código de erro/sucesso.
* Enumeração de pedidos
    * Cliente
        * Usa o CID informado pelo administrador
    * Portal de Pedidos
        * Autentica o cliente pelo seu CID
        * Retorna lista de pedidos associados ao cliente.

## Interação entre portais

### Etapa 1 - Usuários/Portais

* Implementar os casos de uso usando como cache tabelas hash locais aos portais Administrador e de Pedidos.
* Certificar-se de que todas as API possam retornar erros/exceções e que estas são tratadas; explicar sua decisão de tratamento dos erros.
* Implementar testes automatizados de sucesso e falha de cada uma das operações na API.
* Documentar o esquema de dados usados nas tabelas.
* O sistema deve permitir a execução de múltiplos clientes, administradores, portais de pedido e portais administrador.
* Implementar a propagação de informação entre as diversas caches do sistema usando necessariamente *pub-sub*, já que a comunicação é de 1 para muitos.
* Gravar um vídeo de no máximo 10 minutos demonstrando que os requisitos foram atendidos.

![Projeto](drawings/projeto.drawio#1)

<!--

### Etapa 2 - Banco de dados Replicado
Nesta etapa você modificará o sistema para que atualizações dos dados sejam feitas consistentemente entre todas as réplicas usando um protocolo de difusão atômica.

![Projeto](drawings/projeto.drawio#3)

* Objetivos
    * Replicar a base de dados para obter tolerância a falhas.

* Desafios
    * Certificar-se de que o portais são máquinas de estados determinística
    * Compreender o uso de Difusão Atômica em nível teórico
    * Compreender o uso de Difusão Atômica em nível prático 
        * Use [Ratis](https://paulo-coelho.github.io/ds_notes/cases/ratis) para java
        * Para Python, [PySyncObj](https://github.com/bakwc/PySyncObj) é uma boa opção.
        * Aplicar difusão atômica na replicação do servidor
        * Utilizar um banco de dados simples do tipo chave-valor, necessariamente [LevelDB](https://github.com/google/leveldb) ou [LMDB](https://git.openldap.org/openldap/openldap/tree/mdb.master)
            * Embora originalmente escritas em C++/C, há *ports* para diversas outras linguagens, (e.g., [aqui](https://github.com/lmdbjava/lmdbjava) e [aqui](https://github.com/dain/leveldb))
        * Utilizar três réplicas (servidores)
* Portal
    * A API para clientes e administradores continua a mesma.
    * Requisições para o servidor (linha contínua) são encaminhadas via Ratis (linha tracejada) para ordená-las e entregar a todas as réplicas (linha pontilhada) para só então serem executadas e respondidas (pontilhado fino).  
    * Dados não são mais armazenados em disco pela sua aplicação mas somente via Ratis.
* Cliente e Administrador
    * Sem alteração.
* Testes
    * O mesmo *framework* de testes deve continuar funcional
* Comunicação
    * Entre cliente/administrador e portais, não é alterado.
    * Entre servidores, usar Ratis/PySyncOb
* Apresentação
    * Sem alteração, isto é, gravar um vídeo demonstrando que os requisitos foram atendidoa.

-->

<!--

### Etapa 2 - Banco de dados.
Nesta etapa você modificará o sistema para que modificações dos dados sejam refletidas no banco de dados particionado implementado usando *consistent hashing*.

![Projeto](drawings/projeto.drawio#2)

### Etapa 3 - Replicação
Nesta etapa você modificará o sistema para que todas as modificações nas partições do banco de dados sejam replicadas em outras partições.

![Projeto](drawings/projeto.drawio#0)




### Etapa 2 - Cache

Nesta segunda etapa você modificará o sistema para que os portais, em vez de armazenar os dados em tabelas hash locais, o façam em uma tabela remota, compartilhada entre os portais.
A tabela hash remota é particionada para permitir o armazenamento de mais dados do que caberiam em apenas um computador. Isto é, é essencialmente uma Distributed Hash Table, a base dos bancos de dados NoSQL como Redis, Memcached ou Cassandra.

* Portais
    * Os dados são armazenados no banco distribuído
    * A comunicação com o banco é feita via MQTTP ou Kafka
* Banco
    * As partições usam *consistent hashing*  para distribuir os dados
    * Uma requisição feita para a partição errada deve ser encaminhada para a partição correta usando o algoritmo de roteamento Chord.

### Etapa 3 - Durabilidade

Nesta etapa tornaremos todas as operações feitas no banco de dados permanentes por meio de um log remoto[^log] ou pela replicação das partições. Mais detalhes se seguirão.


## Versão independente

no sql cresceram rapidamente em uso e implementacoes.a facilidade e familiaridade do sql tem atrativos fortes. cockroach and yugabyte.
entender como funcionam é importate para qquer um interessado em SD.

In this project we will develop a rudimentary no SQL database and use that many difficulties to implement such a project to introduce concepts and frameworks related to the development of distributing systems. We will start by exploring the Waze stocked each other in a Distributed system. Then we will Dan will be explored different Architectures used to combine the efforts of components in the distributed system. Next we explore the guarantees that databases can provide to their users and how these guarantees are insured. 

To do move the session to an introductory part with either the preface or introduction itself.

O objetivo deste projeto é praticar o projeto de sistemas distribuídos, usando várias arquiteturas e tecnologias.
A ideia é implementar um banco de dados NoSQL (Not only SQL) rudimentar.
Mesmo uma versão simples de um banco de dados distribuído é um sistema complexo e por isso você deverá trabalhar em fases. Infelizmnte enquanto esta abordagem facilita a jornada, ela poderá levar a um pouco de retrabalho no final.

Para garantir que todo o seu esforço será concentrado no lugar certo e que sua avaliação seja justa, atente-se aos detalhes e aos passos na especificação abaixo.

* Etapa 1 - Cliente/Servidor usando RPC
    * Objetivos
        * Hash Table acessível remotamente por interface CRUD usando gRPC.
        * Armazenamento em disco com recuperação de dados no caso de falhas
    * Desafios
        * Especificação do protocolo para dados genéricos
        * Armazenamento atômico no disco
        * Multithreading para garantir escalabilidade
        * Controle de concorrência para garantir corretude nos dados armazenados.
    * Servidor
        * Todos os dados devem ser armazenados em um mapa Chave-Valor (Dicionário)
        * Chave é um número de precisão arbitrária do tipo BigInteger
        * Valor é uma tripla (Versão, Timestamp, Dados)
             * Versão é um inteiro com 64 bits (long)
             * Timestamp é um inteiro com 64 bits (long)
             * Dados é um vetor de bytes (byte[]) de tamanho arbitrário
        * O servidor implementa a seguinte API:
             * set(k,ts,d):(e,v') 
                 * adiciona ao mapa a entrada k-v, caso não exista uma entrada com a chave k, onde v=(1,ts,d)
                 * retorna a tupla (e,v') onde e=SUCCESS e v'=NULL se k-v foi inserido
                 * retorna a tupla (e,v') onde e=ERROR e v'=(ver,ts,data) se já existia uma entrada no banco de dados com a chave k e vers, ts e data correspondem, respectivamente, à versão, timestamp e dados de tal entrada
             * get(k):(e,v') 
                 * retorna a tupla (e,v') onde e=ERROR e v'=NULL se não há entrada no banco de dados com chave k
                 * retorna a tupla (e,v') onde e=SUCCESS e v'=(ver,ts,data) se já existia uma entrada no banco de dados com a chave k e vers, ts e data correspondem, respectivamente, à versão, timestamp e dados de tal entrada 
             * del(k):(e,v')
                 * remove a entrada k-v' do banco de dados se existir
                 * retorna a tupla (e,v') onde e=SUCCESS e v'=(ver,ts,data) se já existia uma entrada no banco de dados com a chave k e vers, ts e data correspondem, respectivamente, à versão, timestamp e dados de tal entrada 
                 * retorna a tupla (e,v') onde e=ERROR e v'=NULL se não existia entrada com chave k no banco de dados.
             * del(k,vers):(e,v')
                 * remove a entrada k-v' do banco de dados se existir e tiver versão v
                 * retorna a tupla (e,v') onde e=SUCCESS e v'=(vers,ts,data) se já existia uma entrada no banco de dados com a chave k e vers e ts e data correspondem, respectivamente, timestamp e dados de tal entrada 
                 * retorna a tupla (e,v') onde e=ERROR_NE e v'=NULL se não existia entrada com chave k no banco de dados.
                 * retorna a tupla (e,v') onde e=ERROR_WV e v'=(ver',ts,data) se já existia uma entrada no banco de dados com a chave k  version vers' not equal to vers, e  ts e data correspondem, respectivamente, timestamp e dados de tal entrada
             * testAndSet(k,v,vers):(e,v')
                 * atualiza o mapa se a versão atual no sistema corresponde à versão especificada.
                 * retorna a tupla (e,v') onde e=SUCCESS e v'=(ver,ts,data) se já existia uma entrada no banco de dados com a chave k e version vers, e  ts e data correspondem, respectivamente, timestamp e dados de tal entrada
                 * retorna a tupla (e,v') onde e=ERROR_NE e v'=NULL se não existia uma entrada no banco com chave k;
                 * retorna a tupla (e,v') onde e=ERROR_WV e v'=(ver',ts,data) se já existia uma entrada no banco de dados com a chave k  version vers' not equal to vers, e  ts e data correspondem, respectivamente, timestamp e dados de tal entrada
        * O mapa deve ser salvo em disco
            * Com periodicidade configurável, os dados do mapa devem ser salvos em disco.
            * Os dados em disco devem corresponder a uma versão dos dados em memória. Para entender, veja a seguinte sequência de eventos, que leva a uma versão em disco que nunca ocorreu em memória.
                * Dados em memória (1/lala, 2/lele, 3/lili)
                * Cópia para disco iniciada
                * Dados em disco (1/lala)
                * Dados em memória (1/lolo, 2/lele, 3/lili)
                * Dados em disco (1/lala, 2/lele)
                * Dados em memória (1/lolo, 2/lele, 3/lulu)
                * Dados em disco (1/lala, 2/lele, 3/lulu)

    * Cliente
        * O cliente deve implementar uma UI que permita a interação com o banco de dados usando todas as API
    * Testes
        * Um segundo cliente implementará as seguintes baterias de testes no sistema
            * Teste de todas as API levando a todos os tipos de resultados (sucesso e erro)
            * Teste de estresse em que a API seja exercitada pelo menos 1000 vezes e o resultado final deve ser demonstrado como esperado (por exemplo, inserir 1000 entradas com chaves distintas, atualizar a todas as entradas, ler todas a entradas e verificar que o valor, isto é, versão e dados, correspondem aos esperados.
        * Todos os testes apresentam os resultados esperados mesmo quando múltiplos clientes de teste são executados em paralelo.
    * Comunicação
        * Toda a comunicação entre cliente e servidor deve ser feita usando gRPC
    * Apresentação
        * Demonstrar que todos os itens da especificação foram seguidos
        * Demonstrar a corretude do sistema frente aos testes
        * Enumerar outros testes e casos cobertos implemententados
        * Demonstrar comportamento quando comunicação é interrompida no meio do teste

* Etapa 2 - Tolerância a Falhas
    * Objetivos 
         * Replicar o servidor para obter tolerância a falhas.
    * Desafios
         * Certificar-se de que o servidor é uma máquina de estados determinística
         * Compreender o uso de Difusão Atômica em nível teórico
         * Compreender o uso de Difusão Atômica em nível prático (Via [Ratis](https://lasarojc.github.io/ds_notes/fault/#estudo-de-caso-ratis))
         * Aplicar difusão atômica na replicação do servidor
    * Servidor
         * A API permanece a mesma e implementada via gRPC.
         * Requisições para o servidor (linha contínua) são encaminhadas via Ratis (linha tracejada) para ordená-las e entregar a todas as réplicas (linha pontilhada) para só então serem executadas e respondidas (pontilhado fino).  
         * Dados não são mais armazenados em disco pela sua aplicação mas somente via Ratis.
    * Cliente
         * Sem alteração.
    * Testes
         * O mesmo *framework* de testes deve continuar funcional
    * Comunicação
         * Entre cliente e servidor, usar gRPC
         * Entre servidores, usar Ratis
    * Apresentação
         * Sem alteração, isto é, gravar um vídeo demonstrando que os requisitos foram atendidos.

* Etapa 2 - P2P
    * Objetivos
        * DHT com roteamento estilo Chord
        * Armazenamento em Log de operações e em arquivo de snapshots
        * Comunicação usando RPC
    * Desafios
        * Uso adequado da interface funcional do RPC
        * Uso do log + snapshots para recuperação
        * Roteamento no anel
        * Bootstrap dos processos
        * Log Structured Merge Tree

-->

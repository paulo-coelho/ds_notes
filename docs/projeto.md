A área de computação distribuída é rica em aplicações e desenvolvê-los é topar de frente com vários problemas e decidir como resolvê-los ou contorná-los e, por isto, nada melhor que um projeto para experimentar em primeira mão as angústias e prazeres da área. 
Assim, proponho visitarmos o material destas notas à luz de uma aplicação genérica mas real, desenvolvida por vocês enquanto vemos a teoria.

O projeto consistem em implementar um sistema de armazenamento chave-valor (*key-value store = KVS*) multi-versão.

A arquitetura do sistema será **híbrida**, contendo um pouco de cliente/servidor, publish/subscribe e Peer-2-Peer, além de ser multicamadas.
Apesar de introduzir complexidade extra, também usaremos **múltiplos mecanismos para a comunicação** entre as partes, para que possam experimentar com diversas abordagens.

O sistema utiliza a arquitetura cliente-servidor na comunicação entre o cliente, que realiza as operações em cima do sistema de armazenamento, e o servidor, que atualiza seu estado de acordo e retorna o resultado ao cliente.

Múltiplas instâncias do servidor podem ser executados simultaneamente para prover aumentar a disponibilidade do serviço e/ou atender um maior número de clientes (escalar).
O detalhamento do comportamento e implementação, neste caso, será diferente para cada etapa do projeto, sendo detalhado nas seções seguintes.

Tanto a aplicação cliente quanto a aplicação servidor devem, obrigatoriamente, possuir uma interface de linha de comando (*command line interface - CLI*) para execução e interação.

A aplicação cliente manipula cada chave **K** em operações de escrita e leitura de acordo com a descrição e interface apresentada mais adiante.
O sistema utiliza a noção de versões **i** de valores, ou seja, cada atualização de uma mesma chave **K** para uma valor **V** gera uma nova versão da mesma.
A tupla **(K,V,i)** significa que a chave **K** possui valor **V** em sua versão **i**.
Cada atualização para a mesma chave **K** com um novo valor **V'** gera uma nova versão **j**, em que **j = `System.currentTimeMillis()`**[^1].
[^1]: [System.currentTimeMillis() em Java](https://docs.oracle.com/javase/8/docs/api/java/lang/System.html#currentTimeMillis--)

As consultas podem ser feitas sobre uma versão específica ou sobre a versão mais recente da chave.
Caso seja especificada uma versão **i**, a consulta deve retornar o valor **V** para a chave **K** com versão imediatamente menor ou igual a **i**.

Tanto as consultas quanto as atualizações pode ser feitas para múltiplas chaves simultaneamente. A API completa é apresentada a seguir.

As chaves e valores devem ser do tipo *String* enquanto as versões são variáveis do tipo *inteiro longo*.

A comunicação entre cliente e servidor deve ser, obrigatoriamente, realizada via gRPC de acordo com a interface definida adiante.
A implementação que não seguir a interface definida não será avaliada e terá atribuída a nota zero.

## Casos de Uso

### Interface

O formato exato em que os dados serão armazenados pode variar na sua implementação, mas a API apresentada deve, **obrigatoriamente**, ter a assinada definida a seguir:

```proto
syntax = "proto3";

option java_multiple_files = true;
option java_package = "br.ufu.facom.gbc074.project";

package project;

message KeyRequest {
  // key
  string key = 1;
  // version
  optional int64 ver = 2;
}

message KeyRange {
    // from this key
    KeyRequest from = 1;
    // until this key
    KeyRequest to = 2;
}

message KeyValueRequest {
  // key
  string key = 1;
  // value
  string val = 2;
}

message KeyValueVersionReply {
  // key
  string key = 1;
  // value
  string val = 2;
  // version
  int64 ver = 3;
}

message PutReply {
  // key
  string key = 1;
  // value
  string old_val = 2;
  // old version
  int64 old_ver = 3;
  // assigned version
  int64 ver = 4;
}



service KeyValueStore {
  rpc Get(KeyRequest) returns (KeyValueVersionReply) {}
  rpc GetRange(KeyRange) returns (stream KeyValueVersionReply) {}
  rpc GetAll(stream KeyRequest) returns (stream KeyValueVersionReply) {}
  rpc Put(KeyValueRequest) returns (PutReply) {}
  rpc PutAll(stream KeyValueRequest) returns (stream PutReply) {}
  rpc Del(KeyRequest) returns (KeyValueVersionReply) {}
  rpc DelRange(KeyRange) returns (stream KeyValueVersionReply) {}
  rpc DelAll(stream KeyRequest) returns (stream KeyValueVersionReply) {}
  rpc Trim(KeyRequest) returns (KeyValueVersionReply) {}
}
```

### Descrição dos métodos

*  `rpc Get(KeyRequest) returns (KeyValueVersionReply) {}`: retorna valor para chave e versão informados pelo cliente
    * Cliente:
        - informa chave e versão que deseja obter do servidor
        - versão pode ser deixada em branco para obter valor mais recente
    * Servidor:
        - retorna chave, valor e versão de acordo com requisição do cliente:
            - caso versão esteja em branco, retorna valor e versão mais recentes
            - caso contrário, retorna valor e versão imediatamente menor ou igual à versão informada
*  `rpc GetRange(KeyRange) returns (stream KeyValueVersionReply) {}`: retorna valores no intervalo entre as duas chaves informadas pelo cliente
    * Cliente:
        - informa as duas tuplas chave e versão referente ao intervalo de chaves que deseja obter do servidor
        - versão pode ser deixada em branco para obter valores mais recentes
    * Servidor:
        - retorna conjunto de chaves, valores e versões de acordo com requisição do cliente:
            - caso versões estejam em branco, retorna valores e versões mais recentes
            - caso contrário, escolhe a maior entre as versões recebidas do cliente e retorna valores e versões imediatamente menores ou iguais a esta versão.
*  `rpc GetAll(stream KeyRequest) returns (stream KeyValueVersionReply) {}`: retorna valores para conjunto de chaves informado pelo cliente
    * Cliente:
        - informa conjunto desordenado de tuplas com chave e versão
        - versão pode ser deixada em branco para obter valores mais recentes
    * Servidor:
        - retorna conjunto de chaves, valores e versões de acordo com requisição do cliente:
            - caso versão esteja em branco, retorna valores e versões mais recentes
            - caso contrário, escolhe a maior entre as versões recebidas do cliente e retorna valores e versões imediatamente menores ou iguais a esta versão.
*  `rpc Put(KeyValueRequest) returns (PutReply) {}`: atualiza/insere valor e chave informados, retornando valor para chave e versão anteriores, bem como a nova versão atribuída
    * Cliente:
        - informa chave e valor para inserir/atualizar no servidor
    * Servidor:
        - retorna chave, com valor e versão antigos (se houver), bem como a nova versão atribuída ao valor
*  `rpc PutAll(stream KeyValueRequest) returns (stream PutReply) {}`: atualiza/insere valores e chaves informados, retornando, para cada chave, valor e versão anteriores, bem como a nova versão atribuída
    * Cliente:
        - informa conjunto desordenado de tuplas chave, valor
    * Servidor:
        - retorna conjunto de tuplas com cada chave, com valor e versão antigos (se houver), bem como a nova versão atribuída ao valor
*  `rpc Del(KeyRequest) returns (KeyValueVersionReply) {}`: remove todos os valores associados à chave e retorna valor para chave e versão mais atual 
    * Cliente:
        - informa chave que deseja excluir do servidor
        - versão *deve* ser deixada em branco
    * Servidor:
        - retorna chave, com valor e versão mais recentes, ou valores vazios, caso chave não exista
*  `rpc DelRange(KeyRange) returns (stream KeyValueVersionReply) {}`: remove valores no intervalo entre as duas chaves informadas pelo cliente, retornando os valores mais atuais para estas chaves
    * Cliente:
        - informa as duas tuplas, referente ao intervalo de chaves que deseja excluir do servidor
        - versão *deve* ser deixada em branco
    * Servidor:
        - retorna tuplas com chave, valor e versão mais recentes, ou valores vazios, caso chave não exista
*  `rpc DelAll(stream KeyRequest) returns (stream KeyValueVersionReply) {}`: remove valores para conjunto de chaves informado pelo cliente e retorna valores mais atuais
    * Cliente:
        - informa conjunto desordenado de tuplas com as chaves
        - versão *deve* ser deixada em branco
    * Servidor:
        - retorna tuplas com chave, valor e versão mais recentes, ou valores vazios, caso chave não exista
*  `rpc Trim(KeyRequest) returns (KeyValueVersionReply) {}`: remove todos os valores associados à chave, exceto a versão mais recente, e retorna valor e versão para a chave 
    * Cliente:
        - informa chave
        - versão *deve* ser deixada em branco
    * Servidor:
        - retorna chave, com valor e versão mais recentes, ou valores vazios, caso chave não exista

## Etapa 1 

* Implementar os casos de uso usando como cache tabelas hash locais aos servidores.
* Certificar-se de que todas as API possam retornar erros/exceções e que estas são tratadas; explicar sua decisão de tratamento dos erros.
* Implementar testes automatizados de sucesso e falha de cada uma das operações na API.
* Documentar o esquema de dados usados nas tabelas.
* O sistema deve permitir a execução de múltiplos clientes e servidores.
* Implementar a propagação de informação entre as diversas caches do sistema usando necessariamente *pub-sub*, já que a comunicação é de 1 para muitos.
* Gravar um vídeo de no máximo 10 minutos demonstrando que os requisitos foram atendidos.

### Comunicação entre servidores

O suporte a múltiplos servidores deve garantir que clientes possam se conectar a instâncias diferentes de servidores e ainda sim sejam capazes de manipular os dados armazenados.
Por exemplo, o cliente *c1* pode inserir um valor para a chave **K1** no servidor *s1* e deve ser capaz de recuperar o valor inserir para a mesma chave a partir de um segundo servidor *s2*.

Para isto, cada servidor deve publicar qualquer alteração nas chaves em um broker pub-sub, em tópico conhecido pelos demais, a partir do qual estes receberão as mudanças de estado e atualizarão suas próprias tabelas.

A figura a seguir ilustra a arquitetura exigida para a Etapa 1 do Projeto.

![Projeto](drawings/projeto.drawio#0)


## Linguagens aceitas

**APENAS** serão aceitas estas linguagens:

* Java
* C
* Python
* Go
* Rust

Trabalhos em outra linguagem não serão corrigidos e receberão nota zero!
<!--
A base de dados é replicada em outros nós usando um protocolo de **difusão atômica**.
-->


<!--
O projeto consiste em uma com dois tipos de papeis, **clientes** e **administradores**. **Administradores** gerenciam cadastros de clientes e produtos. **Clientes** realizam pedidos de compra.

As funcionalidades são expostas para estes usuários via **dois tipos** de  aplicações distintas, o **portal de pedidos**  e o **portal administrativo**, mas ambos manipulam a **mesma base de dados**.

Múltiplas instâncias de cada portal podem existir e cada instância mantém um **cache** da base de dados em memória, com as entradas mais recentemente acessadas.


A totalidade da base é particionada usando ***consistent hashing***.
Cada partição é replicada em outros nós usando um protocolo de **difusão atômica**.


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
  "products": [
    {
      "PID": "123",
      "quantity": "30",
      "price": "2.50",
      "non-mandatory-field": "xxxxxxx"
    },
    {
      "PID": "321",
      "quantity": "10",
      "price": "1.50",
      "non-mandatory-field": "xxxxxxx"
    },
    {
      "PID": "456",
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
Assuma que o cliente recebeu o CID do administrador, ou seja, ele sabe seu próprio CID.

* Inserção de Pedido
    * Cliente
        * Usa o CID informado pelo administrador
        * Informa o OID desejado para o pedido
        * Informa zero ou mais pedidos a serem feitos
    * Portal de Pedidos
        * Autentica o cliente pelo seu CID
        * Verifica se o OID informado já foi usado
        * Verifica disponibilidade de produto e quantidade.
        * Ajusta quantidade disponível de produto, se necessário.
        * Executa a operação e retorna código de erro/sucesso.
* Modificação de Pedido
    * Cliente
        * Usa o CID informado pelo administrador
        * Usa o OID informado na criação do pedido
    * Portal de Pedidos
        * Autentica o cliente pelo seu CID
        * Verifica disponibilidade de produto e quantidade.
        * Ajusta quantidade disponível de produto, se necessário.
        * A informação de quantidade zero (0) remove o produto do pedido.
        * Executa a operação e retorna código de erro/sucesso.
* Enumeração do pedido
    * Cliente
        * Usa o CID informado pelo administrador
        * Usa o OID informado na criação do pedido
    * Portal de Pedidos
        * Autentica o cliente pelo seu CID
        * Caso o pedido exista, retorna informações dos produtos e valor total do pedido.
        * Caso contrário, retorna objeto vazio com OID igual a 0 (zero).
* Cancelamento do pedido
    * Cliente
        * Usa o CID informado pelo administrador
        * Usa o OID informado na criação do pedido
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
### Etapa 2 - Banco de dados particionado e Replicação

Nesta etapa você modificará o sistema para que modificações dos dados sejam refletidas no banco de dados particionado (para aumentar a escalabilidade) implementado usando *consistent hashing*, além de
prover tolerância a falhas por meio da replicação de cada partição usando um protocolo de difusão atômica.

![Projeto](drawings/projeto.drawio#0)

* Objetivos
    * Particionar e replicar a base de dados para obter escalabilidade e tolerância a falhas.

* Desafios
    * Compreender o uso de *consistent hashing* em nível teórico
    * Compreender o uso de *consistent hashing* em nível prático
        * Utilizar uma estrutura com duas partições, partição 0 e partição 1, equivalente a uma rede *Chord* com *m = 1*
        * As chaves de cada objeto (*PID, CID, OID*) serão utilizadas para definir a partição em que será armazenado utilizando a saída da função "*mod 2*" (resto da divisão por 2)
    * Certificar-se de que o portais são máquinas de estados determinísticas (API descrita oferece esta garantia)
    * Compreender o uso de Difusão Atômica em nível teórico
    * Compreender o uso de Difusão Atômica em nível prático 
        * Use [Ratis](https://paulo-coelho.github.io/ds_notes/cases/ratis) para java
        * Para Python, utilize [PySyncObj](https://github.com/bakwc/PySyncObj)
        * Aplicar difusão atômica na replicação do cada partição
        * Utilizar um banco de dados simples do tipo chave-valor em cada réplica de cada partição, necessariamente [LevelDB](https://github.com/google/leveldb) ou [LMDB](https://git.openldap.org/openldap/openldap/tree/mdb.master)
            * Embora originalmente escritas em C++/C, há *ports* para diversas outras linguagens, (e.g., [aqui](https://github.com/lmdbjava/lmdbjava) e [aqui](https://github.com/dain/leveldb))
        * Utilizar três réplicas (servidores) por partição
* Portal
    * A API para clientes e administradores continua a mesma.
    * Requisições para o servidor (linha contínua) são encaminhadas via Ratis/PySyncObj (linha tracejada) para ordená-las e entregar a todas as réplicas da partição selecionada (linha pontilhada) para só então serem executadas e respondidas (pontilhado fino).  
    * Dados não são mais armazenados pela sua aplicação original, mas somente via Ratis/PySyncObj
    * Os portais devem manter uma estrutura de cache local para evitar consultas desnecessárias
        * Defina a estratégia de cache utilizada e política de atualização
        * A efetivação da operação no banco de dados deve falhar se a operação for executada com dados antigos do cache, por exemplo, a tentativa de criação de um pedido para um cliente que estava no cache, mas não existe mais, deverá retornar erro ao ser executado na partição correspondente.
* Testes
    * O mesmo *framework* de testes deve continuar funcional
* Comunicação
    * Entre cliente/administrador e portais, não é alterado.
    * Entre servidores de armazenamento, usar Ratis/PySyncOb
* Apresentação
    * Sem alteração, isto é, gravar um vídeo demonstrando que os requisitos foram atendidoa.


<!--
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

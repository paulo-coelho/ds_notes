A proposta deste projeto é, utilizando o material destas notas, desenvolver uma aplicação genérica mas real, desenvolvida por vocês enquanto vemos a teoria.

O projeto consiste em implementar um sistema de armazenamento chave-valor (*key-value store = KVS*).

A arquitetura do sistema será **híbrida**, contendo um pouco de cliente/servidor e publish/subscribe, além de ser multicamadas.

O sistema contempla um conjunto de servidores para armazenamento dos pares chave-valor, além de clientes para acessar o sistema.
__O cliente será fornecido previamente.__

O sistema utiliza a arquitetura cliente-servidor na comunicação com os clientes, que invocam as operações definidas mais adiante.
Os servidores atualizam o estado de acordo e retornam o resultado a cada cliente.

Múltiplas instâncias do servidor  podem ser executados simultaneamente para aumentar a disponibilidade do serviço e/ou atender um maior número de clientes (escalar).

Os servidores devem, obrigatoriamente, possuir uma interface de linha de comando (*command line interface - CLI*) para execução.

A aplicação servidor manipula cada chave **K** em operações de escrita e leitura de acordo com a descrição e interface apresentadas mais adiante.
A tupla **(K,V,ver)** significa que a chave **K** possui valor **V** na versão **ver**.
Cada atualização para a mesma chave **K** com um novo valor **V'** gera uma nova versão para esta chave.
As chaves e valores devem ser do tipo *String* e ter no mínimo 3 caracteres.
As versões são valores *inteiros* que começam em 1 para a primeira inserção de cada chave e são incrementados para cada chave a cada inserção.

Você deve utilizar uma ou mais tabelas _hash_ para armazenar os dados nos servidores.
O esquema de armazenamento utilizado deve ser descrito na documentação do projeto submetido.

A comunicação entre clientes e servidores deve ser, **obrigatoriamente**, realizada via [gRPC](../cases/grpc) de acordo com a interface definida adiante.

A comunicação entre estes servidores será detalhada nas seções a seguir.

A implementação que não seguir o formato dos dados, a interface ou a estratégia de comunicação definidos **não será avaliada e terá atribuída a nota zero**.


## Casos de Uso

### Interface

O formato exato em que os dados serão armazenados pode variar na sua implementação, mas a API apresentada deve, **obrigatoriamente**, ter a assinatura definida a seguir para cada aplicação:

### Portal Administrativo

```proto
syntax = "proto3";

option java_multiple_files = true;
option java_package = "br.ufu.facom.gbc074.projeto.cadastro";

package kvs;

// Retorno de consultas
message Tupla{
  // a chave passada na consulta
  string chave = 1;
  // valor encontrado
  string valor = 2;
  // versao do valor para a chave
  int32 versao = 3;
}

// Parametro de entrada para insercoes
message ChaveValor{
  // a chave
  string chave = 1;
  // o valor
  string valor = 2;
}

// Parametro de entrada para consultas
message ChaveVersao{
  // a chave pesquisada
  string chave = 1;
  // a versao pesquisada (opcional)
  optional int32 versao = 2;
}

// Versao para snapshot
message Versao{
  // a versao pesquisada
  int32 versao = 1;
}

// Retorno de insercoes e remocoes
message Status {
  // valor do status (depende da operacao)
  int32 status = 1; 
  // mensagem de erro, caso necessário
  optional string msg   = 2;
}

service KVS {
  rpc Insere(ChaveValor) returns (Status) {}
  rpc Consulta(ChaveVersao) returns (Tupla) {}
  rpc Remove(ChaveVersao) returns (Status) {}
  rpc ConsultaVarias(stream ChaveVersao) returns (stream Tupla) {}
  rpc RemoveVarias(stream ChaveVersao) returns (stream Status) {}
  rpc Snapshot(Versao) returns (stream Tupla) {}
}
```

#### Descrição dos métodos

* `rpc Insere(ChaveValor) returns (Status) {}`
    * Cliente:
        - informa chave e valor.
    * Servidor:
        - cadastra a nova versão com o valor informado para a chave. O primeiro valor de cada chave tem versão 1.
        - retorna a versão associada ao novo valor no campo `status`.
        - retorna -1 e mensagem de erro caso a chave e/ou o valor sejam inválidos.
* `rpc Consulta(ChaveVersao) returns (Tupla) {}`
    * Cliente:
        - informa chave e versão (opcional) para consulta.
    * Servidor:
        - pesquisa valor para a chave informada com a versão imediatamente menor ou igual à versão passada como argumento, ou a versão mais recente caso nenhuma versão seja informada. 
        - retorna `Tupla` com chave, valor e versão encontrados.
        - retorna `Tupla` com dados em branco, caso contrário.
* `rpc Remove(ChaveVersao) returns (Status) {}`
    * Cliente:
        - informa chave e versão (opcional) para remoção.
    * Servidor:
        - remove valor associado à versão informada para a chave, ou todos os valores caso nenhuma versão seja informada.
        - retorna `Status` com quantidade de valores removidos para a chave.
        - retorna -1 e mensagem de erro caso a chave ou versão sejam inválidas, a chave não exista ou não exista a versão informada para a chave.
* `rpc ConsultaVarias(stream ChaveVersao) returns (stream Tupla) {}`
    * Cliente:
        - informa conjunto chaves e versões (opcional) para consulta.
    * Servidor:
        - para cada chave: pesquisa valor associado à versão imediatamente menor ou igual à versão passada como argumento, ou a versão mais recente caso nenhuma versão seja informada. 
        - para cada chave: retorna `Tupla` com chave, valor e versão encontrados ou `Tupla` com dados em branco, caso contrário.
* `rpc RemoveVarias(stream ChaveVersao) returns (Status) {}`
    * Cliente:
        - informa conjunto de chaves e versões (opcional) para remoção.
    * Servidor:
        - para cada chave: remove valor associado à versão informada, ou todos os valores caso nenhuma versão seja informada.
        - para cada chave: retorna `Status` com quantidade de valores removidos ou retorna -1 com mensagem de erro no caso de chave ou versão inválidas ou inexistentes.
* `rpc Snapshot(Versao) returns (stream Tupla) {}`
    * Cliente:
        - informa versão para snapshot.
    * Servidor:
        - retorna conjunto de `Tupla`s considerando, para cada chave, a versão imediatamente menor ou igual à versão solicitada.
        - retorna `Tupla` em branco caso a versão seja inválida ou não existam chaves que atendem ao critério.
 
* __IMPORTANTE:__ uma chave que teve todos os valores removidos para todas as versões deve fazer novas inserções com versão imediatamente seguinte à maior versão vista anteriormente.


### Retornos

Valores não encontrados são deixados "em branco", isto é, deve-se retornar "" (string vazia) ou 0 (se inteiro) para valores não encontrados.

Exceções (erros de comunicação, formato dos dados, etc), devem ser tratadas no lado servidor para evitar perda do estado durante os testes.


## Etapa Única

Esta etapa consiste na implementação da lógica de do servidor, com comunicação RPC entre clientes e servidores, e comunicação por meio de arquitetura *Publish-Subscribe* entre os servidores.
O cliente já é fornecido no projeto base e deve ser utilizado para desenvolvimento do servidor.
Todos os testes serão baseados neste cliente fornecido.

### Comunicação entre servidores

O suporte a múltiplos servidores deve garantir que clientes possam se conectar a instâncias diferentes de servidores e ainda sim sejam capazes de manipular os dados armazenados.
Por exemplo, o cliente *c1* pode cadastrar uma chave/valor servidor *s1* e deve ser capaz de recuperar o valor inserido para a mesma chave a partir de um segundo servidor *s2*.

Para isto cada servidor deve publicar qualquer alteração nas chaves em um broker pub-sub, em tópico conhecido pelos demais, a partir do qual estes receberão as mudanças de estado e atualizarão suas próprias tabelas.

Os dados publicados no broker pub-sub devem, **obrigatoriamente**, utilizar o formato JSON, com detalhamento do campos escolhidos na documentação do projeto.

A figura a seguir ilustra a arquitetura exigida para esta etapa do Projeto.

![Projeto](drawings/projeto.drawio#3)

### Requisitos básicos

* Organizar-se em grupos de, **obrigatoriamente**, 2 alunos.
    * Não serão aceitas entregas individuais!
* Utilizar, **obrigatoriamente**, gRPC para comunicação entre clientes e servidores.
* Utilizar, **obrigatoriamente**, MQTT para atualização de estado entre servidores.
* Implementar os casos de uso usando tabelas hash locais aos servidores, em memória (hash tables, dicionários, mapas, etc).
* Implementar servidor com interface de linha de comando para execução.
* Certificar-se de que todas as APIs possam retornar erros/exceções e que estas são tratadas, explicando sua decisão de tratamento dos erros.
* Documentar o esquema de dados usados nas tabelas.
* Suportar a execução de múltiplos clientes e servidores.
* Implementar a propagação de informação entre servidores usando necessariamente *pub-sub*, já que a comunicação é de 1 para muitos.
    * Utilizar o *broker pub-sub* [`mosquitto`](../cases/mosquitto) com a configuração padrão e aceitando conexões na interface local (*localhost ou 127.0.0.1*), porta TCP 1883.
* Gravar um vídeo de no máximo 5 minutos demonstrando que os requisitos foram atendidos.

### Submissão

* A submissão será feita até a data limite via formulário do Microsoft Teams, bastando informar o link do repositório **privado** em *github.com*, devidamente compartilhado com o usuário `paulo-coelho`.
* Os 2 integrantes do grupo devem fazer a entrega, para facilitar o retorno com a nota e comentários.
* O repositório privado no *github* deve conter no mínimo:
    * Arquivo `README.md` com:
      * Instruções de compilação
      * Detalhes de instalação e configuração da linguagem de programação
      * Uso do servidor
      * Organização dos dados com indicação dos formato da(s) tabela(s) hash utilizada(s)
      * Descrição das dificuldades
      * Indicação dos requisitos não implementados
    * Arquivo `compile.sh` para baixar/instalar dependências, compilar e gerar binários.
    * Arquivo `server.sh` para executar o servidor, recebendo como parâmetro único a porta em que o servidor deve aguardar conexões.
    * Arquivo `client.sh` (fornecido) para executar o cliente interativo.

## Linguagens aceitas

**APENAS** serão aceitas estas linguagens:

* Java
* C
* Python
* Go
* Rust

Trabalhos em outra linguagem não serão corrigidos e receberão nota zero!


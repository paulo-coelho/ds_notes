A área de computação distribuída é rica em aplicações e desenvolvê-los é topar de frente com vários problemas e decidir como resolvê-los ou contorná-los e, por isto, nada melhor que um projeto para experimentar em primeira mão as angústias e prazeres da área. 
Assim, proponho visitarmos o material destas notas à luz de uma aplicação genérica mas real, desenvolvida por vocês enquanto vemos a teoria.

O projeto consiste em implementar um **Sistema de Matrícula** com armazenamento  chave-valor (*key-value store = KVS*).

A arquitetura do sistema será **híbrida**, contendo um pouco de cliente/servidor, publish/subscribe e Peer-2-Peer, além de ser multicamadas.
Apesar de introduzir complexidade extra, também usaremos **múltiplos mecanismos para a comunicação** entre as partes, para que possam experimentar com diversas abordagens.

O sistema contempla dois portais: **Portal Administrativo** e **Portal de Matrícula**.
O Portal Administrativo é responsável por manter os cadastros de alunos, professores e disciplinas.
O Portal de Matrícula gerencia a atribuição de disciplinas a professores e o cadastro de alunos em matrículas.

O sistema utiliza a arquitetura cliente-servidor na comunicação entre clientes, que realiza as operações em cima do sistema de armazenamento, e servidores, que atualizam o estado de acordo e retornam o resultado a cada cliente.

Múltiplas instâncias do servidor podem ser executados simultaneamente para aumentar a disponibilidade do serviço e/ou atender um maior número de clientes (escalar).
O detalhamento do comportamento e implementação, neste caso, será diferente para cada etapa do projeto, sendo detalhado nas seções seguintes.

Tanto as aplicações clientes quanto os servidores devem, obrigatoriamente, possuir uma interface de linha de comando (*command line interface - CLI*) para execução e interação.

A aplicação servidor manipula cada chave **K** em operações de escrita e leitura de acordo com a descrição e interface apresentadas mais adiante.
A tupla **(K,V)** significa que a chave **K** possui valor **V**.
Cada atualização para a mesma chave **K** com um novo valor **V'** substitui o valor **V** armazenado previamente.
As chaves e valores devem ser do tipo *String*.

Os dados de alunos, professores e disciplinas em cada Portal são mantidos em um tabela hash `ID -> Dados`, todos do tipo `String` e armazenados em memória (use uma tabela hash). 

O campo `Dados` **deve** ser armazenado como uma string JSON, com formato definido por você, devidamente descrito na documentação do projeto.

Além disso, você pode criar tabelas hash adicionais para representar a relação entre os dados. 
Por exemplo, pode-se criar a tabela "DisciplinaAluno" no Portal de Matrículas, cuja chave seja a sigla da disciplina e o valor seja um vetor de matrículas de alunos devidamente representado no formato JSON.

A comunicação entre clientes e servidores deve ser, **obrigatoriamente**, realizada via [gRPC](../cases/grpc) de acordo com a interface definida adiante.

O servidor do Portal de Matrícula deve obter os dados de alunos, professores e disciplinas a partir do servidor do Portal Administrativo.
Recomenda-se manter um "cache" dos dados no servidor do Portal de Matrículas para evitar chamdas repetidas para valores já conhecidos.

A comunicação entre estes servidores será detalhada na descrição da etapa correspondente nas seções a seguir.

A implementação que não seguir o formato dos dados, a interface ou a estratégia de comunicação definidos **não será avaliada e terá atribuída a nota zero**.


## Casos de Uso

### Interface

O formato exato em que os dados serão armazenados pode variar na sua implementação, mas a API apresentada deve, **obrigatoriamente**, ter a assinatura definida a seguir para cada aplicação:

### Portal Administrativo

```proto
syntax = "proto3";

option java_multiple_files = true;
option java_package = "br.ufu.facom.gbc074.projeto.cadastro";

package project;

message Aluno {
  string matricula = 1;
  string nome      = 2;
}

message Professor {
  string siape = 1;
  string nome  = 2;
}

message Disciplina {
  string sigla = 1;
  string nome  = 2;
  // total de alunos permitido
  int32 vagas  = 3;
}

message Status {
  // 0 = sucesso, 1 = erro
  int32 status = 1; 
  // detalhes do erro para status = 1
  string msg   = 2;
}

message Identificador {
  // matricula, siape ou sigla
  string id = 1;
}

message Alunos {
  // lista de alunos
  repeated Aluno alunos = 1;
}

message Professores{
  // lista de professores
  repeated Professor professores = 1;
}

message Disciplinas {
  // lista de alunos
  repeated Disciplina disciplinas = 1;
}

message Vazia {}

service PortalAdministrativo {
  rpc NovoAluno(Aluno) returns (Status) {}
  rpc EditaAluno(Aluno) returns (Status) {}
  rpc RemoveAluno(Identificador) returns (Status) {}
  rpc ObtemAluno(Identificador) returns (Aluno) {}
  rpc ObtemTodosAlunos(Vazia) returns (Alunos) {}
  rpc NovoProfessor(Professor) returns (Status) {}
  rpc EditaProfessor(Professor) returns (Status) {}
  rpc RemoveProfessor(Identificador) returns (Status) {}
  rpc ObtemProfessor(Identificador) returns (Professor) {}
  rpc ObtemTodosProfessores(Vazia) returns (Professores) {}
  rpc NovaDisciplina(Disciplina) returns (Status) {}
  rpc EditaDisciplina(Disciplina) returns (Status) {}
  rpc RemoveDisciplina(Identificador) returns (Status) {}
  rpc ObtemDisciplina(Identificador) returns (Disciplina) {}
  rpc ObtemTodasDisciplinas(Vazia) returns (Disciplinas) {}
}
```

#### Descrição dos métodos

* `rpc NovoAluno(Aluno) returns (Status) {}`
    * Cliente:
        - informa matricula e nome do aluno a ser cadastrado.
    * Servidor:
        - cadastra o novo aluno e retorna 0 se aluno não existia previamente e a matrícula e o nome possuem tamanho maior do que 4.
        - retorna 1 com a descrição do erro, caso contrário.
* `rpc EditaAluno(Aluno) returns (Status) {}`
    * Cliente:
        - informa matricula e nome do aluno a ser atualizado.
    * Servidor:
        - atualiza o aluno e retorna 0 se matrícula informada já existia previamente e o novo nome possui tamanho maior do que 4.
        - retorna 1 com a descrição do erro, caso contrário.
* `rpc RemoveAluno(Identificador) returns (Status) {}`
    * Cliente:
        - informa matricula do aluno a ser removido.
    * Servidor:
        - remove o aluno e retorna 0 se aluno já existia previamente.
        - retorna 1 com a descrição do erro, caso contrário.
* `rpc ObtemAluno(Identificador) returns (Aluno) {}`
    * Cliente:
        - informa matricula do aluno.
    * Servidor:
        - retorna dados do aluno solicitado, caso ele exista.
        - retorna Aluno com matrícula e nome em branco, caso contrário.
* `rpc ObtemTodosAlunos(Vazia) returns (Alunos) {}`
    * Cliente:
        - invoca método sem argumentos\*
    * Servidor:
        - retorna lista de todos os alunos cadastrados.

\* A linguagem de descrição de interface do *ProtoBuf* exige que o método tenha argumento, sendo definida uma mensagem sem atributos denominada `Vazia` para garantir conformidade.

A mesma lógica do cadastro de alunos se aplica aos cadastros de professores e disciplinas.

### Portal de Matrícula

```proto
syntax = "proto3";

option java_multiple_files = true;
option java_package = "br.ufu.facom.gbc074.projeto.matricula";

package project;

message Aluno {
  string matricula = 1;
  string nome      = 2;
}

message Professor {
  string siape = 1;
  string nome  = 2;
}

message Disciplina {
  string sigla = 1;
  string nome  = 2;
}

message RelatorioDisciplina {
  Disciplina disciplina          = 1;
  Professor professor            = 2;
  repeated Aluno alunos          = 3;
}

message ResumoDisciplina {
  Disciplina disciplina          = 1;
  Professor professor            = 2;
  int32 totalAlunos              = 3;
}

// Serve tanto para professor quanto para aluno
message RelatorioDisciplinaAlunoProfessor {
  // lista e disciplinas para o aluno ou professor
  repeated ResumoDisciplina disciplinas = 1;
}

message Status {
  // 0 = sucesso, 1 = erro
  int32 status = 1; 
  // detalhes do erro para status = 1
  string msg   = 2;
}

message Identificador {
  // matricula, siape ou sigla
  string id = 1;
}

message DisciplinaPessoa {
  // id da disciplina
  string disciplina = 1;
  // matricula do aluno ou siape do professor
  string idPessoa = 2;
}

service PortalAdministrativo {
  rpc AdicionaProfessor(DisciplinaPessoa) returns (Status) {}
  rpc RemoveProfessor(DisciplinaPessoa) returns (Status) {}
  rpc AdicionaAluno(DisciplinaPessoa) returns (Status) {}
  rpc RemoveAluno(DisciplinaPessoa) returns (Status) {}
  rpc DetalhaDisciplina(Identificador) returns (RelatorioDisciplina) {}
  rpc ObtemDisciplinasProfessor(Identificador) returns (RelatorioDisciplinaAlunoProfessor) {}
  rpc ObtemDisciplinasAluno(Identificador) returns (RelatorioDisciplinaAlunoProfessor) {}
}

```

#### Descrição dos métodos

* `rpc AdicionaProfessor(DisciplinaPessoa) returns (Status) {}`
    * Cliente:
        - informa a sigla da disciplina e o siape do professor a ser associado à disciplina.
    * Servidor:
        - cadastra o professor na disciplina e retorna 0 se disciplina e professor existem, e a disciplina não tem nenhum professor previamente cadastrado.
        - retorna 1 com a descrição do erro, caso contrário.
* `rpc RemoveProfessor(DisciplinaPessoa) returns (Status) {}`
    * Cliente:
        - informa a sigla da disciplina e o siape do professor a ser desassociado da disciplina.
    * Servidor:
        - remove o professor da disciplina e retorna 0 se disciplina e professor existem, e a o professor estava associado à disciplina.
        - retorna 1 com a descrição do erro, caso contrário.
* `rpc AdicionaAluno(DisciplinaPessoa) returns (Status) {}`
    * Cliente:
        - informa a sigla da disciplina e a matrícula do aluno a ser matriculado na disciplina.
    * Servidor:
        - cadastra o aluno na disciplina e retorna 0 se disciplina e aluno existem, e a disciplina ainda não atingiu o limite de vagas ou não contém o mesmo aluno.
        - retorna 1 com a descrição do erro, caso contrário.
* `rpc RemoveAluno(DisciplinaPessoa) returns (Status) {}`
    * Cliente:
        - informa a sigla da disciplina e a matrícula do aluno a ser desassociado da disciplina.
    * Servidor:
        - remove o aluno da disciplina e retorna 0 se disciplina e disciplina existem, e a o aluno estava associado à disciplina.
        - retorna 1 com a descrição do erro, caso contrário.
* `rpc DetalhaDisciplina(Identificador) returns (RelatorioDisciplina) {}`
    * Cliente:
        - informa a sigla da disciplina.
    * Servidor:
        - retorna relatório para a disciplina solicitada, caso ela exista.
        - retorna relatório em branco, caso contrário.
* `rpc ObtemDisciplinasProfessor(Identificador) returns (RelatorioDisciplinaProfessor) {}`
    * Cliente:
        - informa o siape do professor.
    * Servidor:
        - retorna relatório com as disciplinas associadas ao professor, caso o professor existam e o mesmo esteja associado a alguma disciplina.
        - retorna relatório em branco, caso contrário.
* `rpc ObtemDisciplinasAluno(Identificador) returns (RelatorioDisciplinaAluno) {}`
    * Cliente:
        - informa a matrícula do aluno.
    * Servidor:
        - retorna relatório com as disciplinas associadas ao aluno, caso o aluno exista e a mesmo esteja matriculado em alguma disciplina.
        - retorna relatório em branco, caso contrário.

### Retornos

Valores não encontrados são deixados "em branco", isto é, deve-se retornar "" (string vazia) ou 0 (se inteiro) para valores não encontrados.

Exceções (erros de comunicação, formato dos dados, etc), devem ser tratadas ao menos no lado servidor para evitar perda do estado durante os testes.


## Etapa 1 

Esta etapa consiste na implementação da lógica de cadastros e consultas nos portais, com comunicação RPC entre clientes e servidores, e comunicação por meio de arquitetura *Publish-Subscribe* entre os servidores.


### Comunicação entre servidores

O suporte a múltiplos servidores deve garantir que clientes possam se conectar a instâncias diferentes de servidores e ainda sim sejam capazes de manipular os dados armazenados.
Por exemplo, o cliente *c1* pode cadastrar um aluno no servidor *s1* e deve ser capaz de recuperar o valor inserido para a mesma matrícula a partir de um segundo servidor *s2*.

Para isto, nesta etapa, cada servidor deve publicar qualquer alteração nas chaves em um broker pub-sub, em tópico conhecido pelos demais, a partir do qual estes receberão as mudanças de estado e atualizarão suas próprias tabelas.

Os dados publicados no broker pub-sub devem, **obrigatoriamente**, utilizar o formato JSON, com exemplos na documentação do projeto.


A figura a seguir ilustra a arquitetura exigida para a Etapa 1 do Projeto.

![Projeto](drawings/projeto.drawio#3)

### Requisitos básicos

* Organizar-se em grupos de, **obrigatoriamente**, 3 alunos.
* Implementar os casos de uso usando tabelas hash locais aos servidores, em memória (hash tables, dicionários, mapas, etc).
* Implementar os clientes e servidores com interface de linha de comando para execução.
* Certificar-se de que todas as API possam retornar erros/exceções e que estas são tratadas, explicando sua decisão de tratamento dos erros.
* Implementar testes automatizados de sucesso e falha de cada uma das operações na API.
* Documentar o esquema de dados usados nas tabelas.
* Suportar a execução de múltiplos clientes e servidores.
* Implementar a propagação de informação entre as diversas caches do sistema usando necessariamente *pub-sub*, já que a comunicação é de 1 para muitos.
    * Utilizar o *broker pub-sub* [`mosquitto`](../cases/mosquitto) com a configuração padrão e aceitando conexões na interface local (*localhost ou 127.0.0.1*), porta TCP 1883.
* Gravar um vídeo de no máximo 10 minutos demonstrando que os requisitos foram atendidos.
<!--

## Etapa 2 - Banco de dados Replicado
Nesta etapa você modificará o sistema para que atualizações dos dados sejam feitas persistente e consistentemente entre todas as réplicas usando um protocolo de difusão atômica.

![Projeto](drawings/projeto.drawio#1)

* Objetivos
    * Replicar a base de dados para obter tolerância a falhas.

* Desafios
    * Certificar-se de que os servidores são máquinas de estados determinística
    * Compreender o uso de Difusão Atômica em nível teórico
    * Compreender o uso de Difusão Atômica em nível prático
        * Use [Ratis](https://paulo-coelho.github.io/ds_notes/cases/ratis) para java
        * Para Python, [PySyncObj](https://github.com/bakwc/PySyncObj) é uma boa opção
        * Para Rust, [raft-rs](https://github.com/tikv/raft-rs) parece ser a biblioteca mais utilizada
        * Aplicar difusão atômica na replicação do banco de dados
        * Utilizar um banco de dados simples do tipo chave-valor, necessariamente [LevelDB](https://github.com/google/leveldb) ou [LMDB](https://git.openldap.org/openldap/openldap/tree/mdb.master)
            * Embora originalmente escritas em C++/C, há *ports* para diversas outras linguagens, (e.g., [aqui](https://github.com/lmdbjava/lmdbjava) e [aqui](https://github.com/dain/leveldb))
        * Utilizar três réplicas para o banco de dados
        * Não há limite para a quantidade de servidores acessados pelos clientes
* Implementação
    * A API para clientes e servidores continua a mesma
    * Requisições feitas pelos clientes via gRPC para o servidor (linha contínua) são encaminhadas via Ratis (linha tracejada) para ordená-las e entregar a todas as réplicas (linha pontilhada) para só então serem executadas e respondidas
    * Dados são armazenados em disco pela sua máquina de estado da aplicação via Ratis (DBi)
* Testes
    * O mesmo *framework* de testes deve continuar funcional
* Comunicação
    * Entre cliente e servidor nada é alterado
    * Entre servidores e réplicas do banco de dados, usar Ratis/PySyncOb
* Apresentação
    * Sem alteração, isto é, gravar um vídeo demonstrando que os requisitos foram atendidos.

-->

### Submissão

* A submissão será feita até a data limite via formulário do Microsoft Teams, bastando informar o link do repositório **privado** em *github.com*, devidamente compartilhado com o usuário `paulo-coelho`.
* O repositório privado no *github* deve conter no mínimo:
    * Arquivo `README.md` com instruções de compilação, inicialização e uso de clientes e servidores.
    * Arquivo `compile.sh` para baixar/instalar dependências, compilar e gerar binários.
    * Arquivo `admin-server.sh` para executar o servidor do Portal Administrativo, recebendo como parâmetro ao menos a porta em que o servidor deve aguardar conexões.
    * Arquivo `admin-client.sh` para executar o cliente do Portal Administrativo, recebendo como parâmetro ao menos a porta do servidor que deve se conectar.
    * Arquivo `mat-server.sh` para executar o servidor do Portal de Matrícula, recebendo como parâmetro ao menos a porta em que o servidor deve aguardar conexões.
    * Arquivo `mat-client.sh` para executar o cliente do Portal de Matrícula, recebendo como parâmetro ao menos a porta do servidor que deve se conectar.
    * Descrição das dificuldades com indicação do que não foi implementado.

<!--
    * Arquivo `replica.sh` para executar a réplica do banco de dados, recebendo como parâmetro o parâmetro *bd1*, *bd2* ou *bd3*, representado cada uma das réplicas da figura
-->

## Linguagens aceitas

**APENAS** serão aceitas estas linguagens:

* Java
* C
* Python
* Go
* Rust

Trabalhos em outra linguagem não serão corrigidos e receberão nota zero!

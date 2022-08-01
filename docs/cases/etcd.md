# Key-value store: Etcd

[Etcd](https://etcd.io) é uma infraestrutura de armazenamento do tipo chave-valor que fornece consistência forte. Implementa internamente o protocolo de consenso Raft para garantir tolerância a falhas.

Entre os recursos disponíveis estão a facilidade de execução de leituras e escritas, seja via HTTP (com *curl*, por exemplo) seja via bibliotecas e troca de mensagens no formato JSON.

O armazenamento de dados é organizado de maneira hierárquica em diretórios, similar a um sistema de arquivos.
Suporta SSL e monitoramento de chaves, fornece eleição de líder *builtin*, entre outros recursos.

É utilizado, por exemplo, para prover [armazenamento de alta disponibilidade para todos os dados de clusters Kubernetes](https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/).

A seguir veremos um passo-a-passo de como levantar os servidores e implementar o cliente.

## Instalação 

* Descarregar último *release* disponível para seu sistema operacional no [github](https://github.com/etcd-io/etcd/releases), descompactar em local adequado e definir o `PATH` corretamente.

### Execução

* Abra 3 terminais e execute cada réplica:

    * Réplica 1
```bash
# make sure etcd process has write access to this directory
# remove this directory if the cluster is new; keep if restarting etcd
# rm -rf /tmp/etcd/s1


/tmp/etcd-download-test/etcd --name s1 \
  --data-dir /tmp/etcd/s1 \
  --listen-client-urls http://localhost:12379 \
  --advertise-client-urls http://localhost:12379 \
  --listen-peer-urls http://localhost:12380 \
  --initial-advertise-peer-urls http://localhost:12380 \
  --initial-cluster s1=http://localhost:12380,s2=http://localhost:22380,s3=http://localhost:32380 \
  --initial-cluster-token tkn \
  --initial-cluster-state new
```

    * Réplica 2
```bash
# make sure etcd process has write access to this directory
# remove this directory if the cluster is new; keep if restarting etcd
# rm -rf /tmp/etcd/s2


/tmp/etcd-download-test/etcd --name s2 \
  --data-dir /tmp/etcd/s2 \
  --listen-client-urls http://localhost:22379 \
  --advertise-client-urls http://localhost:22379 \
  --listen-peer-urls http://localhost:22380 \
  --initial-advertise-peer-urls http://localhost:22380 \
  --initial-cluster s1=http://localhost:12380,s2=http://localhost:22380,s3=http://localhost:32380 \
  --initial-cluster-token tkn \
  --initial-cluster-state new
```

    * Réplica 3
```bash
# make sure etcd process has write access to this directory
# remove this directory if the cluster is new; keep if restarting etcd
# rm -rf /tmp/etcd/s3


/tmp/etcd-download-test/etcd --name s3 \
  --data-dir /tmp/etcd/s3 \
  --listen-client-urls http://localhost:32379 \
  --advertise-client-urls http://localhost:32379 \
  --listen-peer-urls http://localhost:32380 \
  --initial-advertise-peer-urls http://localhost:32380 \
  --initial-cluster s1=http://localhost:12380,s2=http://localhost:22380,s3=http://localhost:32380 \
  --initial-cluster-token tkn \
  --initial-cluster-state new
```

* Para verificar se as réplicas estão funcionando corretamente, execute:
```bash
ETCDCTL_API=3 /tmp/etcd-download-test/etcdctl \
  --endpoints localhost:12379,localhost:22379,localhost:32379 \
  endpoint health
```

Mais detalhes de instalação [aqui](http://play.etcd.io/home).

## Cliente

Bibliotecas [aqui](https://etcd.io/docs/v3.5/integrations/).

### Em Java

Crie um novo projeto Maven com o nome `ClienteEtcd` (eu estou usando IntelliJ, mas as instruções devem ser semelhantes para Eclipse).

![Novo Projeto Maven](../images/newmaven.png)

Abra o arquivo `pom.xml` do seu projeto e adicione o seguinte trecho, com as dependências do projeto, incluindo o próprio [jetcd](https://github.com/etcd-io/jetcd).

```xml
<dependencies>
    <dependency>
        <groupId>io.etcd</groupId>
            <artifactId>jetcd-core</artifactId>
        <version>0.7.1</version>
    </dependency>
     <dependency>
        <groupId>org.slf4j</groupId>
            <artifactId>slf4j-api</artifactId>
        <version>1.7.36</version>
     </dependency>
     <dependency>
         <groupId>org.slf4j</groupId>
             <artifactId>slf4j-simple</artifactId>
         <version>1.7.36</version>
     </dependency>
</dependencies>
```

Adicione também o plugin Maven e o plugin para gerar um `.jar` com todas as dependências. 

```xml
<build>
    <plugins>
        <plugin>
            <artifactId>maven-assembly-plugin</artifactId>
            <executions>
                <execution>
                    <phase>package</phase>
                    <goals>
                        <goal>single</goal>
                    </goals>
                </execution>
            </executions>
            <configuration>
                <descriptorRefs>
                    <descriptorRef>jar-with-dependencies</descriptorRef>
                </descriptorRefs>
            </configuration>
        </plugin>
    </plugins>
</build>
```

Crie uma nova classe denominada `ClienteEtcd` no arquivo `ClienteEtcd.java`.
Nesta classe, iremos criar um objeto `KV` que será usado para enviar operações para os servidores. 
Esta classe é importada juntamente com outras várias dependências, adicionadas no `pom.xml`, que devemos instanciar antes do `KV`.

Neste exemplo eu coloco praticamente todos os parâmetros de configuração do Etcd *hardcoded* para simplificar o código.
Obviamente que voce deveria ler estes parâmetros como argumentos para o programa ou de um arquivo de configuração.

```java
import io.etcd.jetcd.ByteSequence;
import io.etcd.jetcd.Client;
import io.etcd.jetcd.KV;
import io.etcd.jetcd.KeyValue;
import io.etcd.jetcd.kv.GetResponse;
import java.util.concurrent.CompletableFuture;

public class ClienteEtcd {
  public static void main(String args[]) throws Exception {
    // create client using target which enable using any name resolution mechanism provided
    // by grpc-java (i.e. dns:///foo.bar.com:2379)
    Client client =
        Client.builder().target("ip:///127.0.0.1:12379,127.0.0.1:22379,127.0.0.1:32379").build();

    KV kvClient = client.getKVClient();
    ByteSequence key = ByteSequence.from("test_key".getBytes());
    ByteSequence value = ByteSequence.from("test_value".getBytes());

    // put the key-value
    kvClient.put(key, value).get();

    // get the CompletableFuture
    CompletableFuture<GetResponse> getFuture = kvClient.get(key);

    // get the value from CompletableFuture
    GetResponse response = getFuture.get();

    for (KeyValue kv : response.getKvs()) {
      System.out.println("key = " + kv.getKey() + ", value = " + kv.getValue());
    }

    // delete the key
    kvClient.delete(key).get();

    // get the CompletableFuture
    getFuture = kvClient.get(key);

    // get the value from CompletableFuture
    response = getFuture.get();

    for (KeyValue kv : response.getKvs()) {
      System.out.println("key = " + kv.getKey() + ", value = " + kv.getValue());
    }

    client.close();
  }
}
```

Então, em um quarto terminal, execute:

```bash
java -cp target/ClienteEtcd-1.0-SNAPSHOT-jar-with-dependencies.jar ClienteEtcd
```

O código está disponível no Teams.

???todo "Exercício"
    * Ler operações, chaves e valores do terminal no formato `op:key:value`
    * Implementar as seguintes operações:
        * `get`
        * `put`
        * `del`
        * `update`


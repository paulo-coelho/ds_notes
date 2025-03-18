# RPC: Thrift

[Thrift](https://thrift.apache.org/)

## Instalação

* [Baixe](https://dlcdn.apache.org/thrift/) (a versão utilizada neste roteiro é a 0.16) e compile o thrift
* ou instale-o usando apt-get, por exemplo. `apt-get install thrift-compiler`
* execute "thrift" na linha de comando.
* Para thrift com Java, também precisarão dos seguintes arquivos
  * [slf4j-api-1.7.36](http://mvnrepository.com/artifact/org.slf4j/slf4j-api/1.7.36)
  * [libthrift-0.16.0.jar](https://mvnrepository.com/artifact/org.apache.thrift/libthrift/0.16.0)
  * [javax.annotation-api-1.3.2.jar](https://mvnrepository.com/artifact/javax.annotation/javax.annotation-api/1.3.2)
  * coloque-os na pasta `jars`
* Baseado no tutorial [oficial](https://thrift.apache.org/tutorial/java.html).

## IDL Thrift

* Serviços
```thrift
service ChaveValor {
    void set(1:i32 key, 2:string value),
    string get(1:i32 key) throws (1:KeyNotFound knf),
    void delete(1:i32 key)
}
```
* **Não se pode retornar NULL!!!**
* Exceções
```thrift
exception KeyNotFound {
   1:i64 hora,
   2:string chaveProcurada="thrifty"
}
```

Exemplo: chavevalor.thrift

```Thrift
namespace java chavevalor
namespace py chavevalor

exception KeyNotFound
{
   1:i64 time,
   2:i32 key
}

service ChaveValor
{
    string getKV(1:i32 key) throws (1:KeyNotFound knf),
    bool setKV(1:i32 key, 2:string value),
    void delKV(1:i32 key)
}
``` 

Compilação

`thrift --gen java chavevalor.thrift`

`thrift --gen py chavevalor.thrift`

ChaveValorHandler.java
```Java
package chavevalor;

import org.apache.thrift.TException;
import java.util.HashMap;
import chavevalor.*;

public class ChaveValorHandler implements ChaveValor.Iface {
   private HashMap<Integer,String> kv = new HashMap<>();
   @Override
   public String getKV(int key) throws TException {
       if(kv.containsKey(key))
          return kv.get(key);
       else
          throw new KeyNotFound();
   }
   @Override
   public boolean setKV(int key, String valor) throws TException {
       kv.put(key,valor);
       return true;
   }
   @Override
   public void delKV(int key) throws TException {
       kv.remove(key);
   }    
}
```

## Arquitetura 

* Runtime library -- componentes podem ser selecionados em tempo de execução e implementações podem ser trocadas
* Protocol -- responsável pela serialização dos dados
  * TBinaryProtocol
  * TJSONProtocol
  * TDebugProtocol
  * ...
* Transport -- I/O no ``fio''
  * TSocket
  * TFramedTransport (non-blocking server)
  * TFileTransport
  * TMemoryTransport
* Processor -- Conecta protocolos de entrada e saída com o \emph{handler}

* Handler -- Implementação das operações oferecidas
* Server -- Escuta portas e repassa dados (protocolo) para o processors
  * TSimpleServer
  * TThreadPool
  * TNonBlockingChannel


## Execução

```bash
javac  -cp jars/libthrift0.16.0.jar:jars/slf4japi1.7.36.jar:gen-java  -d . *.java 
java -cp jars/libthrift0.16.0.jar:jars/slf4japi1.7.36.jar:gen-java:. chavevalor.ChaveValorServer
java -cp jars/libthrift0.16.0.jar:jars/slf4japi1.7.36.jar:gen-java:. chavevalor.ChaveValorClient
```


!!! question "Exercícios - Thrift"
    * Usando o *handler* fornecido no threcho acima, implemente o servidor em java
    * Implemente o cliente em Java 
    * Execute servidor e cliente e teste a invocação dos métodos remotos implementados
    * [EXTRA] Implemente o cliente em python e execute com o servidor em Java 


## Referências

* [Apache Thrift Whitepaper](https://thrift.apache.org/static/files/thrift-20070401.pdf)

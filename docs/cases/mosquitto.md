# Pub/Sub: MQTT e MosQuiTTo

MQTT é um protocolo de transporte para publish/subscribe do tipo cliente-servidor, definido pela OASIS, uma organização aberta responsável por padrões como SAML e DocBook. 
A especificação atual é a de número 5, lançada em março de 2019.
O protocolo é leve, aberto e fácil de implementar, ideal para comunicação *Machine to Machine* (M2M) e uso no contexto de Internet das Coisas (*Internet of Things - I0T*).

>MQTT is a very light weight and binary protocol, and due to its minimal packet overhead, MQTT excels when transferring data over the wire in comparison to protocols like HTTP. Another important aspect of the protocol is that MQTT is extremely easy to implement on the client side. Ease of use was a key concern in the development of MQTT and makes it a perfect fit for constrained devices with limited resources today.

O [Eclipse Mosquitto](https://mosquitto.org) é um *broker* de código livre que implementa o protocolo MQTT v5.0, v3.1.1 e v3.1.
Por ser mínimo, o Mosquitto é ideal para uso em dispositivos pequenos e com pouca capacidade energética, como computadores **low power single board**, mas flexível o suficiente para ser usado em aplicações de larga escala.




###### Instalação

* Ubuntu: `apt-get install mosquitto`
* MacOS: brew install mosquitto
* Windows: baixe [mosquitto-XXXX-install-windows-x64.exe](http://mosquitto.org/download)

###### Inicializando o serviço

O arquivo `mosquito.conf` contém as configurações para o *broker*. 
As configurações funcionam bem para o nosso caso. O *broker* aceita requisições na porta 1883 e *publishers* e *subscribers* também utilizam essa porta por padrão.
Basta iniciar o *broker* com a opção `-v` para ter mais detalhes sobre o que ocorre internamente.

* Ubuntu: `mosquitto -v`
* MacOS: `/usr/local/sbin/mosquitto -c /usr/local/etc/mosquitto/mosquitto.conf`


###### Publicando

Para publicar uma mensagem, o *publisher* deve indicar um host, porta, tópico e mensagem. Caso o host e porta sejam omitidos, assume-se `localhost:1883`.
No MacOS, adicione `/usr/local/opt/mosquitto/bin/mosquitto_sub` ao *path*.

```bash
# publicando valor de 40 para tópicos 'sensor/temperature/1' e 'sensor/temperature/2'
mosquitto_pub -t sensor/temperature/1 -m 40
mosquitto_pub -t sensor/temperature/2 -m 32
```

Caso o *subscriber* não esteja em execução, adicione a opção `-r` para que o broker retenha a mensagem.

###### Consumindo

O consumidor funciona de maneira semelhante, informando o tópico de interesse:

```bash
# consumindo mensagens de tópico /sensor/temperature/*
mosquitto_sub -t sensor/temperature/+
```

###### Programando

Existem também APIs em diversas linguagem para desenvolvimento de aplicações que utilizem o Mosquitto.
A biblioteca pode ser baixada [aqui](https://www.eclipse.org/paho/index.php?page=clients/java/index.php).


```java
package br.ufu.facom.gbc074.mqtt;

import org.eclipse.paho.client.mqttv3.MqttClient;
import org.eclipse.paho.client.mqttv3.MqttConnectOptions;
import org.eclipse.paho.client.mqttv3.MqttException;
import org.eclipse.paho.client.mqttv3.MqttMessage;
import org.eclipse.paho.client.mqttv3.persist.MemoryPersistence;

public class MqttPublishSample {

  public static void main(String[] args) {
    String topic = args.length > 0 ? args[0] : "sensor/temperature/1";
    String content = "100";
    int qos = 2;
    String broker = "tcp://broker.hivemq.com:1883";
    String clientId = "publisher-paulo-gbc074"; //mude para que seja único
    MemoryPersistence persistence = new MemoryPersistence();

    try {
      MqttClient sampleClient = new MqttClient(broker, clientId, persistence);
      MqttConnectOptions connOpts = new MqttConnectOptions();
      connOpts.setCleanSession(true);
      System.out.println("Connecting to broker: " + broker);
      sampleClient.connect(connOpts);
      System.out.println("Connected");
      System.out.println("Publishing message: " + content + " in topic: " + topic);
      MqttMessage message = new MqttMessage(content.getBytes());
      message.setQos(qos);
      sampleClient.publish(topic, message);
      System.out.println("Message published");
      sampleClient.disconnect();
      System.out.println("Disconnected");
      System.exit(0);

    } catch (MqttException me) {
      System.out.println("reason " + me.getReasonCode());
      System.out.println("msg " + me.getMessage());
      System.out.println("loc " + me.getLocalizedMessage());
      System.out.println("cause " + me.getCause());
      System.out.println("excep " + me);
      me.printStackTrace();
    }
  }
}
```

???todo "Hive"
    Em terminais distintos, execute:

     * `mosquitto_sub -h broker.hivemq.com  -p 1883 -t esportes/+/flamengo`
     * `mosquitto_pub -t esportes/nadacao/flamengo -m "perdeu mais uma vez" -r -h broker.hivemq.com`


<!-- Distinction from message queues
There is a lot of confusion about the name MQTT and whether the protocol is implemented as a message queue or not. We will try to shed some light on the topic and explain the differences. In our last post, we mentioned that MQTT refers to the MQseries product from IBM and has nothing to do with “message queue“. Regardless of where the name comes from, it’s useful to understand the differences between MQTT and a traditional message queue:

A message queue stores message until they are consumed When you use a message queue, each incoming message is stored in the queue until it is picked up by a client (often called a consumer). If no client picks up the message, the message remains stuck in the queue and waits to be consumed. In a message queue, it is not possible for a message not to be processed by any client, as it is in MQTT if nobody subscribes to a topic.

A message is only consumed by one client 
Another big difference is that in a traditional message queue a message can be processed by one consumer only. The load is distributed between all consumers for a queue. In MQTT the behavior is quite the opposite: every subscriber that subscribes to the topic gets the message.

Queues are named and must be created explicitly 
A queue is far more rigid than a topic. Before a queue can be used, the queue must be created explicitly with a separate command. Only after the queue is named and created is it possible to publish or consume messages. In contrast, MQTT topics are extremely flexible and can be created on the fly.

If you can think of any other differences that we overlooked, we would love to hear from you in the comments. -->


Na IDE IntelliJ, crie um novo projeto Maven, utilizando o arquivo *pom.xml* fornecido a seguir:

```xml
<?xml version="1.0" encoding="UTF-8"?>

<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <groupId>br.ufu.facom.gbc074.mqtt</groupId>
  <artifactId>mqtt-pubsub</artifactId>
  <version>1.0-SNAPSHOT</version>

  <name>mqtt-pubsub</name>
  <!-- FIXME change it to the project's website -->
  <url>http://www.example.com</url>

  <properties>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <maven.compiler.release>18</maven.compiler.release>
    <maven.compiler.source>18</maven.compiler.source>
    <maven.compiler.target>18</maven.compiler.target>
  </properties>

  <dependencies>
    <dependency>
      <groupId>junit</groupId>
      <artifactId>junit</artifactId>
      <version>4.11</version>
      <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.eclipse.paho</groupId>
        <artifactId>org.eclipse.paho.client.mqttv3</artifactId>
        <version>1.2.5</version>
    </dependency>
  </dependencies>

  <build>
    <plugins>
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-shade-plugin</artifactId>
        <version>3.3.0</version>
        <configuration>
          <!-- put your configurations here -->
        </configuration>
        <executions>
          <execution>
            <phase>package</phase>
            <goals>
              <goal>shade</goal>
            </goals>
          </execution>
        </executions>
      </plugin>
    </plugins>
  </build>
</project>

```


!!! question "Exercícios - RPC e Publish/Subscribe"
    * Usando o *broker* mosquitto instalado localmente (utilize broker.hivemq.com se não for possível instalar), faça em Java um *publisher* que simula um sensor de temperatura e publica valores aleatórios entre 15 e 45 a cada segundo.
    * Faça o *subscriber* que irá consumir esses dados de temperatura.
<!--
    * Usando *thrift* e a linguagem Java, estenda o serviço ChaveValor para retornar o valor antigo de uma determinada chave na operação `setKV()`  caso a chave já exista.
-->





## Referências
* [MQTT Essentials](https://www.hivemq.com/mqtt-essentials/)
* [MQTT Client in Java](https://www.baeldung.com/java-mqtt-client)
* [Thrift Tutorial](http://thrift-tutorial.readthedocs.org/en/latest/index.html)

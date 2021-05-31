## Introdução ao KafkaStreams em Java

# 1. Visão geral

Neste artigo, veremos a biblioteca KafkaStreams.

KafkaStreams é projetado pelos criadores do Apache Kafka. O principal objetivo deste software é permitir que os programadores criem aplicativos de streaming eficientes e em tempo real que possam funcionar como microsserviços.

KafkaStreams nos permite consumir de tópicos Kafka, analisar ou transformar dados e, potencialmente, enviá-los para outro tópico Kafka.

Para demonstrar KafkaStreams, criaremos um aplicativo simples que lê frases de um tópico, conta ocorrências de palavras e imprime a contagem por palavra.

É importante observar que a biblioteca KafkaStreams não é reativa e não tem suporte para operações assíncronas e tratamento de contrapressão.

# 2. Dependência Maven
Para começar a escrever lógica de processamento de fluxo usando KafkaStreams, precisamos adicionar uma dependência para kafka-streams e kafka-clients:

```
<dependency>
    <groupId>org.apache.kafka</groupId>
    <artifactId>kafka-streams</artifactId>
    <version>1.0.0</version>
</dependency>
<dependency>
    <groupId>org.apache.kafka</groupId>
    <artifactId>kafka-clients</artifactId>
    <version>1.0.0</version>
</dependency>
```

Também precisamos ter o Apache Kafka instalado e iniciado porque usaremos um tópico Kafka. Este tópico será a fonte de dados para nosso trabalho de streaming.

Podemos baixar o Kafka e outras dependências necessárias do site oficial.

# 3. Configurando a entrada KafkaStreams
A primeira coisa que faremos é a definição do tópico Kafka de entrada.

Podemos usar a ferramenta Confluent que baixamos - ela contém um servidor Kafka. Ele também contém o kafka-console-producer que podemos usar para publicar mensagens no Kafka.

Para começar, vamos executar nosso cluster Kafka:

```
./confluent start
```

Assim que o Kafka iniciar, podemos definir nossa fonte de dados e o nome de nosso aplicativo usando APPLICATION_ID_CONFIG:

```
String inputTopic = "inputTopic";
```

```
Properties streamsConfiguration = new Properties();
streamsConfiguration.put(
  StreamsConfig.APPLICATION_ID_CONFIG, 
  "wordcount-live-test");
```

Um parâmetro de configuração crucial é o BOOTSTRAP_SERVER_CONFIG. Este é o URL para nossa instância local do Kafka que acabamos de iniciar:

```
private String bootstrapServers = "localhost:9092";
streamsConfiguration.put(
  StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, 
  bootstrapServers);
```

Em seguida, precisamos passar o tipo de chave e o valor das mensagens que serão consumidas de inputTopic:

```
streamsConfiguration.put(
  StreamsConfig.DEFAULT_KEY_SERDE_CLASS_CONFIG, 
  Serdes.String().getClass().getName());
streamsConfiguration.put(
  StreamsConfig.DEFAULT_VALUE_SERDE_CLASS_CONFIG, 
  Serdes.String().getClass().getName());
```

O processamento de stream geralmente é monitorador de estado. Quando queremos salvar resultados intermediários, precisamos especificar o parâmetro STATE_DIR_CONFIG.

Em nosso teste, estamos usando um sistema de arquivos local:

```
streamsConfiguration.put(
  StreamsConfig.STATE_DIR_CONFIG, 
  TestUtils.tempDirectory().getAbsolutePath());
```

# 4. Construindo uma Topologia de Streaming
Depois de definir nosso tópico de entrada, podemos criar uma Topologia de fluxo - que é uma definição de como os eventos devem ser tratados e transformados.

Em nosso exemplo, gostaríamos de implementar um contador de palavras. Para cada frase enviada para inputTopic, queremos dividi-la em palavras e calcular a ocorrência de cada palavra.

Podemos usar uma instância da classe KStreamsBuilder para começar a construir nossa topologia:

```
KStreamBuilder builder = new KStreamBuilder();
KStream<String, String> textLines = builder.stream(inputTopic);
Pattern pattern = Pattern.compile("\\W+", Pattern.UNICODE_CHARACTER_CLASS);

KTable<String, Long> wordCounts = textLines
  .flatMapValues(value -> Arrays.asList(pattern.split(value.toLowerCase())))
  .groupBy((key, word) -> word)
  .count();
```

Para implementar a contagem de palavras, em primeiro lugar, precisamos dividir os valores usando a expressão regular.

O método de divisão está retornando uma matriz. Estamos usando flatMapValues() para achatá-lo. Do contrário, acabaríamos com uma lista de arrays e seria inconveniente escrever código usando essa estrutura.

Por fim, estamos agregando os valores de cada palavra e chamando a contagem() que calculará as ocorrências de uma palavra específica.

5. Tratamento de resultados
Já calculamos a contagem de palavras de nossas mensagens de entrada. Agora vamos imprimir os resultados na saída padrão usando o método foreach():

```
wordCounts
  .foreach((w, c) -> System.out.println("word: " + w + " -> " + c));
```

Na produção, muitas vezes, esse trabalho de streaming pode publicar a saída em outro tópico Kafka.

Podemos fazer isso usando o método to():

```
String outputTopic = "outputTopic";
Serde<String> stringSerde = Serdes.String();
Serde<Long> longSerde = Serdes.Long();
wordCounts.to(stringSerde, longSerde, outputTopic);
```

A classe Serde nos fornece serializadores pré-configurados para tipos Java que serão usados para serializar objetos em uma matriz de bytes. A matriz de bytes será então enviada ao tópico Kafka.

Estamos usando String como uma chave para nosso tópico e Long como um valor para a contagem real. O método to() salvará os dados resultantes em outputTopic.

# 6. Iniciando o trabalho KafkaStream
Até este ponto, construímos uma topologia que pode ser executada. No entanto, o trabalho ainda não começou.

Precisamos iniciar nosso trabalho explicitamente chamando o método start() na instância KafkaStreams:

```
KafkaStreams streams = new KafkaStreams(builder, streamsConfiguration);
streams.start();

Thread.sleep(30000);
streams.close();
```

Observe que estamos aguardando 30 segundos para que o trabalho seja concluído. Em um cenário do mundo real, esse trabalho estaria em execução o tempo todo, processando eventos de Kafka à medida que chegam.

Podemos testar nosso trabalho publicando alguns eventos em nosso tópico Kafka.

Vamos iniciar um produtor kafka-console e enviar manualmente alguns eventos para o nosso inputTopic:

```
./kafka-console-producer --topic inputTopic --broker-list localhost:9092
>"this is a pony"
>"this is a horse and pony"
```

Assim, publicamos dois eventos para o Kafka. Nosso aplicativo consumirá esses eventos e imprimirá a seguinte saída:

```
word:  -> 1
word: this -> 1
word: is -> 1
word: a -> 1
word: pony -> 1
word:  -> 2
word: this -> 2
word: is -> 2
word: a -> 2
word: horse -> 1
word: and -> 1
word: pony -> 2
```

Podemos ver que quando a primeira mensagem chegou, a palavra pônei ocorreu apenas uma vez. Mas quando enviamos a segunda mensagem, a palavra pônei aconteceu pela segunda vez imprimindo: “palavra: pônei -> 2 ″.

# 6. Conclusão
Este artigo discute como criar um aplicativo de processamento de fluxo primário usando Apache Kafka como fonte de dados e a biblioteca KafkaStreams como biblioteca de processamento de fluxo.
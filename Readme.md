## Descrição
O objetivo é criar uma solução de processamento de dados em tempo real usando Apache Spark Streaming e Apache Airflow, que irá consumir dados em tempo real a partir de um Data Lake. Esse Data Lake deve conter dados estruturados ou semi-estruturados armazenados em bancos de dados PostgreSQL ou MongoDB, além de arquivos JSON e CSV no sistema de arquivos local. As aplicações Spark serão desenvolvidas em pySpark e devem consumir os dados em tempo real do Data Lake para realizar transformações e análises. O Apache Kafka será usado para ingestão e entrega de dados em tempo real para as aplicações Spark. A Figura 1 mostra a arquitetura sugerida para o projeto. O Apache Airflow será responsável por programar e monitorar os fluxos de ETL do projeto, garantindo que as tarefas de processamento de dados em tempo real sejam executadas de forma confiável e escalável.


## Objetivos
- Criar um Data Lake que combine várias fontes de dados para que as aplicações Apache Spark desenvolvidas possam utilizá-lo. A escolha do conjunto de dados (dataset) é livre.
- Desenvolver fluxos de processamento de dados em tempo real (streaming) com Apache Spark, consumindo dados das fontes disponíveis no Data Lake. O Apache Airflow será responsável por agendar e orquestrar esses processos.
- Realizar uma análise simples dos dados para mostrar o funcionamento dos pipelines criados, utilizando o Apache Airflow para automatizar a criação e entrega dessas análises.


# Resolução
## Preparação da infraestrutura
Para configurar a infraestrutura do projeto, foram utilizados containers Docker. A imagem do Airflow foi usada para iniciar seu container, além de mais três containers para o Hadoop/Spark (um para o master e dois para os slaves). O arquivo Docker Compose utilizado para subir o container do Spark está disponível neste link: <https://github.com/cmdviegas/hadoop-spark>.

Para reproduzir o ambiente, basta acessar as pastas correspondentes a cada ferramenta no repositório e seguir os comandos para criar os containers.

## Criação do Data Lake
Vamos usar o dataset do Kaggle intitulado "Top 1000 Global Tech Companies Dataset (2024)", especificamente o arquivo companies.csv. Você pode encontrá-lo neste link: <https://www.kaggle.com/datasets/muhammadehsan000/top-1000-global-tech-companies-dataset-2024>.

Para atender aos requisitos do projeto, o dataset será dividido em duas partes: uma parte será mantida como arquivo CSV e a outra parte será armazenada no banco de dados PostgreSQL.

Para realizar essa divisão, usaremos o pySpark.

### Pyspark para manipular CSV
Com o cluster Spark em funcionamento, primeiro é necessário colocar o arquivo CSV no HDFS. Para isso seguimos os seguintes comandos:

1. Copiar o arquivo do computador local para o container Docker:
   bash
   docker cp companies.csv node-master:/home/spark
   

2. Copiar o arquivo do container para o HDFS:
   bash
   hdfs dfs -put companies.csv
   

Depois de colocar o CSV no sistema de arquivos, você pode começar a usar o Spark para manipulá-lo:

1. Importar o CSV para um DataFrame:
   python
   df = spark.read.csv("hdfs://node-master:9000/user/spark/companies.csv", header=True, inferSchema=True)
   

#### Separando as empresas pelo tipo de indústria
    df_software = df.filter(df["Industry"] == "Software—Application")

#### Salvando os dataframes em novos CSVs

    df_software.coalesce(1).write.csv("/user/spark/software", header=True, sep=";")

Como estamos utilizando o HDFS, o comando coalesce(1) é usado para garantir que o arquivo resultante esteja em uma única parte. Assim, o arquivo final será chamado de software.csv. Após salvar o CSV, vamos mover e renomear o arquivo para facilitar o tratamento dos caminhos.


### Armazenando as empresas
Para fazer isso, usaremos o driver JDBC assim como foi ensinado durante as aulas da disciplina para exportar o dataframe para o postgres.

    # Baixando o driver
    wget https://jdbc.postgresql.org/download/postgresql-42.6.0.jar

    # Iniciando o spark com o jar do driver
    pyspark --jars /home/spark/postgresql-42.6.0.jar

    # Importando o dataframe do csv preciamente salvo
    df_software = spark.read.csv("hdfs://node-master:9000/user/spark/companies.csv", header=True, inferSchema=True, sep=";")

    # Exportando o dataframe para o banco de dados
    df_software.write.format("jdbc").option("url","jdbc:postgresql://172.30.0.254:5432/").option("dbtable","software").option("user","postgres").option("password","spark").option("driver","org.postgresql.Driver").save()

## Instalação e configuração do Kafka
Para instalar o Kafka, utilizaremos o passo a passo também disponibilizado pelo professor durante as aulas. O arquivo para instalar o mesmo vai estar disponícel no repositório como: 7_Primeiros_exemplos_-_Kafka.txt

### Configuração do kafka

Para realizar a configuração do kafka, também seguimos o tutorial para configuração, porém vale ressaltar alguns comandos:

    # Mesmo depois de configurado, sempre que iniciar o node-master, precisamos usar os seguintes comandos para carregar o kafka

    # Configurar as variáveis de ambiente no .bashrc
    export KAFKA_HOME="/home/spark/kafka"
    export PATH="$PATH:$KAFKA_HOME/bin"

    # Carregar o .bashrc
    source ~/.bashrc

    # Comando para iniciar o servidor kafka
    kafka-server-start.sh $KAFKA_HOME/config/kraft/server.properties

    # Como criar um tópico
    kafka-topics.sh --create --topic meu-topico --bootstrap-server node-master:9092

### Debezium para monitorar um banco de dados
Uilizaremos o debezium junto com o kafka para monitorar o banco de dados postgres. Foi seguido o passo a passo do exemplo 3 para realizar essa parte do trabalho:

    # Importando funções
    from pyspark.sql.functions import *
    from pyspark.sql.types import *
    
    # Criar o dataframe do tipo stream, apontando para o servidor kafka e o tópico a ser consumido
    df = (spark.readStream
        .format("kafka")
        .option("kafka.bootstrap.servers", "node-master:9092")
        .option("subscribe", "meu-topico.public.minhatabela2")
        .option("startingOffsets", "earliest")
        .load()
    )

    # Definir o schema dos dados inseridos no tópico
    
    schema = StructType([
        StructField("payload", StructType([
            StructField("after", StructType([
                StructField("Ranking", IntegerType(), True),
                StructField("Company", StringType(), True),
                StructField("Market Cap", StringType(), True),
                StructField("Stock", StringType(), True),
                StructField("Country", StringType(), True),
                StructField("Sector", StringType(), True),
                StructField("Industry", StringType(), True)
            ]))
        ]))
    ])

    # Converter o valor dos dados do Kafka de formato binário para JSON usando a função from_json
    dx = df.select(from_json(df.value.cast("string"), schema).alias("data")).select("data.payload.after.*")

    # Realizar as transformações e operações desejadas no DataFrame 'df'
    # Neste exemplo, apenas vamos imprimir os dados em tela
    ds = (dx.writeStream 
        .outputMode("append") 
        .format("console")
        .option("truncate", False)
        .start()
    )

Com isso, o kafka vai continuar monitorando o banco de dados e qualquer dado inserido vai ser mostrado em tela se o comando anterior estiver sendo executado ainda.

Com isso, a parte de configuração do kafka foi feita.

## Airflow
Devido às dificuldades com a ferramenta, não conseguimos implementar o Airflow no nosso projeto.

## Engenharia de dados
Alunos: 
- Cristian Soares de Souza Chagas Filho 
- Pedro Rêgo Vilar Neto
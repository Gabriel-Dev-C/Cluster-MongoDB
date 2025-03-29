# Cluster MongoDB

> Repositório criado para documentar o processo de criação de um cluster de 4 nós no MongoDB utilizando o Docker Container.

Nesse arquivo irei listar e documentar o processo completo realizado em vídeo para que seja possível criar um sistema Replica Set funcional dentro do MongoDB.


<p align="center">
<img src="mongodb-capa.png" alt="MongoDB.">
</p>

## Ambiente de trabalho

Primeiro criamos um ambiente de trabalho favorável para a execução do projeto. Para isso devemos baixar e configurar o Docker Desktop e instalar o MongoDb Compass, para que seja mais fácil visualizar o processo.

## Docker Desktop

Já baixado e configurado, entramos no Docker e baixamos a imagem do MongoDB. Para isso, abrimos um terminal e rodamos o código:

``` docker pull mongodb/mongodb-community-server:latest ```

O próximo passo é criar uma rede Docker. Essa rede, chamada de "mongoCluster" permitirá que cada um dos contêineres em execução nessa rede vejam uns aos outros. Para criar uma rede, execute o comando:

``` docker network create mongoCluster ```

Com imagem baixada e network criado, podemos instânciar nossos contêiners, e para isso usamos o comando:

``` docker run -d --rm -p 27017:27017 --name mongo10 --network mongoCluster mongodb/mongodb-community-server:latest --replSet myReplicaSet --bind_ip localhost,mongo10 ```

Com esse comando criamos o primeiro contêiner na porta host 27017 com o nome "mongo10". Repetimos esse mesmo comando para os próximos 3 contêiners alterando apenas esses dois detalhes mencionados. Ficando assim:

``` docker run -d --rm -p 27018:27017 --name mongo20 --network mongoCluster mongodb/mongodb-community-server:latest --replSet myReplicaSet --bind_ip localhost,mongo20 ```

Terceiro contêiner:

``` docker run -d --rm -p 27019:27017 --name mongo30 --network mongoCluster mongodb/mongodb-community-server:latest --replSet myReplicaSet --bind_ip localhost,mongo30 ```

Quarto e último contêiner:

``` docker run -d --rm -p 27020:27017 --name mongo40 --network mongoCluster mongodb/mongodb-community-server:latest --replSet myReplicaSet --bind_ip localhost,mongo40 ```

O próximo passo é criar o conjunto de réplicas real com os quatro membros. Para isso utilizamos o comando:

``` docker exec -it mongo10 mongosh ```

Após isso, salvamos a string "Connection To" em um bloco de notas para que voce possa utilizar depois, na string de conexão do MongoDB Compass:

``` mongodb://127.0.0.1:27017/?directConnection=true&serverSelectionTimeoutMS=2000&appName=mongosh+2.3.8 ```

Verificamos se o container docker conectado está executando, com o comando:

``` db.runCommand ({hello:1}) ```

Agora, precisamos inicializar o nosso Replica Set para que seja feita a criação dos nós entre cada um dos contêiners criados. Para isso, usamos:

``` rs.initiate ({ _id: "myReplicaSet", members:[{_id:0, host: "mongo10"}, {_id:1, host: "mongo20"}, {_id:2, host: "mongo30"}, {_id:3, host: "mongo40"}]}) ```

Agora podemos sair do Contâiner Shell usando o comando:

``` exit ```

Para verificar se o conjunto de réplicas está em execução, podemos utilizar o comando:

``` docker exec -it mongo10 mongosh --eval "rs.status()" ```

## MongoDB Compass

Agora, dentro do MongoDB Compass, adicionamos uma nova conexão e inserimos a string anteriormente salva, assim iremos criar uma conexão direta com o nó primário do nosso Cluster.

``` mongodb://127.0.0.1:27017/?directConnection=true&serverSelectionTimeoutMS=2000&appName=mongosh+2.3.8 ```

Dentro do mongosh, podemos ver qual nó estamos conectados usando o comando:

``` rs.isMaster().primary ```

Agora podemos inserir dados através do nó primário usando o sistema corporativo:

```
use CorporeSystem

db.cliente.insertOne({codigo:1, nome: "Ana Maria"});
db.cliente.insertOne({codigo:2, nome: "Maria Jose"});
db.cliente.insertOne({codigo:3, nome: "Jose Silva"});
db.cliente.insertOne({codigo:4, nome: "Luis Souza"});
db.cliente.insertOne({codigo:5, nome: "Fernanda Silva"});

db.cliente.find()

```

Para verificação se o conjunto de réplicas está funcionando, iremos parar o contêiner primário com docker stop e tentar ler do seu banco de dados novamente:

``` docker stop mongo10 ```

Verificamos novamente o estado do Cluster:

``` docker exec -it mongo20 mongosh --eval "rs.status()" ```

Se tentarmos dar um novo insert na mesma seção aberta anteriormente, com esse comando:
``` db.cliente.insertOne({codigo:6, nome: "Teresa Maria"}); ```
irá dar um erro.

Para corrigir, abrimos outra conexão com a porta do Cluster primário, de acordo com o que o MongoDB elegeu:

``` mongodb://127.0.0.1:27018/?directConnection=true&serverSelectionTimeoutMS=2000&appName=mongosh+2.3.8 ```

Usaremos o banco de dados CorporeSystem e faremos a consulta de um registro:

```
use CorporeSystem

db.cliente.findOne()
```

E então faremos uma nova inserção de dados para verificar se o cluster continuou ativo:

``` db.cliente.insertOne({codigo:7, nome: "Joao Cardoso"}); ```

Agora podemos voltar novamente o nó "mongo10" para reestabelecer uma conexão:

``` docker run -d --rm -p 27017:27017 --name mongo10 --network mongoCluster mongodb/mongodb-community-server:latest --replSet myReplicaSet --bind_ip localhost,mongo10 ```

E por fim podemos verificar novamente o estado atual do nosso Cluster:

``` docker exec -it mongo20 mongosh --eval "rs.status()" ```

Confirmando assim que o Cluster se manteve ativo mesmo com a queda do servidor primário.

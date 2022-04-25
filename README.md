# Guide-deployment-basic
Openshift deployment for voting app

Red Hat OpenShift Container Platform (RHOCP) est un projet opensource base sur la solution d'orchestration de contenru Kubernetes. Il manage un cluster de serveur permettant de manager des applications conteneurise dans des data-center public ou prive.

L'objectif de ce guide et de deployer une application comprenant un front-end, un backend-end et une database sur OCP4.

Pour ce guide, nous allons utiliser un simple server Node avec Express qui ecoute sur le port 8080.


## Result 

```javascript
var express = require('express'),
    async = require('async'),
    pg = require('pg'),
    { Pool } = require('pg'),
    path = require('path'),
    cookieParser = require('cookie-parser'),
    bodyParser = require('body-parser'),
    methodOverride = require('method-override'),
    app = express(),
    server = require('http').Server(app),
    io = require('socket.io')(server);

io.set('transports', ['polling']);

var port = process.env.PORT || 8080;

io.sockets.on('connection', function (socket) {

  socket.emit('message', { text : 'Welcome!' });

  socket.on('subscribe', function (data) {
    socket.join(data.channel);
  });
});

var pool = new pg.Pool({
  connectionString: 'postgres://postgres:postgres@db/postgres'
});

async.retry(
  {times: 1000, interval: 1000},
  function(callback) {
    pool.connect(function(err, client, done) {
      if (err) {
        console.error("Waiting for db");
      }
      callback(err, client);
    });
  },
  function(err, client) {
    if (err) {
      return console.error("Giving up");
    }
    console.log("Connected to db");
    getVotes(client);
  }
);

function getVotes(client) {
  client.query('SELECT vote, COUNT(id) AS count FROM votes GROUP BY vote', [], function(err, result) {
    if (err) {
      console.error("Error performing query: " + err);
    } else {
      var votes = collectVotesFromResult(result);
      io.sockets.emit("scores", JSON.stringify(votes));
    }

    setTimeout(function() {getVotes(client) }, 1000);
  });
}

function collectVotesFromResult(result) {
  var votes = {a: 0, b: 0};

  result.rows.forEach(function (row) {
    votes[row.vote] = parseInt(row.count);
  });

  return votes;
}

app.use(cookieParser());
app.use(bodyParser());
app.use(methodOverride('X-HTTP-Method-Override'));
app.use(function(req, res, next) {
  res.header("Access-Control-Allow-Origin", "*");
  res.header("Access-Control-Allow-Headers", "Origin, X-Requested-With, Content-Type, Accept");
  res.header("Access-Control-Allow-Methods", "PUT, GET, POST, DELETE, OPTIONS");
  next();
});

app.use(express.static(__dirname + '/views'));

app.get('/', function (req, res) {
  res.sendFile(path.resolve(__dirname + '/views/index.html'));
});

server.listen(port, function () {
  var port = server.address().port;
  console.log('App running on port ' + port);
});
```

## Containerize result

Pour deployer une application dans Openshift, nous avons besoin d'une image Docker de cette application. A cette etape nous conteneurisons la partie result de notra application. 

```Dockerfile
FROM node:10-slim
# add curl for healthcheck
RUN apt-get update \
    && apt-get install -y --no-install-recommends \
    curl \
    && rm -rf /var/lib/apt/lists/*
# Add Tini for proper init of signals
ENV TINI_VERSION v0.19.0
ADD https://github.com/krallin/tini/releases/download/${TINI_VERSION}/tini /tini
RUN chmod +x /tini
WORKDIR /app
RUN npm install -g nodemon
RUN useradd -u 1001 -r -g 0 -d /app -s /sbin/nologin \
    -c "Default Application User" default && \
    chown -R 1001:0 /app && \
    chmod -R g+rw /app
USER 1001
COPY src/package*.json ./
RUN npm ci \
 && npm cache clean --force
COPY src/. .
ENV PORT 8080
EXPOSE 8080
CMD ["/tini", "--", "node", "server.js"]
```

J'ai utilise ici l'image node du dockerhub. Identifiez vous a votre regisytry puis pour creer l'image Docker executer la commande suivante:

```shell
podman build -t quay.io/<your-quay-name>/result .
podman push quay.io/<your-quay-name>/result
```

## Deployment  et service result

Nous allons ensuite creer un deployment qui contiendra l'image utilise, le nombre de replicas etc. Dans le fichier result-deployment.yaml ecrivew :

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: result-deployment
  labels:
    app: result-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: result-pod
  template:
    metadata:
      name: result-pod
      labels:
        deployment: result-pod
spec:
    containers:
      - name: result
        image: quay.io/feven/result
        imagePullPolicy: IfNotPresent
        ports:
          - containerPort: 8080
            protocol: TCP
```
Nous indiquons a openshift qu'il doit creer un deployment nomme result-deployment qui a un replicas de pod appele result-pod. Result-pod est un conteneur base sur l'image backend qui est expose sur le port 8080. Pour creer ce deployment executer la commande suivante :

```shell
oc apply -f result-deployment.yaml
```

Nous avons cree le deployment mais nous n'avons pas de mpoyne de communique avec lui. Pour resourdre ce probleme nous allons devoir creer un service. Un service est une abstraction kubernetes qui assigne une addresse IP et un DNS unique a un set de pod. Notre fichier result-service.yaml doit ressembler a cela:

```yaml
kind: Service
apiVersion: v1
metadata:
  name: result-service
  namespace: voting-app
  labels:
    app: result-service
spec:
  ports:
    - name: 8080-tcp
      protocol: TCP
      port: 8080
      targetPort: 8080
  selector:
    deployment: result-deployment
  type: ClusterIP
```

Applique ce service avec la commande suivante et expose le a l'aide d'une route 
```shell
oc apply -f result-service.yaml
oc expose service/result-service
```


## Voter 

```python
from flask import Flask, render_template, request, make_response, g
from redis import Redis
import os
import socket
import random
import json
import logging

option_a = os.getenv('OPTION_A', "Cats")
option_b = os.getenv('OPTION_B', "Dogs")
hostname = socket.gethostname()

app = Flask(__name__)

gunicorn_error_logger = logging.getLogger('gunicorn.error')
app.logger.handlers.extend(gunicorn_error_logger.handlers)
app.logger.setLevel(logging.INFO)

def get_redis():
    if not hasattr(g, 'redis'):
        g.redis = Redis(host="redis", db=0, socket_timeout=5, password='redis')
    return g.redis

@app.route("/", methods=['POST','GET'])
def hello():
    voter_id = request.cookies.get('voter_id')
    if not voter_id:
        voter_id = hex(random.getrandbits(64))[2:-1]

    vote = None

    if request.method == 'POST':
        redis = get_redis()
        vote = request.form['vote']
        app.logger.info('Received vote for %s', vote)
        data = json.dumps({'voter_id': voter_id, 'vote': vote})
        redis.rpush('votes', data)

    resp = make_response(render_template(
        'index.html',
        option_a=option_a,
        option_b=option_b,
        hostname=hostname,
        vote=vote,
    ))
    resp.set_cookie('voter_id', voter_id)
    return resp


if __name__ == "__main__":
    app.run(host='0.0.0.0', port=8080, debug=True, threaded=True)
```

## Containerize voter

Ensuite nous allonrs conteneuriser notre application . Pour notre image front-end nous allons utiliser une image python. Nous installons les packages present dans le fichier requirements.txt et nous ajoutons notre code dans le fichier /app du conteneur. On expose ensuite notre port 8080.

```Dockerfile
# Using official python runtime base image
FROM python:3.9-slim

# add curl for healthcheck
RUN apt-get update \
    && apt-get install -y --no-install-recommends \
    curl \
    && rm -rf /var/lib/apt/lists/*

# Set the application directory
WORKDIR /app

ENV PATH="/app/.local/bin/:${PATH}"

RUN useradd -u 1001 -r -g 0 -d /app -s /sbin/nologin \
    -c "Default Application User" default && \
    chown -R 1001:0 /app && \
    chmod -R g+rw /app

USER 1001

# Install our requirements.txt
COPY src/requirements.txt /app/requirements.txt
RUN pip install -r requirements.txt

# Copy our code from the current folder to /app inside the container
COPY src/. .

# Make port 8080 available for links and/or publish
EXPOSE 8080

# Define our command to be run when launching the container
CMD ["gunicorn", "app:app", "-b", "0.0.0.0:8080", "--log-file", "-", "--access-logfile", "-", "--workers", "4", "--keep-alive", "0"]
```

Une fois conteneurise pousser votre image dans votre registry

```shell
podman build -t quay.io/<your-quay-name>/result .
podman push quay.io/<your-quay-name>/result
```

## Deployment  et service voter

Nous allons ensuite creer un deployment  et un service qui seront tres similaires a celui de notre image result:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: voter-deployment
  labels:
    app: voter-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: voter-pod
  template:
    metadata:
      name: voter-pod
      labels:
        deployment: voter-pod
spec:
    containers:
      - name: voter
        image: quay.io/feven/voter
        imagePullPolicy: IfNotPresent
        ports:
          - containerPort: 8080
            protocol: TCP
    securityContext:
      runAsUser: 1001
```
On notera qu'une section security context a ete ajoute pour que le pods soit execute par l'utilisateur 1001 conformement au Dockerfile

```shell
oc apply -f voter-deployment.yaml
```

Notre fichier voter-service.yaml doit ressembler a cela:

```yaml
kind: Service
apiVersion: v1
metadata:
  name: result-service
  namespace: voting-app
  labels:
    app: result-service
spec:
  ports:
    - name: 8080-tcp
      protocol: TCP
      port: 8080
      targetPort: 8080
  selector:
    deployment: result-deployment
  type: ClusterIP
```

```shell
oc apply -f result-service.yaml
oc expose service/result-service
```


## Worker

```java
package worker;

import redis.clients.jedis.Jedis;
import redis.clients.jedis.exceptions.JedisConnectionException;
import java.sql.*;
import org.json.JSONObject;

class Worker {
  public static void main(String[] args) {
    try {
      Jedis redis = connectToRedis("redis");
      Connection dbConn = connectToDB("db");

      System.err.println("Watching vote queue");

      while (true) {
        String voteJSON = redis.blpop(0, "votes").get(1);
        JSONObject voteData = new JSONObject(voteJSON);
        String voterID = voteData.getString("voter_id");
        String vote = voteData.getString("vote");

        System.err.printf("Processing vote for '%s' by '%s'\n", vote, voterID);
        updateVote(dbConn, voterID, vote);
      }
    } catch (SQLException e) {
      e.printStackTrace();
      System.exit(1);
    }
  }

  static void updateVote(Connection dbConn, String voterID, String vote) throws SQLException {
    PreparedStatement insert = dbConn.prepareStatement(
      "INSERT INTO votes (id, vote) VALUES (?, ?)");
    insert.setString(1, voterID);
    insert.setString(2, vote);

    try {
      insert.executeUpdate();
    } catch (SQLException e) {
      PreparedStatement update = dbConn.prepareStatement(
        "UPDATE votes SET vote = ? WHERE id = ?");
      update.setString(1, vote);
      update.setString(2, voterID);
      update.executeUpdate();
    }
  }

  static Jedis connectToRedis(String host) {
    Jedis conn = new Jedis(host);
    conn.auth("redis");

    while (true) {
      try {
        conn.keys("*");
        break;
      } catch (JedisConnectionException e) {
        System.err.println("Waiting for redis");
        sleep(1000);
      }
    }

    System.err.println("Connected to redis");
    return conn;
  }

  static Connection connectToDB(String host) throws SQLException {
    Connection conn = null;

    try {

      Class.forName("org.postgresql.Driver");
      String url = "jdbc:postgresql://" + host + "/postgres";

      while (conn == null) {
        try {
          conn = DriverManager.getConnection(url, "postgres", "postgres");
        } catch (SQLException e) {
          System.err.println("Waiting for db");
          sleep(1000);
        }
      }

      PreparedStatement st = conn.prepareStatement(
        "CREATE TABLE IF NOT EXISTS votes (id VARCHAR(255) NOT NULL UNIQUE, vote VARCHAR(255) NOT NULL)");
      st.executeUpdate();

    } catch (ClassNotFoundException e) {
      e.printStackTrace();
      System.exit(1);
    }

    System.err.println("Connected to db");
    return conn;
  }

  static void sleep(long duration) {
    try {
      Thread.sleep(duration);
    } catch (InterruptedException e) {
      System.exit(1);
    }
  }
}
```

## Containerize voter

 Pour notre image back-end nous allons utiliser une image maven. Les fichiers du dossier sonrce son ajoute compule et package dans un fat jar

```Dockerfile
FROM maven:3.5-jdk-8-alpine AS build

WORKDIR /code

COPY src/pom.xml /code/pom.xml
RUN ["mvn", "dependency:resolve"]
RUN ["mvn", "verify"]

# Adding source, compile and package into a fat jar
COPY ["src/src/main", "/code/src/main"]
RUN ["mvn", "package"]
FROM openjdk:8-jre
WORKDIR /app
RUN useradd -u 1001 -r -g 0 -d /app -s /sbin/nologin \
    -c "Default Application User" default && \
    chown -R 1001:0 /app && \
    chmod -R g+rw /app
USER 1001
COPY --from=build /code/target/worker-jar-with-dependencies.jar /
CMD ["java", "-XX:+UnlockExperimentalVMOptions", "-XX:+UseCGroupMemoryLimitForHeap", "-jar", "/worker-jar-with-dependencies.jar"]
```

Une fois conteneurise pousser votre image dans votre registry

```shell
podman build -t quay.io/<your-quay-name>/worker .
podman push quay.io/<your-quay-name>/worker
```

## Deployment  et service worker

Nous allons ensuite creer un deployment  et un service. Ceux-ci n'ont pas besoin d'etre exposer on ne retrouve donc pas de section port et nous ne creerons pas de service ici.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: worker-deployment
  labels:
    app: worker-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: worker-pod
  template:
    metadata:
      name: worker-pod
      labels:
        deployment: worker-pod
spec:
    containers:
      - name: worker
        image: quay.io/feven/worker
        imagePullPolicy: IfNotPresent
        securityContext:
          runAsUser: 1001  
```

```shell
oc apply -f result-deployment.yaml
```




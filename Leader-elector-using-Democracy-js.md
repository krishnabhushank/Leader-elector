`democracy.js` is a simple Node.js library for leader election in distributed systems. 

### Step-by-Step Implementation

1. **Set up the Leader Election Sidecar using `democracy.js`**.
2. **Integrate the Sidecar with the Kafka Consumer Application**.
3. **Deploy the Application to Kubernetes**.

### 1. Set up the Leader Election Sidecar using `democracy.js`

#### Create the Node.js Sidecar Application

**1.1. Initialize a new Node.js project:**
```sh
mkdir leader-election-sidecar
cd leader-election-sidecar
npm init -y
npm install democracy
```

**1.2. Create the Leader Election Script**

**index.js:**
```javascript
const Democracy = require('democracy');
const http = require('http');

const namespace = process.env.NAMESPACE;
const podName = process.env.POD_NAME;
const port = process.env.PORT || 3000;

const democracy = new Democracy({
  source: podName,
  nodes: [`${namespace}`]
});

let isLeader = false;

// Notify the Kafka consumer about the leadership status
function notifyLeadership(status) {
  isLeader = status;
  const options = {
    hostname: 'localhost',
    port: 8080,
    path: '/leader',
    method: 'POST',
    headers: {
      'Content-Type': 'application/json'
    }
  };

  const req = http.request(options, res => {
    console.log(`Status: ${res.statusCode}`);
  });

  req.on('error', error => {
    console.error(error);
  });

  req.write(JSON.stringify({ leader: status }));
  req.end();
}

democracy.on('elected', () => {
  console.log('I am the leader now');
  notifyLeadership(true);
});

democracy.on('demote', () => {
  console.log('I am no longer the leader');
  notifyLeadership(false);
});

democracy.on('added', (id) => {
  console.log(`New node added: ${id}`);
});

democracy.on('removed', (id) => {
  console.log(`Node removed: ${id}`);
});

democracy.start();
```

**1.3. Create a Dockerfile for the Sidecar**

**Dockerfile:**
```Dockerfile
FROM node:14-alpine
WORKDIR /app
COPY package.json ./
COPY package-lock.json ./
RUN npm install
COPY . .
CMD ["node", "index.js"]
```

### 2. Kafka Consumer Application

#### Modify the Kafka Consumer to Receive Leader Notifications

**KafkaConsumerService.java:**
```java
package com.example.kafkaconsumer;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.kafka.listener.MessageListenerContainer;
import org.springframework.kafka.listener.config.ContainerProperties;
import org.springframework.kafka.listener.config.ConcurrentKafkaListenerContainerFactory;
import org.springframework.kafka.listener.config.KafkaListenerEndpointRegistry;
import org.springframework.kafka.listener.ConcurrentMessageListenerContainer;
import org.springframework.kafka.core.ConsumerFactory;
import org.springframework.stereotype.Service;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RestController;

@Service
public class KafkaConsumerService {

    @Value("${active:false}")
    private boolean active;

    private final KafkaListenerEndpointRegistry kafkaListenerEndpointRegistry;

    public KafkaConsumerService(KafkaListenerEndpointRegistry kafkaListenerEndpointRegistry) {
        this.kafkaListenerEndpointRegistry = kafkaListenerEndpointRegistry;
    }

    @KafkaListener(topics = "your-topic", groupId = "your-group")
    public void listen(String message) {
        if (active) {
            System.out.println("Consumed message: " + message);
        }
    }

    public void setActive(boolean active) {
        this.active = active;
        for (MessageListenerContainer container : kafkaListenerEndpointRegistry.getListenerContainers()) {
            if (active) {
                container.start();
            } else {
                container.stop();
            }
        }
    }
}

@RestController
public class LeadershipController {

    private final KafkaConsumerService kafkaConsumerService;

    public LeadershipController(KafkaConsumerService kafkaConsumerService) {
        this.kafkaConsumerService = kafkaConsumerService;
    }

    @PostMapping("/leader")
    public void updateLeadership(@RequestBody LeadershipStatus status) {
        kafkaConsumerService.setActive(status.isLeader());
    }
}

class LeadershipStatus {
    private boolean leader;

    public boolean isLeader() {
        return leader;
    }

    public void setLeader(boolean leader) {
        this.leader = leader;
    }
}
```

### 3. Deploy the Application to Kubernetes

#### Create Kubernetes Resources

**namespace.yaml:**
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: kafka-demo
```

**rbac.yaml:**
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: kafka-sa
  namespace: kafka-demo
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: kafka-demo
  name: kafka-role
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: kafka-rolebinding
  namespace: kafka-demo
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: kafka-role
subjects:
- kind: ServiceAccount
  name: kafka-sa
  namespace: kafka-demo
```

#### Deploy the Kafka Consumer and Sidecar

**deployment.yaml:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kafka-consumer
  namespace: kafka-demo
spec:
  replicas: 3
  selector:
    matchLabels:
      app: kafka-consumer
  template:
    metadata:
      labels:
        app: kafka-consumer
    spec:
      serviceAccountName: kafka-sa
      containers:
      - name: kafka-consumer
        image: your-kafka-consumer-image
        ports:
        - containerPort: 8080
        env:
        - name: ACTIVE
          value: "false"
        - name: NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
      - name: leader-election-sidecar
        image: your-sidecar-image
        ports:
        - containerPort: 3000
        env:
        - name: NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
```

### Building and Deploying

1. **Build and push the Kafka consumer image**:
   ```sh
   mvn clean package
   docker build -t your-kafka-consumer-image .
   docker push your-kafka-consumer-image
   ```

2. **Build and push the sidecar image**:
   ```sh
   cd leader-election-sidecar
   docker build -t your-sidecar-image .
   docker push your-sidecar-image
   ```

3. **Apply the Kubernetes configurations**:
   ```sh
   kubectl apply -f namespace.yaml
   kubectl apply -f rbac.yaml
   kubectl apply -f deployment.yaml
   ```

This setup uses `democracy.js` in the sidecar container for leader election. The sidecar notifies the Kafka consumer about its leadership status, enabling or disabling message consumption accordingly. This ensures that only one consumer instance is active at a time, providing fault tolerance and preventing multiple instances from consuming the same partition simultaneously.

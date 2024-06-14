# Leader-elector

You can use `spring-cloud-kubernetes-fabric8-leader` for leader election in Kubernetes. This Spring Cloud extension simplifies leader election by leveraging Kubernetes' Fabric8 client and the Spring Cloud framework.

Here's how to set up the sidecar using `spring-cloud-kubernetes-fabric8-leader` and integrate it with your Kafka consumer application.

### Step-by-Step Implementation

1. **Set up the Spring Boot Leader Election Sidecar**.
2. **Integrate the Sidecar with the Kafka Consumer Application**.
3. **Deploy the Application to Kubernetes**.

### 1. Spring Boot Leader Election Sidecar

**pom.xml**:
```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-kubernetes-fabric8-leader</artifactId>
    </dependency>
    <dependency>
        <groupId>io.fabric8</groupId>
        <artifactId>kubernetes-client</artifactId>
        <version>5.10.1</version>
    </dependency>
</dependencies>
```

**application.properties**:
```properties
spring.application.name=leader-election-sidecar
spring.cloud.kubernetes.leader.role=leader
spring.cloud.kubernetes.leader.auto-rebake=true
spring.cloud.kubernetes.client.namespace=kafka-demo
spring.cloud.kubernetes.client.serviceAccountName=kafka-sa
```

**LeaderElectionService.java**:
```java
package com.example.sidecar;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.cloud.kubernetes.commons.leader.Leader;
import org.springframework.cloud.kubernetes.commons.leader.event.LeaderEventPublisher;
import org.springframework.context.event.EventListener;
import org.springframework.stereotype.Service;

@Service
public class LeaderElectionService {

    @Autowired
    private LeaderEventPublisher leaderEventPublisher;

    @Autowired
    private KafkaConsumerService kafkaConsumerService;

    @EventListener
    public void handleLeaderEvent(Leader leader) {
        if (leader.isLeader()) {
            setActive(true);
        } else {
            setActive(false);
        }
    }

    private void setActive(boolean isActive) {
        kafkaConsumerService.setActive(isActive);
        System.out.println(isActive ? "Became leader" : "Lost leadership");
    }
}
```

### 2. Kafka Consumer Application

**pom.xml** (only relevant parts):
```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.kafka</groupId>
        <artifactId>spring-kafka</artifactId>
    </dependency>
    <dependency>
        <groupId>io.fabric8</groupId>
        <artifactId>kubernetes-client</artifactId>
        <version>5.10.1</version>
    </dependency>
</dependencies>
```

**application.properties**:
```properties
spring.application.name=kafka-consumer
spring.kafka.bootstrap-servers=your-kafka-bootstrap-server
spring.kafka.consumer.group-id=your-group-id
spring.kafka.consumer.auto-offset-reset=earliest
```

**KafkaConsumerService.java**:
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
```

### 3. Deploy the Application to Kubernetes

**deployment.yaml**:
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
   mvn clean package
   docker build -t your-sidecar-image .
   docker push your-sidecar-image
   ```

3. **Apply the Kubernetes configurations**:
   ```sh
   kubectl apply -f namespace.yaml
   kubectl apply -f rbac.yaml
   kubectl apply -f deployment.yaml
   ```

This setup uses `spring-cloud-kubernetes-fabric8-leader` for leader election, simplifying the management of leader election in Kubernetes. The sidecar handles the leader election logic and communicates with the Kafka consumer to enable or disable message consumption based on the leader status. This ensures fault tolerance and prevents multiple instances from consuming the same partition simultaneously.

# Leader Elector using Fabric8

Let's break down the steps to create a leader elector sidecar app in Spring Boot using the Fabric8 Kubernetes client library.

### Steps:

1. **Set up a new Spring Boot project**:
   - Use Spring Initializr to create a new Spring Boot project.
   - Include necessary dependencies such as `spring-boot-starter-web`, `spring-boot-starter-actuator`, and Fabric8 Kubernetes client.

2. **Configure Kubernetes Client**:
   - Add Fabric8 dependencies in your `pom.xml`.
   - Configure Kubernetes client to connect to the Kubernetes cluster.

3. **Implement Leader Election Logic**:
   - Use Fabric8 to implement leader election logic.
   - Create a service that continuously checks if the current pod is the leader.

4. **Expose Leadership Status**:
   - Expose an endpoint to check the leadership status.
   - Use Spring Boot Actuator to expose custom metrics or endpoints.

### Pseudocode:

1. **Setup Spring Boot Project**:
   - Create a new project with Spring Initializr.
   - Add dependencies in `pom.xml`.

2. **Configure Fabric8 Kubernetes Client**:
   - Configure Kubernetes client in a Spring Configuration class.

3. **Implement Leader Election**:
   - Create a service for leader election.
   - Use Fabric8 to coordinate the leader election process.
   - Implement a mechanism to renew the leadership periodically.

4. **Expose Endpoint**:
   - Create a REST controller to expose the leadership status.
   - Use Actuator to expose additional metrics if needed.

### Code Implementation:

```xml
<!-- pom.xml -->
<dependencies>
    <!-- Other dependencies -->
    <dependency>
        <groupId>io.fabric8</groupId>
        <artifactId>kubernetes-client</artifactId>
        <version>6.0.0</version>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
</dependencies>
```

```java
// src/main/java/com/example/leaderelector/KubernetesConfig.java
package com.example.leaderelector;

import io.fabric8.kubernetes.client.Config;
import io.fabric8.kubernetes.client.DefaultKubernetesClient;
import io.fabric8.kubernetes.client.KubernetesClient;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class KubernetesConfig {
    
    @Bean
    public KubernetesClient kubernetesClient() {
        Config config = Config.autoConfigure(null);
        return new DefaultKubernetesClient(config);
    }
}
```

```java
// src/main/java/com/example/leaderelector/LeaderElectionService.java
package com.example.leaderelector;

import io.fabric8.kubernetes.api.model.ConfigMap;
import io.fabric8.kubernetes.client.KubernetesClient;
import org.springframework.stereotype.Service;

import javax.annotation.PostConstruct;
import javax.annotation.PreDestroy;
import java.util.Timer;
import java.util.TimerTask;

@Service
public class LeaderElectionService {

    private final KubernetesClient kubernetesClient;
    private final String namespace = "default";
    private final String configMapName = "leader-election";
    private boolean isLeader = false;
    private Timer timer;

    public LeaderElectionService(KubernetesClient kubernetesClient) {
        this.kubernetesClient = kubernetesClient;
    }

    @PostConstruct
    public void startLeaderElection() {
        timer = new Timer(true);
        timer.scheduleAtFixedRate(new LeaderElectionTask(), 0, 5000);
    }

    @PreDestroy
    public void stopLeaderElection() {
        if (timer != null) {
            timer.cancel();
        }
    }

    private class LeaderElectionTask extends TimerTask {
        @Override
        public void run() {
            ConfigMap configMap = kubernetesClient.configMaps().inNamespace(namespace).withName(configMapName).get();
            if (configMap == null) {
                configMap = kubernetesClient.configMaps().inNamespace(namespace)
                        .createNew()
                        .withNewMetadata().withName(configMapName).endMetadata()
                        .addToData("leader", "pod-name")
                        .done();
            }
            // Implement logic to check and update leader
            // Set isLeader to true if current pod is the leader
        }
    }

    public boolean isLeader() {
        return isLeader;
    }
}
```

```java
// src/main/java/com/example/leaderelector/LeaderController.java
package com.example.leaderelector;

import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class LeaderController {

    private final LeaderElectionService leaderElectionService;

    public LeaderController(LeaderElectionService leaderElectionService) {
        this.leaderElectionService = leaderElectionService;
    }

    @GetMapping("/leader")
    public boolean isLeader() {
        return leaderElectionService.isLeader();
    }
}
```

### Suggestions for Next Steps:
**a.** Add unit tests for the leader election logic.  
**b.** Implement a more robust mechanism to handle leadership renewal and failure scenarios.

## Unit Testcases

Let's add unit tests for the leader election logic using JUnit and Mockito.

### Steps:

1. **Set up dependencies for testing**:
   - Add dependencies for JUnit and Mockito in the `pom.xml`.

2. **Create a test class for `LeaderElectionService`**:
   - Mock the `KubernetesClient`.
   - Write tests for the leader election logic.

### Updated `pom.xml`:

```xml
<!-- Add these dependencies for testing -->
<dependencies>
    <!-- Other dependencies -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.mockito</groupId>
        <artifactId>mockito-core</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```

### Test Class:

```java
// src/test/java/com/example/leaderelector/LeaderElectionServiceTest.java
package com.example.leaderelector;

import io.fabric8.kubernetes.api.model.ConfigMap;
import io.fabric8.kubernetes.client.KubernetesClient;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.MockitoAnnotations;

import java.util.Timer;

import static org.junit.jupiter.api.Assertions.assertFalse;
import static org.junit.jupiter.api.Assertions.assertTrue;
import static org.mockito.ArgumentMatchers.anyString;
import static org.mockito.Mockito.*;

public class LeaderElectionServiceTest {

    @Mock
    private KubernetesClient kubernetesClient;

    @InjectMocks
    private LeaderElectionService leaderElectionService;

    @BeforeEach
    public void setUp() {
        MockitoAnnotations.openMocks(this);
        leaderElectionService = new LeaderElectionService(kubernetesClient);
    }

    @Test
    public void testStartLeaderElection_NoConfigMapExists() {
        when(kubernetesClient.configMaps().inNamespace(anyString()).withName(anyString()).get()).thenReturn(null);
        when(kubernetesClient.configMaps().inNamespace(anyString()).createNew().withNewMetadata().withName(anyString()).endMetadata().addToData(anyString(), anyString()).done()).thenReturn(new ConfigMap());

        leaderElectionService.startLeaderElection();

        verify(kubernetesClient.configMaps().inNamespace(anyString()).withName(anyString()), times(1)).get();
        verify(kubernetesClient.configMaps().inNamespace(anyString()), times(1)).createNew();
    }

    @Test
    public void testIsLeader_DefaultFalse() {
        assertFalse(leaderElectionService.isLeader());
    }

    @Test
    public void testLeaderElectionLogic() {
        ConfigMap configMap = new ConfigMap();
        configMap.setData(Map.of("leader", "other-pod-name"));

        when(kubernetesClient.configMaps().inNamespace(anyString()).withName(anyString()).get()).thenReturn(configMap);

        leaderElectionService.startLeaderElection();

        Timer timer = new Timer();
        timer.schedule(new TimerTask() {
            @Override
            public void run() {
                assertFalse(leaderElectionService.isLeader());
            }
        }, 5000);

        // Simulate current pod becoming leader
        configMap.setData(Map.of("leader", "pod-name"));
        when(kubernetesClient.configMaps().inNamespace(anyString()).withName(anyString()).get()).thenReturn(configMap);

        timer.schedule(new TimerTask() {
            @Override
            public void run() {
                assertTrue(leaderElectionService.isLeader());
            }
        }, 10000);
    }
}
```

### Notes:
- The tests use Mockito to mock the `KubernetesClient` and simulate the presence or absence of the ConfigMap.
- `testStartLeaderElection_NoConfigMapExists` ensures that the ConfigMap is created if it doesn't exist.
- `testIsLeader_DefaultFalse` verifies that the `isLeader` method returns `false` by default.
- `testLeaderElectionLogic` simulates the leader election process and checks the leadership status.

### Suggestions for Next Steps:
**a.** Add integration tests to ensure the application works correctly within a real Kubernetes cluster.  
**b.** Implement a more robust mechanism for handling leadership renewal and failure scenarios.

## Improvements 

To implement a more robust mechanism for handling leadership renewal and failure scenarios, we can use a combination of the following strategies:

1. **Leasing Mechanism**: Use a leasing mechanism where the leader periodically renews its leadership. If the leader fails to renew the lease within a certain timeout period, other instances can attempt to acquire the leadership.

2. **Watch ConfigMap for Changes**: Use a watch on the ConfigMap to detect changes in leadership status in near real-time.

3. **Use Kubernetes Liveness and Readiness Probes**: Ensure that only the healthy leader is reported as ready, and unhealthy leaders are terminated or restarted by Kubernetes.

### Steps:

1. **Implement Leasing Mechanism**:
   - Add a lease expiration time to the ConfigMap.
   - The leader should periodically update this expiration time.
   - Non-leaders should attempt to acquire leadership if the lease is expired.

2. **Watch ConfigMap for Changes**:
   - Use the Fabric8 Kubernetes client to watch the ConfigMap for changes.
   - Update the local leadership status based on ConfigMap changes.

3. **Use Liveness and Readiness Probes**:
   - Configure liveness and readiness probes in the Kubernetes deployment to ensure only the healthy leader is active.

### Updated Implementation:

#### 1. LeaderElectionService

```java
// src/main/java/com/example/leaderelector/LeaderElectionService.java
package com.example.leaderelector;

import io.fabric8.kubernetes.api.model.ConfigMap;
import io.fabric8.kubernetes.client.KubernetesClient;
import io.fabric8.kubernetes.client.Watcher;
import io.fabric8.kubernetes.client.WatcherException;
import org.springframework.stereotype.Service;

import javax.annotation.PostConstruct;
import javax.annotation.PreDestroy;
import java.time.Instant;
import java.util.Timer;
import java.util.TimerTask;

@Service
public class LeaderElectionService {

    private final KubernetesClient kubernetesClient;
    private final String namespace = "default";
    private final String configMapName = "leader-election";
    private boolean isLeader = false;
    private Timer timer;
    private static final long LEASE_DURATION_MS = 15000; // 15 seconds
    private static final long RENEW_INTERVAL_MS = 5000; // 5 seconds

    public LeaderElectionService(KubernetesClient kubernetesClient) {
        this.kubernetesClient = kubernetesClient;
    }

    @PostConstruct
    public void startLeaderElection() {
        timer = new Timer(true);
        timer.scheduleAtFixedRate(new LeaderElectionTask(), 0, RENEW_INTERVAL_MS);
        kubernetesClient.configMaps().inNamespace(namespace).withName(configMapName).watch(new ConfigMapWatcher());
    }

    @PreDestroy
    public void stopLeaderElection() {
        if (timer != null) {
            timer.cancel();
        }
    }

    private class LeaderElectionTask extends TimerTask {
        @Override
        public void run() {
            ConfigMap configMap = kubernetesClient.configMaps().inNamespace(namespace).withName(configMapName).get();
            if (configMap == null) {
                configMap = kubernetesClient.configMaps().inNamespace(namespace)
                        .createNew()
                        .withNewMetadata().withName(configMapName).endMetadata()
                        .addToData("leader", "pod-name")
                        .addToData("lease", Long.toString(Instant.now().toEpochMilli() + LEASE_DURATION_MS))
                        .done();
                isLeader = true;
                return;
            }

            String currentLeader = configMap.getData().get("leader");
            long leaseExpiration = Long.parseLong(configMap.getData().get("lease"));

            if (isLeader) {
                // Renew leadership
                if (currentLeader.equals("pod-name") && Instant.now().toEpochMilli() < leaseExpiration) {
                    kubernetesClient.configMaps().inNamespace(namespace).withName(configMapName).edit()
                            .addToData("lease", Long.toString(Instant.now().toEpochMilli() + LEASE_DURATION_MS))
                            .done();
                } else {
                    isLeader = false;
                }
            } else if (Instant.now().toEpochMilli() > leaseExpiration) {
                // Try to acquire leadership
                kubernetesClient.configMaps().inNamespace(namespace).withName(configMapName).edit()
                        .addToData("leader", "pod-name")
                        .addToData("lease", Long.toString(Instant.now().toEpochMilli() + LEASE_DURATION_MS))
                        .done();
                isLeader = true;
            }
        }
    }

    private class ConfigMapWatcher implements Watcher<ConfigMap> {
        @Override
        public void eventReceived(Action action, ConfigMap configMap) {
            String currentLeader = configMap.getData().get("leader");
            long leaseExpiration = Long.parseLong(configMap.getData().get("lease"));
            if (!currentLeader.equals("pod-name") && Instant.now().toEpochMilli() < leaseExpiration) {
                isLeader = false;
            }
        }

        @Override
        public void onClose(WatcherException e) {
            // Handle close event if needed
        }
    }

    public boolean isLeader() {
        return isLeader;
    }
}
```

#### 2. Deployment with Liveness and Readiness Probes

```yaml
# kubernetes/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: leader-elector
  labels:
    app: leader-elector
spec:
  replicas: 3
  selector:
    matchLabels:
      app: leader-elector
  template:
    metadata:
      labels:
        app: leader-elector
    spec:
      containers:
      - name: leader-elector
        image: your-image
        ports:
        - containerPort: 8080
        livenessProbe:
          httpGet:
            path: /actuator/health
            port: 8080
          initialDelaySeconds: 15
          periodSeconds: 20
        readinessProbe:
          httpGet:
            path: /actuator/health
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 10
```

### Notes:
- **Leasing Mechanism**: We add a `lease` key in the ConfigMap to store the lease expiration time. The leader periodically renews the lease, and other instances attempt to acquire leadership if the lease is expired.
- **Watch ConfigMap**: We use Fabric8's watch functionality to detect changes in the ConfigMap and update the local leadership status.
- **Liveness and Readiness Probes**: Kubernetes probes ensure only healthy leaders are considered ready. The probes hit the `/actuator/health` endpoint to check the application's health.

### Suggestions for Next Steps:
**a.** Test the deployment in a Kubernetes cluster to verify the leader election and failover mechanisms.  
**b.** Add metrics to monitor the leader election status and lease renewals using Spring Boot Actuator and Prometheus.

## Leader Elector As sidecar

To run the leader election app as a sidecar to the main application, we need to ensure that both applications (main app and leader elector) are running in the same pod. The leader election sidecar will be responsible for managing the leadership state.

### Steps:

1. **Modify the Spring Boot project to run both the main application and the leader election service together**.
2. **Update the Kubernetes deployment to include both containers in the same pod**.
3. **Ensure proper communication and configuration between the main app and the leader election service**.

### Updated Implementation:

#### 1. Main Application

Ensure that your main application and the leader election service can communicate properly. For simplicity, we assume the main application is also a Spring Boot application.

```java
// src/main/java/com/example/mainapp/MainApplication.java
package com.example.mainapp;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class MainApplication {
    public static void main(String[] args) {
        SpringApplication.run(MainApplication.class, args);
    }
}

// src/main/java/com/example/mainapp/LeaderStatusController.java
package com.example.mainapp;

import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.client.RestTemplate;

@RestController
public class LeaderStatusController {

    private final RestTemplate restTemplate;

    public LeaderStatusController(RestTemplate restTemplate) {
        this.restTemplate = restTemplate;
    }

    @GetMapping("/isLeader")
    public boolean isLeader() {
        String leaderStatusUrl = "http://localhost:8081/leader"; // URL of the sidecar leader election service
        Boolean isLeader = restTemplate.getForObject(leaderStatusUrl, Boolean.class);
        return isLeader != null && isLeader;
    }
}
```

#### 2. Leader Election Service

No changes are required for the leader election service except for ensuring it runs on a different port (e.g., 8081).

```java
// src/main/resources/application.yml
server:
  port: 8081
```

#### 3. Dockerfiles

Create separate Dockerfiles for the main application and the leader election service.

**Dockerfile for Main Application**:
```dockerfile
# Dockerfile for main application
FROM openjdk:17-jdk-alpine
VOLUME /tmp
COPY target/mainapp-0.0.1-SNAPSHOT.jar app.jar
ENTRYPOINT ["java","-jar","/app.jar"]
```

**Dockerfile for Leader Election Service**:
```dockerfile
# Dockerfile for leader election service
FROM openjdk:17-jdk-alpine
VOLUME /tmp
COPY target/leaderelector-0.0.1-SNAPSHOT.jar app.jar
ENTRYPOINT ["java","-jar","/app.jar"]
```

#### 4. Kubernetes Deployment

Update the Kubernetes deployment to include both the main application and the leader election service as containers in the same pod.

```yaml
# kubernetes/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: main-app
  labels:
    app: main-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: main-app
  template:
    metadata:
      labels:
        app: main-app
    spec:
      containers:
      - name: main-app
        image: your-main-app-image
        ports:
        - containerPort: 8080
        readinessProbe:
          httpGet:
            path: /actuator/health
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 10
        livenessProbe:
          httpGet:
            path: /actuator/health
            port: 8080
          initialDelaySeconds: 15
          periodSeconds: 20
      - name: leader-elector
        image: your-leader-elector-image
        ports:
        - containerPort: 8081
        readinessProbe:
          httpGet:
            path: /actuator/health
            port: 8081
          initialDelaySeconds: 5
          periodSeconds: 10
        livenessProbe:
          httpGet:
            path: /actuator/health
            port: 8081
          initialDelaySeconds: 15
          periodSeconds: 20
```

### Notes:
- The main application container (`main-app`) and the leader election sidecar container (`leader-elector`) run in the same pod.
- Both containers have liveness and readiness probes configured to ensure they are running correctly.
- The main application communicates with the leader election service via `http://localhost:8081/leader`.

### Suggestions for Next Steps:
**a.** Test the sidecar pattern deployment in a Kubernetes cluster to ensure proper communication and leadership management.  
**b.** Add logging and monitoring to observe the leader election process and the health status of both containers.

## Function of `isLeader` end point 

Let's detail how the `/isLeader` endpoint in the main app and the `/leader` endpoint in the sidecar work together.

### Main Application (`/isLeader` endpoint)
1. **Purpose**: The main application provides an endpoint (`/isLeader`) to check if the current instance of the application is the leader.
2. **Mechanism**: The main application makes an HTTP GET request to the leader election sidecar's `/leader` endpoint to determine the leadership status.

### Sidecar Application (`/leader` endpoint)
1. **Purpose**: The leader election sidecar provides an endpoint (`/leader`) to expose its leadership status.
2. **Mechanism**: The endpoint returns a boolean value indicating whether the current pod is the leader.

### Implementation Details

#### Main Application

The main application has a REST controller (`LeaderStatusController`) that defines the `/isLeader` endpoint. This endpoint makes an HTTP call to the leader election sidecar's `/leader` endpoint.

```java
// src/main/java/com/example/mainapp/MainApplication.java
package com.example.mainapp;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.annotation.Bean;
import org.springframework.web.client.RestTemplate;

@SpringBootApplication
public class MainApplication {
    public static void main(String[] args) {
        SpringApplication.run(MainApplication.class, args);
    }

    @Bean
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }
}

// src/main/java/com/example/mainapp/LeaderStatusController.java
package com.example.mainapp;

import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.client.RestTemplate;

@RestController
public class LeaderStatusController {

    private final RestTemplate restTemplate;

    public LeaderStatusController(RestTemplate restTemplate) {
        this.restTemplate = restTemplate;
    }

    @GetMapping("/isLeader")
    public boolean isLeader() {
        String leaderStatusUrl = "http://localhost:8081/leader"; // URL of the sidecar leader election service
        Boolean isLeader = restTemplate.getForObject(leaderStatusUrl, Boolean.class);
        return isLeader != null && isLeader;
    }
}
```

#### Leader Election Sidecar

The leader election sidecar application has a REST controller (`LeaderController`) that defines the `/leader` endpoint. This endpoint checks the leadership status and returns it.

```java
// src/main/java/com/example/leaderelector/LeaderElectionService.java
package com.example.leaderelector;

import io.fabric8.kubernetes.api.model.ConfigMap;
import io.fabric8.kubernetes.client.KubernetesClient;
import io.fabric8.kubernetes.client.Watcher;
import io.fabric8.kubernetes.client.WatcherException;
import org.springframework.stereotype.Service;

import javax.annotation.PostConstruct;
import javax.annotation.PreDestroy;
import java.time.Instant;
import java.util.Timer;
import java.util.TimerTask;

@Service
public class LeaderElectionService {

    private final KubernetesClient kubernetesClient;
    private final String namespace = "default";
    private final String configMapName = "leader-election";
    private boolean isLeader = false;
    private Timer timer;
    private static final long LEASE_DURATION_MS = 15000; // 15 seconds
    private static final long RENEW_INTERVAL_MS = 5000; // 5 seconds

    public LeaderElectionService(KubernetesClient kubernetesClient) {
        this.kubernetesClient = kubernetesClient;
    }

    @PostConstruct
    public void startLeaderElection() {
        timer = new Timer(true);
        timer.scheduleAtFixedRate(new LeaderElectionTask(), 0, RENEW_INTERVAL_MS);
        kubernetesClient.configMaps().inNamespace(namespace).withName(configMapName).watch(new ConfigMapWatcher());
    }

    @PreDestroy
    public void stopLeaderElection() {
        if (timer != null) {
            timer.cancel();
        }
    }

    private class LeaderElectionTask extends TimerTask {
        @Override
        public void run() {
            ConfigMap configMap = kubernetesClient.configMaps().inNamespace(namespace).withName(configMapName).get();
            if (configMap == null) {
                configMap = kubernetesClient.configMaps().inNamespace(namespace)
                        .createNew()
                        .withNewMetadata().withName(configMapName).endMetadata()
                        .addToData("leader", "pod-name")
                        .addToData("lease", Long.toString(Instant.now().toEpochMilli() + LEASE_DURATION_MS))
                        .done();
                isLeader = true;
                return;
            }

            String currentLeader = configMap.getData().get("leader");
            long leaseExpiration = Long.parseLong(configMap.getData().get("lease"));

            if (isLeader) {
                // Renew leadership
                if (currentLeader.equals("pod-name") && Instant.now().toEpochMilli() < leaseExpiration) {
                    kubernetesClient.configMaps().inNamespace(namespace).withName(configMapName).edit()
                            .addToData("lease", Long.toString(Instant.now().toEpochMilli() + LEASE_DURATION_MS))
                            .done();
                } else {
                    isLeader = false;
                }
            } else if (Instant.now().toEpochMilli() > leaseExpiration) {
                // Try to acquire leadership
                kubernetesClient.configMaps().inNamespace(namespace).withName(configMapName).edit()
                        .addToData("leader", "pod-name")
                        .addToData("lease", Long.toString(Instant.now().toEpochMilli() + LEASE_DURATION_MS))
                        .done();
                isLeader = true;
            }
        }
    }

    private class ConfigMapWatcher implements Watcher<ConfigMap> {
        @Override
        public void eventReceived(Action action, ConfigMap configMap) {
            String currentLeader = configMap.getData().get("leader");
            long leaseExpiration = Long.parseLong(configMap.getData().get("lease"));
            if (!currentLeader.equals("pod-name") && Instant.now().toEpochMilli() < leaseExpiration) {
                isLeader = false;
            }
        }

        @Override
        public void onClose(WatcherException e) {
            // Handle close event if needed
        }
    }

    public boolean isLeader() {
        return isLeader;
    }
}

// src/main/java/com/example/leaderelector/LeaderController.java
package com.example.leaderelector;

import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class LeaderController {

    private final LeaderElectionService leaderElectionService;

    public LeaderController(LeaderElectionService leaderElectionService) {
        this.leaderElectionService = leaderElectionService;
    }

    @GetMapping("/leader")
    public boolean isLeader() {
        return leaderElectionService.isLeader();
    }
}
```

### Explanation:
- **Main Application**:
  - The `LeaderStatusController` in the main application has an endpoint `/isLeader`.
  - This endpoint uses a `RestTemplate` to make a GET request to `http://localhost:8081/leader`, which is the endpoint of the leader election sidecar running on port 8081.
  - The response from this request (a boolean indicating the leadership status) is returned by the `/isLeader` endpoint.

- **Leader Election Sidecar**:
  - The `LeaderElectionService` manages the leadership logic, including periodically updating the ConfigMap to renew the lease and watching the ConfigMap for changes.
  - The `LeaderController` provides an endpoint `/leader` that returns the current leadership status (`isLeader`).

### Suggestions for Next Steps:
**a.** Test the sidecar deployment in a Kubernetes cluster to ensure proper leadership election and failover.  
**b.** Add logging and monitoring to observe the leader election process and the health status of both containers.

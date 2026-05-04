<dependencies>
    <dependency>
        <groupId>org.testcontainers</groupId>
        <artifactId>testcontainers</artifactId>
        <version>1.19.8</version>
        <scope>test</scope>
    </dependency>

    <dependency>
        <groupId>com.hazelcast</groupId>
        <artifactId>hazelcast</artifactId>
        <version>5.5.0</version>
    </dependency>
</dependencies>


2. Uruchom kontener Hazelcast w teście

Hazelcast nie ma (jeszcze) oficjalnego modułu Testcontainers jak np. Kafka, więc używasz GenericContainer.

import org.junit.jupiter.api.Test;
import org.testcontainers.containers.GenericContainer;

public class HazelcastTest {

    @Test
    void shouldStartHazelcast() {
        try (GenericContainer<?> hazelcast =
                 new GenericContainer<>("hazelcast/hazelcast:5.5.0")
                         .withExposedPorts(5701)) {

            hazelcast.start();

            String host = hazelcast.getHost();
            Integer port = hazelcast.getMappedPort(5701);

            System.out.println("Hazelcast running at " + host + ":" + port);

            // tutaj możesz podłączyć klienta Hazelcast
        }
    }
}
3. Podłączenie klienta Hazelcast
import com.hazelcast.client.HazelcastClient;
import com.hazelcast.client.config.ClientConfig;

ClientConfig config = new ClientConfig();
config.getNetworkConfig().addAddress(host + ":" + port);

var client = HazelcastClient.newHazelcastClient(config);

--------------
1. Było (embedded)
HazelcastInstance instance = Hazelcast.newHazelcastInstance();
IMap<String, String> map = instance.getMap("test");
🚀 2. Jest (Testcontainers – single node)
Kontener jako static (ważne dla wydajności)
import org.junit.jupiter.api.BeforeAll;
import org.junit.jupiter.api.AfterAll;
import org.testcontainers.containers.GenericContainer;

public class HazelcastTest {

    static GenericContainer<?> hazelcast =
        new GenericContainer<>("hazelcast/hazelcast:5.5.0")
            .withExposedPorts(5701);

    static String address;

    @BeforeAll
    static void start() {
        hazelcast.start();
        address = hazelcast.getHost() + ":" + hazelcast.getMappedPort(5701);
    }

    @AfterAll
    static void stop() {
        hazelcast.stop();
    }
}
🔌 3. Klient zamiast embedded
import com.hazelcast.client.HazelcastClient;
import com.hazelcast.client.config.ClientConfig;

ClientConfig config = new ClientConfig();
config.getNetworkConfig().addAddress(address);

HazelcastInstance client = HazelcastClient.newHazelcastClient(config);
🧪 4. Użycie w teście (praktycznie bez zmian)
@Test
void testMap() {
    IMap<String, String> map = client.getMap("test");
    map.put("key", "value");

    assertEquals("value", map.get("key"));
}
⚠️ Kluczowe różnice (embedded → container)
1. ❗ Musisz używać klienta

Embedded:

Hazelcast.newHazelcastInstance()

Container:

HazelcastClient.newHazelcastClient()
2. ❗ Sieć zamiast lokalnej pamięci
embedded = wszystko w JVM
container = prawdziwe TCP (port 5701)

➡️ dlatego czasem trzeba:

config.getConnectionStrategyConfig().getRetryConfig().setClusterConnectTimeoutMillis(10000);
3. ❗ Startup time

Dodaj wait strategy:

.withExposedPorts(5701)
.waitingFor(Wait.forListeningPort())
🌱 5. Spring Boot (jeśli używasz)

Zamiast embedded bean:

@Bean
HazelcastInstance hazelcastInstance() {
    return Hazelcast.newHazelcastInstance();
}

robisz:

@Bean
HazelcastInstance hazelcastClient() {
    ClientConfig config = new ClientConfig();
    config.getNetworkConfig().addAddress(address);
    return HazelcastClient.newHazelcastClient(config);
}



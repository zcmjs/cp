1. Przykładowy Serwis (bez CompletableFuture)

Załóżmy, że Twój serwis wygląda w ten sposób:
Java

import org.springframework.scheduling.annotation.Async;
import org.springframework.stereotype.Service;
// Importy np. dla KafkaTemplate

@Service
public class KafkaMessageService {

    private final KafkaTemplate<String, String> kafkaTemplate;

    public KafkaMessageService(KafkaTemplate<String, String> kafkaTemplate) {
        this.kafkaTemplate = kafkaTemplate;
    }

    @Async("customExecutor")
    public void sendMessage(String message) {
        // Tu dzieje się asynchroniczna wysyłka
        kafkaTemplate.send("moj-topic", message);
    }
}

2. Test z użyciem CountDownLatch i Mockito

W teście podmieniamy KafkaTemplate na mocka. Dzięki temu w momencie, gdy asynchroniczny wątek spróbuje wysłać wiadomość, zmusimy mocka do zapisania nazwy obecnego wątku (którym będzie wątek z Twojego @Async) i natychmiastowo zwolnimy blokadę dla testu głównego.
Java

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.mock.mockito.MockBean;

import java.util.concurrent.CountDownLatch;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicReference;

import static org.junit.jupiter.api.Assertions.assertNotEquals;
import static org.junit.jupiter.api.Assertions.assertTrue;
import static org.mockito.ArgumentMatchers.anyString;
import static org.mockito.ArgumentMatchers.eq;
import static org.mockito.Mockito.doAnswer;

@SpringBootTest
public class KafkaMessageServiceTest {

    @Autowired
    private KafkaMessageService kafkaMessageService;

    // Podmieniamy klasę wysyłającą na mocka, aby nie wysyłać prawdziwych wiadomości w teście
    @MockBean
    private KafkaTemplate<String, String> kafkaTemplate;

    @Test
    public void testAsyncMethodWithoutCompletableFuture() throws InterruptedException {
        // Zapisujemy nazwę wątku głównego testu
        String mainThreadName = Thread.currentThread().getName();
        
        // Zmienne do przechwycenia stanu z innego wątku
        AtomicReference<String> asyncThreadName = new AtomicReference<>();
        CountDownLatch latch = new CountDownLatch(1);

        // Konfigurujemy zachowanie mocka
        doAnswer(invocation -> {
            // Zapisujemy nazwę wątku, który wywołał tę metodę (nasz wątek async)
            asyncThreadName.set(Thread.currentThread().getName());
            
            // Zwalniamy blokadę - dajemy sygnał do głównego wątku, że może kontynuować
            latch.countDown();
            return null; // Metoda .send() to najczęściej void lub wrzuca CompletableFuture, tu dla uproszczenia null
        }).when(kafkaTemplate).send(anyString(), anyString());

        // Wywołujemy asynchroniczną metodę (uruchamia się w tle i natychmiast wraca)
        kafkaMessageService.sendMessage("Test Message");

        // Zatrzymujemy główny wątek na maksymalnie 2 sekundy, czekając aż async dojdzie do mocka
        boolean completedInTime = latch.await(2, TimeUnit.SECONDS);

        // Asercje
        assertTrue(completedInTime, "Metoda asynchroniczna nie wykonała się w zadanym czasie!");
        
        assertNotEquals(mainThreadName, asyncThreadName.get(), 
                "Metoda powinna wykonać się w innym wątku!");
        
        assertTrue(asyncThreadName.get().startsWith("CustomExecutorThread-"), 
                "Wątek powinien pochodzić z puli CustomExecutor!");
    }
}

Dlaczego to podejście jest idealne?

    Czystość domeny: Nie zanieczyszczasz warstwy domenowej ani serwisowej obiektami CompletableFuture wyłącznie dla celów testowych. Metoda pozostaje typu void.

    Bezpieczeństwo wielowątkowe w teście: Używamy AtomicReference do bezpiecznego przeniesienia nazwy wątku z wątku wykonawczego do wątku testowego.

    Brak sztucznych opóźnień: Wiele osób w takich przypadkach używa Thread.sleep(1000) w teście. Użycie CountDownLatch powoduje, że test przechodzi natychmiast po wykonaniu zadania, nawet jeśli zajęło to 5ms.

Grand Finale:
Testowanie asynchroniczne bez zwracania wartości wymaga użycia zatrzasku.

https://medium.com/@AlexanderObregon/how-spring-boot-configures-custom-thread-pools-for-async-processing-2f05d6fb3e42

------------------------
To świetne podejście! Dynamiczne dostosowanie liczby wątków konsumujących do zasobów sprzętowych to bardzo dobra praktyka w aplikacjach, które muszą być przenośne i skalowalne na różnych środowiskach.

Aby to osiągnąć w Spring Kafka, musisz utworzyć własnego beana typu ConcurrentKafkaListenerContainerFactory i wykorzystać klasę Runtime z języka Java do pobrania informacji o systemie.

Oto gotowy i optymalny sposób, jak to zapisać:
Definicja Beana Kafka Consumer Config
Java

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.kafka.annotation.EnableKafka;
import org.springframework.kafka.config.ConcurrentKafkaListenerContainerFactory;
import org.springframework.kafka.core.ConsumerFactory;

@Configuration
@EnableKafka
public class KafkaConsumerConfig {

    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, String> customKafkaListenerContainerFactory(
            ConsumerFactory<String, String> consumerFactory) {

        ConcurrentKafkaListenerContainerFactory<String, String> factory = 
                new ConcurrentKafkaListenerContainerFactory<>();
                
        // 1. Ustawienie fabryki konsumentów (zazwyczaj autowired przez Springa)
        factory.setConsumerFactory(consumerFactory);

        // 2. Pobranie liczby dostępnych rdzeni procesora w środowisku uruchomieniowym
        int coreCount = Runtime.getRuntime().availableProcessors();

        // 3. Ustawienie concurrency na liczbę rdzeni
        factory.setConcurrency(coreCount);

        return factory;
    }
}

Jak to działa krok po kroku:

    Runtime.getRuntime().availableProcessors(): Ta metoda zwraca liczbę procesorów logicznych (wątków procesora) dostępnych dla wirtualnej maszyny Javy. Jeśli uruchamiasz aplikację w kontenerze (np. Docker, Kubernetes), nowoczesne wersje Javy (powyżej 8u191) prawidłowo rozpoznają limity nałożone na kontener i zwrócą odpowiednią liczbę przydzielonych rdzeni, a nie rdzeni całego hosta fizycznego.

    setConcurrency(coreCount): Mówi frameworkowi Spring Kafka, ile niezależnych wątków konsumenta (pod maską klas KafkaMessageListenerContainer) ma zostać uruchomionych w ramach jednej adnotacji @KafkaListener, która używa tej fabryki.

Ważna uwaga architektoniczna (Kafka Partitions)

Zanim wdrożysz to na produkcję, pamiętaj o jednym kluczowym ograniczeniu samej architektury Kafki: jeden wątek konsumenta (concurrency) może czytać z jednej lub wielu partycji, ale jedna partycja nie może być czytana przez więcej niż jednego konsumenta w tej samej grupie (Consumer Group).

    Scenariusz Idealny: Masz topic z 8 partycjami i serwer z 8 rdzeniami. concurrency ustawi się na 8, a Kafka przydzieli dokładnie 1 partycję do 1 rdzenia.

    Scenariusz Zmarnowanych Zasobów: Masz topic z 4 partycjami, ale wdrażasz aplikację na maszynę z 16 rdzeniami. concurrency ustawi się na 16, ale 12 wątków będzie całkowicie bezczynnych (idle), ponieważ brakuje dla nich partycji.

Aby ta dynamika miała maksymalny sens, upewnij się, że liczba partycji na docelowych topicach Kafki jest równa lub większa niż maksymalna liczba rdzeni, z jakimi może zostać uruchomiona Twoja aplikacja. W przeciwnym razie warto dodać do kodu bezpiecznik Math.min(coreCount, maxPartitions).

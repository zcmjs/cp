


Stała pula wątków w Springu: Aby w Springu mieć stałą pulę wątków, możesz użyć @Bean w konfiguracji, aby utworzyć wspólny Executor lub ThreadPoolTaskExecutor. Konfiguracja wygląda następująco:

@Bean
public Executor taskExecutor() {
    ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
    executor.setCorePoolSize(5);
    executor.setMaxPoolSize(10);
    executor.setQueueCapacity(25);
    executor.setThreadNamePrefix("MyThreadPool-");
    executor.initialize();
    return executor;
}






import static org.junit.jupiter.api.Assertions.*;

import java.util.concurrent.CountDownLatch;
import java.util.concurrent.Executor;
import java.util.concurrent.TimeUnit;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;

@SpringBootTest
public class ThreadPoolTest {

    @Autowired
    private Executor taskExecutor;

    @Test
    public void testMultithreadingExecution() throws InterruptedException {
        // Liczba wątków do uruchomienia
        int numberOfThreads = 5;
        // CountDownLatch do synchronizacji
        CountDownLatch latch = new CountDownLatch(numberOfThreads);
        
        // Uruchamiamy wielowątkowe zadania
        for (int i = 0; i < numberOfThreads; i++) {
            taskExecutor.execute(() -> {
                try {
                    // Symulacja zadania
                    System.out.println("Zadanie wykonywane przez wątek: " + Thread.currentThread().getName());
                } finally {
                    // Zmniejszamy licznik
                    latch.countDown();
                }
            });
        }
        
        // Oczekujemy na zakończenie wszystkich zadań
        boolean completed = latch.await(5, TimeUnit.SECONDS);
        
        // Sprawdzamy, czy wszystkie wątki zakończyły pracę
        assertTrue(completed, "Nie wszystkie zadania zostały ukończone na czas.");
    }
}

public interface ProductRepository extends JpaRepository<Product, Long> {

    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("SELECT p FROM Product p WHERE p.id IN :ids")
    List<Product> findAllByIdWithLock(@Param("ids") List<Long> ids);
}

@Transactional: Adnotacja ta zapewnia, że cała metoda zostanie wykonana w ramach jednej transakcji, co jest kluczowe dla poprawnego działania blokowania.

Executors.newScheduledThreadPool(n) - Interfejs ScheduledExecutorService w Javie rozszerza interfejs ExecutorService i umożliwia planowanie zadań do wykonania po określonym czasie opóźnienia lub okresowo. Jest bardzo przydatny do zadań, które muszą być uruchamiane w regularnych odstępach czasu, takich jak zadania konserwacyjne, monitorowanie systemu, lub jakiekolwiek inne operacje cykliczne.

schedule - Służy do uruchomienia zadania po określonym czasie opóźnienia.


scheduleAtFixedRate - Służy do uruchomienia zadania okresowo z określoną stałą częstotliwością wraz z określonym opóźnieniem.



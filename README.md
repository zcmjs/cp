public interface ProductRepository extends JpaRepository<Product, Long> {

    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("SELECT p FROM Product p WHERE p.id IN :ids")
    List<Product> findAllByIdWithLock(@Param("ids") List<Long> ids);
}

@Transactional: Adnotacja ta zapewnia, że cała metoda zostanie wykonana w ramach jednej transakcji, co jest kluczowe dla poprawnego działania blokowania.

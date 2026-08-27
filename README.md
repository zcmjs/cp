1. Średni czas trwania (Average Duration)

Aby uzyskać średnią, musisz obliczyć przyrost (rate) sumy czasów i podzielić go przez przyrost liczby zdarzeń w określonym oknie czasowym.
Fragment kodu

rate(twoj_timer_seconds_sum[5m]) / rate(twoj_timer_seconds_count[5m])

Dlaczego 5 minut, a nie 30?
Proponowałeś okno 30-minutowe ([30m]). W architekturze mikrousług takie okno jest zazwyczaj zbyt szerokie. Mocno "wygładzi" ono wykres, ukrywając krótkotrwałe (np. 1-2 minutowe) anomalie, zatory w pulach wątków lub problemy z siecią. Interwał [5m] (lub nawet [1m]) daje znacznie lepszą widoczność rzeczywistych wahań wydajnościowych.
2. Maksymalny czas (Max)

Micrometer ma specyficzne podejście do metryki max. W przeciwieństwie do sum i count, max jest tzw. wskaźnikiem okienkowym (time-window rolling max). Oznacza to, że pokazuje najwyższą wartość odnotowaną w ostatnim, krótkim przedziale czasu (domyślnie opada do zera, jeśli ruch ustaje).
Fragment kodu

max by (instance) (twoj_timer_seconds_max)

Używamy max by (instance), aby zobaczyć maksymalny czas per instancja aplikacji na wypadek, gdybyś miał ich uruchomionych kilka za load balancerem.
3. Przepustowość (Throughput / RPS)

Mając do dyspozycji parametr count, warto od razu rzucić na wykres liczbę zapytań na sekundę. To często kluczowy kontekst przy analizowaniu skoków opóźnień.
Fragment kodu

rate(twoj_timer_seconds_count[5m])

    Krok dalej: Percentyle (p95, p99)
    Mierzenie średniej (sum / count) oraz max to dobry punkt wyjścia, ale w systemach rozproszonych o dużej współbieżności bywa zwodnicze. Średnia ukrywa ekstrema, a max jest podatny na jednorazowe anomalie (np. przerwa na Garbage Collection). Złotym standardem w branży jest mierzenie percentyli, takich jak p95 czy p99, wykorzystując funkcję histogram_quantile.

Kiedy używasz Timera z biblioteki Micrometer (np. za pomocą adnotacji @Timed lub tworząc go ręcznie), framework automatycznie rejestruje kilka powiązanych metryk w Prometheusie. Kropki z nazwy metryki (np. moja.funkcja.czas) są automatycznie zamieniane na podkreślenia (moja_funkcja_czas).

Dla każdego timera Micrometer generuje zazwyczaj trzy podstawowe szeregi czasowe, do których odnosisz się w PromQL:

    _count – całkowita liczba wywołań funkcji,

    _sum – łączny czas trwania wszystkich wywołań,

    _max – maksymalny czas wykonania w danym oknie czasowym.

Załóżmy, że Twój timer w kodzie nazywa się metoda_czas. Oto jak powinny wyglądać zapytania (PromQL) w Grafanie, aby wyświetlić najprzydatniejsze statystyki:
1. Średni czas wykonania (Average Response Time)

Aby uzyskać średni czas wykonania, musisz podzielić tempo wzrostu sumy czasu przez tempo wzrostu liczby wywołań (np. z ostatnich 5 minut). To najpopularniejszy wykres na dashboardach.
Fragment kodu

rate(metoda_czas_sum[5m]) / rate(metoda_czas_count[5m])

2. Liczba wywołań na sekundę (Throughput / RPS)

Jeśli chcesz wiedzieć, jak często funkcja jest wywoływana (np. w wywołaniach na sekundę), sprawdzasz tempo przyrostu metryki _count.
Fragment kodu

rate(metoda_czas_count[5m])

3. Maksymalny czas wykonania (Max Duration)

Micrometer domyślnie publikuje wartość maksymalną w formie wskaźnika opadającego z czasem, więc możesz odpytać go bezpośrednio, bez funkcji rate().
Fragment kodu

metoda_czas_max

4. Percentyl 95 (95th Percentile) – Opcjonalnie

Średnia często ukrywa anomalie (np. pojedyncze bardzo wolne żądania). Znacznie lepszą miarą jest percentyl 95 (oznaczający, że 95% żądań było szybszych niż X).
Uwaga: Aby to zapytanie zadziałało, musisz w konfiguracji Spring Boota (lub w builderze timera) włączyć publikowanie histogramów: management.metrics.distribution.percentiles-histogram.http.server.requests=true (lub dla własnej metryki).
Fragment kodu

histogram_quantile(0.95, sum(rate(metoda_czas_bucket[5m])) by (le))

Wskazówka do Grafany: Zwróć uwagę na jednostki. Micrometer w Spring Boot domyślnie raportuje czas w sekundach. Upewnij się, że w ustawieniach panelu w Grafanie (w sekcji Standard options -> Unit) wybierzesz Seconds (sekundy), aby Grafana poprawnie przeskalowała wartości do milisekund czy minut.

Grand Finale: PromQL idealnie pokazuje czas działania wybranych funkcji.


Masz tutaj dwa wyjścia – jedno po stronie języka zapytań (PromQL), a drugie (znacznie lepsze architektonicznie) po stronie kodu aplikacji w Spring Boot.
1. Rozwiązanie doraźne: Regex w PromQL (filtrowanie po nazwie)

Jeśli Twoje metryki już zostały wygenerowane z różnymi nazwami (np. scheduler_time_execution_metodaA_sum, scheduler_time_execution_metodaB_sum), możesz odpytać je za pomocą ukrytej etykiety __name__ oraz wyrażenia regularnego =~.

Aby na jednym wykresie wyświetlić średni czas dla wszystkich tych metod, użyj zapytania:
Fragment kodu

rate({__name__=~"scheduler_time_execution.*_sum"}[5m])
/
rate({__name__=~"scheduler_time_execution.*_count"}[5m])

Grafana automatycznie rozdzieli te metryki na osobne linie na podstawie ich pełnych nazw.
2. Rozwiązanie docelowe: Użycie tagów (Labels) w Micrometer

Tworzenie unikalnej nazwy metryki dla każdej oddzielnej metody to tzw. metric name explosion. Jest to antywzorzec, który w dużych systemach mocno obciąża bazę Prometheusa.

Zamiast tego, w Spring Boot używa się jednej nazwy metryki dla wszystkich schedulerów, a poszczególne metody rozróżnia się za pomocą tagów (etykiet). W kodzie, w adnotacji @Timed, wystarczy dodać parametr extraTags:
Java

@Timed(value = "scheduler_time_execution", extraTags = {"method", "synchronizeData"})
public void synchronizeData() { ... }

@Timed(value = "scheduler_time_execution", extraTags = {"method", "cleanupDb"})
public void cleanupDb() { ... }

Dzięki temu Twoje zapytania w Grafanie stają się znacznie czystsze. Aby policzyć średnią dla każdej z metod osobno, po prostu grupujesz wyniki po dodanym tagu method, używając klauzuli by (method):
Fragment kodu

sum by (method) (rate(scheduler_time_execution_sum[5m]))
/
sum by (method) (rate(scheduler_time_execution_count[5m]))

Ostrzeżenie przed pułapką uśredniania

Gdy masz do czynienia z wieloma metodami i różnymi czasami ich trwania, musisz uważać na to, jak interpretujesz agregacje. Jeśli metoda A przez chwilę działała bardzo wolno (np. 100s), a metoda B wykonała się błyskawicznie (10s), to wrzucenie ich do jednego uśrednionego "worka" da wynik 55s. To całkowicie ukrywa prawdę o rzeczywistym skoku wydajności. Z zewnątrz system wydaje się działać "akceptowalnie", podczas gdy jedna z funkcji ma poważny problem.

Aby uniknąć tego błędu, zawsze rozdzielaj ruch po tagach (jak pokazano wyżej), a do wyłapywania takich incydentów dodatkowo monitoruj wartości maksymalne zgrupowane po metodach:
Fragment kodu

max by (method) (scheduler_time_execution_max)

Grand Finale: Używaj tagów do precyzyjnego monitorowania wielu metod.

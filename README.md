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

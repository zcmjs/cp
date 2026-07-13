  # --- 2. Kafka UI ---
  kafka-ui:
    image: provectuslabs/kafka-ui
    container_name: kafka-ui
    depends_on:
      - broker
    ports:
      - "7777:8080"
    restart: always
    environment:
      - KAFKA_CLUSTERS_0_NAME=broker
      - KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS=broker:29092
      # UI w trybie KRaft nie musi podawać adresu Zookeepera

  # --- 3. Kafka Exporter (Wystawia metryki JMX z Kafki, aby klient mógł się do nich podłączyc) ---
  kafka-exporter:
    image: danielqsj/kafka-exporter:latest
    container_name: kafka-exporter
    depends_on:
      broker:
        condition: service_healthy
    command:
      # Łączymy się z brokerem po wewnętrznym adresie sieci Docker
      - --kafka.server=broker:29092
    ports:
      - "9308:9308"
    restart: always

  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ./config/prometheus.yml:/etc/prometheus/prometheus.yml
    extra_hosts:
      - "host.docker.internal:host-gateway"
    depends_on:
      - kafka-exporter

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    ports:
      - "3000:3000"
    environment:
      GF_SECURITY_ADMIN_USER: admin
      GF_SECURITY_ADMIN_PASSWORD: admin
    volumes:
      # Wolumen potrzebny, abyś nie tracił dashboardów po restarcie kontenera
      - grafana_data:/var/lib/grafana
    depends_on:
      - prometheus
    restart: always

volumes:
  grafana_data:

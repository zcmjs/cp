`spring:
  quartz:
    job-store-type: jdbc
    properties:
      org:
        quartz:
          scheduler:
            instanceId: AUTO
            instanceName: ClusteredScheduler
          jobStore:
            isClustered: true
            clusterCheckinInterval: 20000
            class: org.quartz.impl.jdbcjobstore.JobStoreTX
            driverDelegateClass: org.quartz.impl.jdbcjobstore.StdJDBCDelegate
            tablePrefix: QRTZ_`

            



            Zadanie z @DisallowConcurrentExecution:

import org.quartz.DisallowConcurrentExecution;
import org.quartz.Job;
import org.quartz.JobExecutionContext;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

@DisallowConcurrentExecution
public class MyQuartzJob implements Job {

    private static final Logger logger = LoggerFactory.getLogger(MyQuartzJob.class);

    @Override
    public void execute(JobExecutionContext context) {
        logger.info("Starting job: {}", context.getJobDetail().getKey());
        try {
            // Symulacja długotrwałego zadania
            Thread.sleep(10000);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            logger.error("Job interrupted", e);
        }
        logger.info("Finished job: {}", context.getJobDetail().getKey());
    }
}

Działanie:

    Jeśli zadanie jest uruchamiane np. co 5 sekund, a trwa 10 sekund, drugie wywołanie zostanie wstrzymane, dopóki pierwsze się nie zakończy.

Konfiguracja Spring Quartz w aplikacji:

W Spring Quartz konfiguracja może być wykonana za pomocą Spring Boot, wykorzystując SchedulerFactoryBean. Oto przykład konfiguracji:

import org.quartz.*;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class QuartzConfig {

    @Bean
    public JobDetail myJobDetail() {
        return JobBuilder.newJob(MyQuartzJob.class)
                .withIdentity("myJob", "group1")
                .storeDurably()
                .build();
    }

    @Bean
    public Trigger myJobTrigger() {
        return TriggerBuilder.newTrigger()
                .forJob(myJobDetail())
                .withIdentity("myTrigger", "group1")
                .withSchedule(SimpleScheduleBuilder.simpleSchedule()
                        .withIntervalInSeconds(5) // Co 5 sekund
                        .repeatForever())
                .build();
    }
}

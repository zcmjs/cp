import org.testcontainers.containers.GenericContainer;
import org.testcontainers.utility.MountableFile;

import java.nio.file.Files;
import java.nio.file.StandardCopyOption;
import java.util.Set;
import java.util.jar.JarEntry;
import java.util.jar.JarFile;

Set<String> wanted = Set.of(
    "lib-a.jar",
    "lib-b.jar",
    "lib-c.jar"
);

GenericContainer<?> container =
    new GenericContainer<>("hazelcast/hazelcast:5.3");

try (JarFile jar = new JarFile("app.jar")) {

    jar.stream()
        .filter(e -> e.getName().startsWith("BOOT-INF/lib/"))
        .forEach(e -> {
            String fileName = e.getName()
                .substring("BOOT-INF/lib/".length());

            if (!wanted.contains(fileName)) return;

            try {
                var temp = Files.createTempFile("lib-", ".jar");

                try (var in = jar.getInputStream(e)) {
                    Files.copy(in, temp, StandardCopyOption.REPLACE_EXISTING);
                }

                container.withCopyFileToContainer(
                    MountableFile.forHostPath(temp.toString()),
                    "/opt/hazelcast/lib/" + fileName
                );

            } catch (Exception ex) {
                throw new RuntimeException(ex);
            }
        });
}

container.start();

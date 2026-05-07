import org.testcontainers.containers.GenericContainer;
import org.testcontainers.utility.MountableFile;

import java.util.jar.JarEntry;
import java.util.jar.JarFile;

public class HazelcastTestContainer {

    public static void main(String[] args) throws Exception {

        String jarPath = "app.jar";

        GenericContainer<?> hz =
            new GenericContainer<>("hazelcast/hazelcast:5.3");

        try (JarFile jar = new JarFile(jarPath)) {

            jar.stream()
                .filter(e -> e.getName().startsWith("BOOT-INF/lib/") && e.getName().endsWith(".jar"))
                .forEach(e -> {
                    try {
                        String fileName = e.getName().substring("BOOT-INF/lib/".length());

                        MountableFile mf = MountableFile.forHostPath(
                            extractToTempFile(jar, e)
                        );

                        hz.withCopyFileToContainer(
                            mf,
                            "/opt/hazelcast/lib/" + fileName
                        );

                    } catch (Exception ex) {
                        throw new RuntimeException(ex);
                    }
                });
        }

        hz.start();
    }

    static String extractToTempFile(JarFile jar, JarEntry entry) throws Exception {

        var temp = java.nio.file.Files.createTempFile("lib-", ".jar");

        try (var in = jar.getInputStream(entry)) {
            java.nio.file.Files.copy(in, temp, java.nio.file.StandardCopyOption.REPLACE_EXISTING);
        }

        return temp.toAbsolutePath().toString();
    }
}

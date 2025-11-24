app:
  items:
    X:
      true: 21
      false: 37

@Component
@ConfigurationProperties(prefix = "app")
public class AppItemsProperties {

    private Map<String, Map<Boolean, Integer>> items = new HashMap<>();

    public Map<String, Map<Boolean, Integer>> getItems() {
        return items;
    }

    public void setItems(Map<String, Map<Boolean, Integer>> items) {
        this.items = items;
    }

    @PostConstruct
    public void init() {
        System.out.println("Załadowane dane: " + items);
    }

    /** pomocnicza metoda */
    public Integer getValue(String name, boolean available) {
        Map<Boolean, Integer> availabilityMap = items.get(name);
        if (availabilityMap == null) {
            return null;
        }
        return availabilityMap.get(available);
    }
}


==========================

@Component
@ConfigurationProperties(prefix = "app")
public class AppItemsProperties {

    private Map<String, Map<Boolean, Integer>> items;

    public Map<String, Map<Boolean, Integer>> getItems() {
        return items;
    }

    public void setItems(Map<String, Map<Boolean, Integer>> items) {
        this.items = items;
    }

    public Integer getValue(String name, boolean available) {
        Map<Boolean, Integer> map = items.get(name);
        if (map == null) {
            return null;  // lub rzuć wyjątek
        }
        return map.get(available);
    }
}

================================================================================
 System.out.println(props.getValue("X", true));   // 21
    System.out.println(props.getValue("X", false));  // 37

    =================================================================================
TEST:
Jeśli chcesz mieć osobny YAML specjalnie do testu, dodaj np.:

src/test/resources/application-test.yml:

app:
  items:
    X:
      true: 21
      false: 37
    Y:
      true: 100
      false: 200
==========================================================
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.ActiveProfiles;

import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest
@ActiveProfiles("test")  // <— ładuje application-test.yml
class AppItemsPropertiesTest {

    @Autowired
    private AppItemsProperties props;

    @Test
    void shouldReturnCorrectValuesForNameAndAvailability() {
        // given
        String name = "X";

        // when
        Integer valueWhenTrue = props.getValue(name, true);
        Integer valueWhenFalse = props.getValue(name, false);

        // then
        assertThat(valueWhenTrue).isEqualTo(21);
        assertThat(valueWhenFalse).isEqualTo(37);
    }

    @Test
    void shouldLoadWholeStructure() {
        assertThat(props.getItems()).isNotEmpty();
        assertThat(props.getItems().get("X")).containsEntry(true, 21);
        assertThat(props.getItems().get("X")).containsEntry(false, 37);
    }
}
===============================================================

    @Test
    void shouldBindYamlAndReturnCorrectValues() {
        contextRunner.run(context -> {
            AppItemsProperties props = context.getBean(AppItemsProperties.class);

            Integer valueTrue = props.getValue("X", true);
            Integer valueFalse = props.getValue("X", false);

            assertThat(valueTrue).isEqualTo(21);
            assertThat(valueFalse).isEqualTo(37);

            // sprawdzamy drugą nazwę
            assertThat(props.getValue("Y", true)).isEqualTo(100);
        });
    }

    @Test
    void shouldLoadStructureIntoMap() {
        contextRunner.run(context -> {
            AppItemsProperties props = context.getBean(AppItemsProperties.class);

            assertThat(props.getItems()).containsKey("X");
            assertThat(props.getItems().get("X")).containsEntry(true, 21);
            assertThat(props.getItems().get("X")).containsEntry(false, 37);
        });
    }

  ==================================================================================
  


    


====================

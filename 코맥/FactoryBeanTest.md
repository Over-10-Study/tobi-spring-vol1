- 설정 파일

```java
@Configuration
public class MessageConfig {

    @Bean(name = "text")
    public MessageFactoryBean messageFactory() {
        MessageFactoryBean factory = new MessageFactoryBean();
        factory.setText("TEST");
        return factory;
    }

    @Bean
    public Message message() throws Exception {
        return messageFactory().getObject();
    }
}
```

- 테스트 파일

```java
@ExtendWith(SpringExtension.class)
@ContextConfiguration(classes = MessageConfig.class)
public class MessageTest {

    @Autowired
    private Message message;

    @Resource(name = "text")
    private MessageFactoryBean messageFactoryBean;

    @Test
    void 싱글톤_테스트() {
        try {
//            assertThat(messageFactoryBean.getObject()).isEqualTo(messageFactoryBean.getObject());
            assertThat(messageFactoryBean.getObject() == messageFactoryBean.getObject()).isTrue();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    @Test
    void test() {
        try {
            assertThat(messageFactoryBean.getObject().getText()).isEqualTo("TEST");
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

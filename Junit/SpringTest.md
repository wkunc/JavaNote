# 一般环境集成Spring
使用Junit4进行测试
@RunWith(SpringJUnit4ClassRunner.class)

@ContextConfiguration 加载配置文件 既可以加载xml 也可以加载class

@ContextHierarchy可以包含多个@ContextConfiguration 以达到同时加载多个ioc容器的复杂环境

@DirtiesContext 表明这是一个会污染ioc容器环境的测试，让spring在测试结束后从新加载

web测试
```java
@DirtiesContext
//@ContextConfiguration(locations={"classpath:spring-mybatis.xml"})
@RunWith(SpringJUnit4ClassRunner.class)
//用来加载web环境
@ContextHierarchy({
        @ContextConfiguration(locations = "classpath:Springmvc.xml"),
        @ContextConfiguration(locations = "classpath:spring-mybatis.xml")
})
@WebAppConfiguration("classpath:src/main/resources")
public class WebTest {
    @Autowired
    private WebApplicationContext wac;
    private MockMvc mockMvc;

    @Before
    public void setup(){
        this.mockMvc = MockMvcBuilders.webAppContextSetup(this.wac).build();
    }

    ...测试方法
}
```java
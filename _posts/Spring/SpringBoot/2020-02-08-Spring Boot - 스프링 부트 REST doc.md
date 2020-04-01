---
title:  "Spring Boot - 스프링 부트 Rest doc!"

read_time: false
share: false
author_profile: false
classes: wide

categories:
  - Spring

tags:
  - Spring
  - java

toc: true

---

## Rest doc

협업하다 보면 서버간의 API호출은 빈번하게 일어나고 해당 API가 어떤 역할을 하는 API인지 문서화는 필수이다.  

일일이 google 공유 document를 생성하여 협업하는 경우도 있지만 프로젝트가 커질수로 관리는 어려워지고 최신화 및 동기화가 힘들어진다.  

`Spring Rest doc`를 사용하면 이런 문제점을 일부 해결해준다.  

spring-project 깃허브에서 제공하는 샘플 프로젝트로 간단히 테스트해보자.  

> https://github.com/spring-projects/spring-restdocs/tree/master/samples/junit5

> 현재 스프링 버전은 `2.2.4.RELEASE`이기에 `junit5`를 사용한다.  

maven으로 생성한 프로젝트의 pom.xml에 rest doc dependency를 추가하자.  

```xml
<!--rest doc dependency-->
<dependency>
    <groupId>org.springframework.restdocs</groupId>
    <artifactId>spring-restdocs-mockmvc</artifactId>
    <scope>test</scope>
</dependency>
```

플러그인은 아래처럼 설정하였다.  

```xml
<plugins>
  <plugin>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-maven-plugin</artifactId>
  </plugin>
  <!--rest doc plugin-->
  <plugin>
    <groupId>org.asciidoctor</groupId>
    <artifactId>asciidoctor-maven-plugin</artifactId>
    <version>1.5.8</version>
    <executions>
      <execution>
        <id>generate-docs</id>
        <phase>prepare-package</phase>
        <goals>
          <goal>process-asciidoc</goal>
        </goals>
        <configuration>
          <backend>html</backend>
          <doctype>book</doctype>
        </configuration>
      </execution>
    </executions>
    <dependencies>
      <dependency>
        <groupId>org.springframework.restdocs</groupId>
        <artifactId>spring-restdocs-asciidoctor</artifactId>
        <version>${spring-restdocs.version}</version>
      </dependency>
    </dependencies>
  </plugin>
  <!--생성된 문서를 jar파일에 추가-->
  <plugin>
    <artifactId>maven-resources-plugin</artifactId>
    <version>2.7</version>
    <executions>
      <execution>
        <id>copy-resources</id>
        <phase>prepare-package</phase>
        <goals>
          <goal>copy-resources</goal>
        </goals>
        <configuration>
          <outputDirectory>
            ${project.build.outputDirectory}/static/docs
            <!-- target/classes/static/docs 아래에 rest doc 파일들 생성 -->
          </outputDirectory>
          <resources>
            <resource>
              <directory>
                ${project.build.directory}/generated-docs
              </directory>
            </resource>
          </resources>
        </configuration>
      </execution>
    </executions>
  </plugin>
</plugins>
```

테스트를 위한 간단한 컨트롤러 클래스 작성 

```java
@Controller
@RequestMapping("/product")
public class ProductController {

    @GetMapping("/{id}")
    public ResponseEntity<?> getProduct(@PathVariable Integer id) {
        ProductDTO productDTO = new ProductDTO();
        productDTO.setId(id);
        productDTO.setName("Spring Boot");
        productDTO.setDesc("Spring");
        productDTO.setQuantity(10);
        return new ResponseEntity<>(productDTO, HttpStatus.OK);
    }
}

@Getter
@Setter
class ProductDTO {
    private Integer id;
    private String name;
    private String desc;
    private Integer quantity;
}
```

`localhost:8080/product/{id}` url을 호출하면 고정해둔 문자열을 세팅해둔 객체를 json형식으로 반환하는 간단한 컨트롤러 메서드이다.  

그리고 이를 테스트하는 코드를 작성하자 (위의 git url 참고)

```java
@SpringBootTest
@ExtendWith({RestDocumentationExtension.class, SpringExtension.class}) 
public class ProductControllerTest {

    private RestDocumentationResultHandler document;

    @Autowired
    private WebApplicationContext context;

    private MockMvc mockMvc;

    @BeforeEach
    public void setUp(RestDocumentationContextProvider restDocumentation) {
        this.document = document(
                "{class-name}/{method-name}",
                preprocessResponse(prettyPrint()) //json 문자열 줄맞춤
        );
        this.mockMvc = MockMvcBuilders.webAppContextSetup(this.context)
                .apply(documentationConfiguration(restDocumentation))
                .alwaysDo(document)
                .build();
    }

    @Test
    public void getProduct() throws Exception {
        mockMvc.perform(
                get("/product/{id}", 1)
                        .accept(MediaType.APPLICATION_JSON))
                .andDo(print())
                .andExpect(status().isOk())
                .andDo(document.document(
                        pathParameters(parameterWithName("id").description("Product's Id")),
                        responseFields(
                                fieldWithPath("id").description("product's Id"),
                                fieldWithPath("name").description("product's name"),
                                fieldWithPath("desc").description("product's desc"),
                                fieldWithPath("quantity").description("product's quantity")
                        )
                ))
                .andExpect(jsonPath("$.name", is(notNullValue())))
                .andExpect(jsonPath("$.desc", is(notNullValue())))
                .andExpect(jsonPath("$.quantity", is(notNullValue())));
    }
}
```

테스트 코드를 실행하고 `target/generated-snippets`에 `rest-doc`가 생성 되는지 확인  

![restdoc1]({{ "/assets/2020/restdoc1.png" | absolute_url }}){: .shadow}  

작성된 파일을 하나의 파일로 합칠수 있도록 `src/main/asciidoc` 아래에 생성된 restdoc들의 위치를 참조하는 문서 작성  
`product-controller.adoc` 이름으로 하나 생성하였다.   

```
= ProductController

== getProduct

include::{snippets}/product-controller-test/get-product/path-parameters.adoc[]
include::{snippets}/product-controller-test/get-product/http-response.adoc[]
include::{snippets}/product-controller-test/get-product/response-fields.adoc[]
include::{snippets}/product-controller-test/get-product/curl-request.adoc[]
include::{snippets}/product-controller-test/get-product/http-request.adoc[]
include::{snippets}/product-controller-test/get-product/httpie-request.adoc[]
include::{snippets}/product-controller-test/get-product/request-body.adoc[]
include::{snippets}/product-controller-test/get-product/response-body.adoc[]
```

생성하후 아래 maven명령 실행 


```
mvn clean
mvn install
```

테스트가 진행된후 `rest doc`가 생성되고 `src/main/asciidoc` 에 생성해둔 파일양식대로 하나의 API 문서를 `pom.xml`에 설정해둔 위치에 생성하는지 확인.   

이후 jar파일을 실행하던 개발툴을 통해 서버 실행 후 `localhost:8080/docs/product-controller.html` 요청

아래처럼 출력되는지 확인  
![restdoc2]({{ "/assets/2020/restdoc2.png" | absolute_url }}){: .shadow}  

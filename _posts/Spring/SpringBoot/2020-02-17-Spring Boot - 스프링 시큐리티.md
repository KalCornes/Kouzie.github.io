---
title:  "Spring Boot - 스프링 부트 시큐리티!"

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

## Spring Securitu

> Spring doc : https://docs.spring.io/spring-security/site/docs/5.2.3.BUILD-SNAPSHOT/reference/htmlsingle/


이전에 `Spring Framework` 에서 시큐리티를 다룬적이 있다.  
> https://kouzie.github.io/spring/Spring-%EC%8A%A4%ED%94%84%EB%A7%81-%EC%8B%9C%ED%81%90%EB%A6%AC%ED%8B%B0/#

스프링 부트에선 어떻게 변했는지 확인해보자.  

프로젝트에 `dependency`를 하나 추가하고 config용 java파일을 하나 설정하자.  
```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```

컨트롤러를 아무거나 추가해서 실행해보자.  

아래 문구가 출력된다.  

```
Using generated security password: 60e8b37d-147a-4174-9003-3ca02800aada
```

컨트롤러와 뷰페이지 하나 생성후 접근해보자.   
아래의 이미지처럼 로그인 페이지가 생성된다.  

![springboot_security1]({{ "/assets/springboot/springboot_security1.png" | absolute_url }}){: .shadow}  

아이디에는 `user`, 비밀번호에는 위에 출력된 `security password` 를 입력해야한다.  

기본적으로 모든 `request` 에 `security filter chain`을 적용한다.  

아래처럼 아무런 필터도 설정 못하도록 `WebSecurityConfigurerAdapter`의 `configure`를 공백으로 처리해 필터를 생략해놓고 후에 하나씩 추가해보자.  


```java
@Log
@EnableWebSecurity
public class SecurityConfig extends WebSecurityConfigurerAdapter {
    @Override
    protected void configure(HttpSecurity http) {
        log.info("security config....");
    }
}
```

## 멤버 클래스 설계  

```java
@Getter
@Setter
@Entity
@Table(name = "tbl_members")
public class Member {
    @Id
    private String uid;

    private String upw;
    private String uname;

    @OneToMany(mappedBy = "member", fetch = FetchType.EAGER)
    List<MemberRole> roles;

    @CreationTimestamp
    private LocalDateTime regdate;
    @UpdateTimestamp
    private LocalDateTime updatedate;
}

@Getter
@Setter 
@Entity
@Table(name = "tbl_member_role")
public class MemberRole {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long fno;

    private String roleName;

    @ManyToOne
    @JoinColumn(name = "member")
    @JsonIgnore
    Member member;
}
```

Member는 여러개의 role을 가질 수 있는 N:1 관계.  


## 로그인/로그아웃 필터 처리

특정 권한을 가진 유저만 `request` 요청을 허용하고 싶을때 `configure` 메서드에 필터를 등록해 처리할 수 있다.  

또한 테스트 멤버용 데이터를 `AuthenticationManagerBuilder`의 `inMemoryAuthentication`로 메모리에 등록 후 테스트 진행이 가능하다.  

```java
@Log
@EnableWebSecurity
public class SecurityConfig extends WebSecurityConfigurerAdapter {

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        log.info("SecurityConfig configure...");
        http.authorizeRequests()
                .antMatchers("/boards/list").permitAll()
                .antMatchers("/boards/register").hasAnyRole("BASIC", "MANAGER", "ADMIN");
        http.formLogin();
    }

    @Autowired
    public void configureGlobal(AuthenticationManagerBuilder auth) throws Exception {
        log.info("SecurityConfig configureGlobal...");
        auth.inMemoryAuthentication()
                .withUser("manager")
                .password("{noop}1111")
                .roles("MANAGER");
    }
}
```

> 스프링 5.0 버전 이상부턴 입력된 패스워드를 `PasswordEncoder`를 통해 인코딩 후 비교한다고 한다. `{noop}`을 앞에 붙여 해당 과정을 생략한다.  

id 는 `manger`, 비밀번호는 `1111` 로 설정  

`boards/register` 로 이동해 `AuthenticationManagerBuilder` 으로 생성한 테스트 데이터가 작동하는지 확인하자.  

> 테스트로 db안의 데이터를 사용하고 싶다면 아래처럼 설정  

```java
@Autowired
DataSource datasource;
...

@Autowired
public void configureGlobal(AuthenticationManagerBuilder auth) throws Exception {

  log.info("SecurityConfig configureGlobal...");

  //enable 은 해당 계정 사용가능 여부
  String query1 = "SELECT uid username, CONCAT('{noop}', upw) password, true enabled FROM tbl_members WHERE uid = ?"; 
  String query2 = "SELECT member uid, role_name role FROM tbl_member_role WHERE member = ?";
  auth.jdbcAuthentication()
    .dataSource(datasource)
    .usersByUsernameQuery(query1)
    .authoritiesByUsernameQuery(query2)
    .rolePrefix("ROLE_");
}
```

`Spring security login` 페이지에서 중요한건 필터가 요구하는 데이터의 `alias`이다.  
`user`에 대한 데이터를 조회할땐 `username`, `password`, `enabled`  
`role`에 대한 데이터를 조회할땐 `uid`, `role` 


> `rolePrefix("ROLE_")` : Spring security 는 롤 베이스 기반 보안정책을 제공하며 기본적으로 `ROLE_`접두사가 기본적으로 붙이도록 설정함.  
공식문서에선 `RoleVoter`를 커스텀하면 접두사 변경이 가능하다 하는데 커스텀이 쉽지 않다...  
https://javadeveloperzone.com/spring-boot/spring-security-custom-rolevoter-example/



### 커스텀 로그인, 로그아웃, 접근제한 페이지 

```java
@Override
protected void configure(HttpSecurity http) throws Exception {
    log.info("SecurityConfig configure...");
    http.authorizeRequests()
            .antMatchers("/boards/list").permitAll()
            .antMatchers("/boards/register").hasAnyRole("BASIC", "MANAGER", "ADMIN");
    http.formLogin();

    http.csrf().disable().formLogin().loginPage("/login");
    //.loginProcessingUrl("")
    //.successForwardUrl("")
    //.failureForwardUrl("")
    // 로그인 데이터 post 로 전달할 url 변경, 로그인 success, failure 리다이렉트 페이지 변경

    //.usernameParameter("user_id")
    //.passwordParameter("user_pw")
    // login 요청시 사용 파라미터 명

    // login

    http.exceptionHandling().accessDeniedPage("/accessDenied");

    http.logout().logoutUrl("/logout").invalidateHttpSession(true);

    http.userDetailsService(customUsersService);
}

@Controller
public class LoginController {

    @GetMapping("/login")
    public void login() {
      // resource/template 밑에 login.html 생성 요망 
    }

    @GetMapping("/logout")
    public void logout() {

    }

    @GetMapping("/accessDenied")
    public void accessDenied() {

    }
}
```



### userDetailsService

간단한 데이터베이스 조회 후 인증작업을 거치려면 `AuthenticationManagerBuilder`의 `jdbcAuthentication`를 사용하면 되지만  
커스텀 인증과정을 거치려면 `userDetailsService` 를 사용해야 한다.  

`UserDetailsService`를 상속받는 서비스를 정의  

```java
@Service
@Log
public class CustomUsersService implements UserDetailsService {

    @Autowired
    MemberRepository memberRepository;

    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        memberRepository.findById(username);
        return null;
    }
}
```

어떻게 `Member` 객체를 `Spring security` 에서 요구하는 `UserDetails` 로 반환하는지 알아보자.   

방법은 3가지다.  

1. `Member` 를 `UserDetails`의 구현체로 만드는것.  
2. `Member` 를 `UserDetails`의 구현체인 `User`의 하위객체로 만드는것.  
3. `User`의 정보 `Member`의 정보 2개를 포함하는 새로운 객체를 정의  

3번 방법을 사용해 `UserDetails` 를 반환해보자.  

```java
import org.springframework.security.core.userdetails.User;

@Getter
@Setter
public class CustomSecurityUser extends User {

    private static final String ROLE_PREFIX = "ROLE_";

    private Member member;
    public CustomSecurityUser(Member member) {
        super(member.getUid(), "{noop}" + member.getUpw(), makeGrantedeAuth(member.getRoles()));
        this.member = member;
    }

    private static List<GrantedAuthority> makeGrantedeAuth(List<MemberRole> roles) {
        List<GrantedAuthority> list = new ArrayList<>();
        roles.forEach(
                memberRole -> list.add(new SimpleGrantedAuthority(ROLE_PREFIX + memberRole.getRoleName()))
        );
        return list;
    }
}

@Service
@Log
public class CustomSecurityUsersService implements UserDetailsService {

    @Autowired
    MemberRepository memberRepository;

    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        Member member = memberRepository.findById(username);
        return new CustomSecurityUser(member);
    }
}
```

#### 컨트롤러에서 로그인 정보 접근  

```java
@GetMapping("/list")
@Transactional
public void list(
    Authentication authentication,
    @ModelAttribute("pageVO") PageVO vo,
    Model model
) {
  log.info("list() called");
  Pageable page = vo.makePageable(0, "bno");
  Page<Object[]> result = customCrudRepository.getCustomPages(vo.getType(), vo.getKeyword(), page);

  if (authentication != null) {
    CustomSecurityUser customSecurityUser = (CustomSecurityUser) authentication.getPrincipal();
    log.info("meber: " + customSecurityUser.getMember());
  }
  model.addAttribute("result", new PageMaker(result));
}
```

### 로그인 정보 표시

현재 `thymeleaf`를 통해 뷰 페이지를 출력하고 있으며 시큐리티에 대한 태그를 사용하려면 `thymeleaf-extras-springsecurity5` 의존성을 추가해야 한다.  

```xml
<!-- 현재 사용중인 의존객체들 -->
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-thymeleaf</artifactId>
</dependency>
<dependency>
  <groupId>nz.net.ultraq.thymeleaf</groupId>
  <artifactId>thymeleaf-layout-dialect</artifactId>
  <version>2.4.1</version>
</dependency>
<dependency>
  <groupId>org.thymeleaf.extras</groupId>
  <artifactId>thymeleaf-extras-springsecurity5</artifactId>
  <version>3.0.4.RELEASE</version>
</dependency>
```

```jsp
<div class="page-header">
  <h1>Boot06 Project <small>for Spring MVC + JPA</small></h1>
  <h3 sec:authentication="name">Spring seucurity username</h3>
  <h3>[[${#authentication.name}]]</h3>
  <h3 sec:authorize="hasRole('ROLE_ADMIN')">This Conetent Only For ADMIN</h3>
  <h3 sec:authorize="hasRole('ROLE_MANAGER')">This Conetent Only For MANAGER</h3>
  <h3 sec:authorize="hasRole('ROLE_BASIC')">This Conetent Only For BASIC</h3>
  <h3 sec:authorize="hasAnyRole('ROLE_ADMIN', 'ROLE_MANAGER', 'ROLE_BASIC')">This Conetent For Everyone</h3>
</div>
```

![springboot_security2]({{ "/assets/springboot/springboot_security2.png" | absolute_url }}){: .shadow}  


> `Role`(역할) 과 `Authority`(권한): 스프링 프레임워크에선 둘의 기능은 같다. 하지만 다른 프레임워크에선 다르게 쓰이는 용어일 수 있다. 
> `Authority`가 좀더 세세한 의미로 관리자는 모두 `ROLE_ADMIN` 이란 역할(`ROLE`)을 가지지만 각 관리자별로 별도의 `AUTHORITY`(권한)을 할당해 줄 수 있다.  


### 로그인 유지 기능 추가  

로그인 유지를 위해 서버상 `session`에 데이터를 저장해놓고 처리하는 방법이 있고
클라이언트에 유저정보를 담은 쿠키를 생성하고 해당 쿠키를 전달받아 처리하는 방법이 있다.  

우선 로그인 폼에 remember-me 파라미터를 추가 

```html
<form method="post">
  <p>
    <label for="username">Username</label> <input type="text" id="username" name="username" value="user88" />
  </p>
  <p>
    <label for="password">Password</label> <input type="password" id="password" name="password" value="pw88" />
  </p>
  <p>
    <label for="remember-me">Remember-Me</label> <input type="checkbox" id="remember-me" name="remember-me" />
  </p>
  <button type="submit" class="btn">Log in</button>
  </form>
```

또한 필터설정에서도 remember-me에 대한 정보를 클라이언트에게 반환해야 하기때문에 아래 설정 추가  

```java
@Override
protected void configure(HttpSecurity http) throws Exception {
  log.info("SecurityConfig configure...");
  ...
  ...
  http.rememberMe().key("securitykey")
          .userDetailsService(customSecurityUsersService)
          .tokenValiditySeconds(60 * 60 * 24); //24시간
}
```

`securitykey`라는 문자열을 키로 `customSecurityUsersService`객체로 사용자 데이터를 가져와 암호화(Hash) 하여 사용자에게 반환한다.  
해당 쿠키는 기본 2주 유지되지만 `tokenValiditySeconds` 메서드를 통해 변경 가능하다.  

> `Base64Encode(username:expiryTime:Md5(username:expiryTime:password:key)` 한 값을 반환한다고 한다.  

이렇게 토큰을 쿠키값으로 유지하도록 설정하면 브라우저를 종료하더라도 쿠기는 남아있기 때문에 바로 로그인 처리된 세션생성이 가능하다.  

문제는 비밀번호 변경시 쿠키값도 변경되어야 한다는 거다.  
이를 해결하기 위해 DB에 유저에 대한 토큰값을 저장하고 지속적으로 업데이트가 일어날 수 있도록 설정하자.  

먼저 `Spring security`가 토큰 정보를 관리하는 테이블 생성  

```java
@Getter
@Setter
@Table(name = "persistent_logins")
@Entity
public class PersistentLogin {
    private String username;
    @Id
    private String series;
    private String token;
    private LocalDateTime lastUsed;
}
```

```java
@Override
protected void configure(HttpSecurity http) throws Exception {
  log.info("SecurityConfig configure...");
  ...
  ...
  http.rememberMe().key("securitykey")
    .userDetailsService(customSecurityUsersService)
    .tokenRepository(getJDBCRepository())
    .tokenValiditySeconds(60 * 60 * 24); //24시간
}

private PersistentTokenRepository getJDBCRepository() {
  JdbcTokenRepositoryImpl jdbcTokenRepository = new JdbcTokenRepositoryImpl();
  jdbcTokenRepository.setDataSource(datasource);
  return jdbcTokenRepository;
}
```

### Controller Method 접근 제한  

위의 `WebSecurityConfigurerAdapter`의 필터설정 `http.authorizeRequests().antMatchers("...").hasAnyRole("...", "...", ...)` 을 통해서도 접근제한이 가능하지만  
메서드에 어노테이션을 지정하는 것으로도 접근제한이 가능하다.  

먼저 `WebSecurityConfigurerAdapter` 하위 클래스에 `@EnableGlobalMethodSecurity` 어노테이션 설정, 다른 클래스에서도 시큐리티 어노테이션을 사용할 수 있도록 설정한다.  

```java
@Log
@EnableWebSecurity
@EnableGlobalMethodSecurity(securedEnabled = true)
public class SecurityConfig extends WebSecurityConfigurerAdapter {
  ...
  ...
}
```

간단한 `ROLE_MANAGER` 만 접근 가능한 컨트롤러 메서드 정의  

```java
@Controller
@RequestMapping("/manager")
@Log
public class ManagerController{

    @Secured({"ROLE_MANAGER"})
    @RequestMapping("/page")
    public void getPage() {
        log.info("getPage revoke ");
    }
}
```

### 패스워드 암호화  

`PasswordEncoder` 구현객체를 통해 암호화 가능  
이중 `BCryptPasswordEncoder` 클래스를 통해 단방향 해시 암호화 구현   

```java
@Log
@EnableWebSecurity
@EnableGlobalMethodSecurity(securedEnabled = true)
public class SecurityConfig extends WebSecurityConfigurerAdapter {
  ...
  ...
  @Bean
  public PasswordEncoder passwordEncoder() {
      return new BCryptPasswordEncoder();
  }
  @Autowired
  public void configureGlobal(AuthenticationManagerBuilder auth) throws Exception {
      log.info("SecurityConfig configureGlobal...");
      auth.userDetailsService(customSecurityUsersService).passwordEncoder(passwordEncoder());
  }
}
```

또한 `AuthenticationManagerBuilder` 를 통해 `customSecurityUsersService` 에 `PasswordEncoder` 를 등록한다.   
`PasswordEncoder`는 다른 서비스에서도 쓰일 수 있음으로 빈객체로 등록.  

#### 회원가입

```html
<form method="post">
  <p>
    <label for="uid">UID</label> <input type="text" id="uid" name="uid" value="newbie"/>
  </p>
  <p>
    <label for="upw">UPW</label> <input type="password" id="upw" name="upw" value="newbie"/>
  </p>
  <p>
    <label for="uname">UNAME</label> <input type="text" id="uname" name="uname" value="newbie"/>
  </p>
  <p>
    <input type="checkbox" class="roles[0].roleName" value="BASIC" checked>BASIC
    <input type="checkbox" class="roles[1].roleName" value="MANAGER">MANAGER
    <input type="checkbox" class="roles[2].roleName" value="ADMIN">ADMIN
  </p>
  <button type="submit" class="btn">Log in</button>
</form>
```

```java
@PostMapping("/join")
public String joinPost(@ModelAttribute("member") Member member) {
  log.info("joinPost invoke, member:" + member);
  String encryptPassword = passwordEncoder.encode(member.getUpw());
  member.setUpw(encryptPassword);
  memberRepository.save(member);

  member.getRoles().forEach(
    memberRole -> memberRole.setMember(member)
  );
  memberRoleRepository.saveAll(member.getRoles());

  return "/member/joinResult";
}
```

### 로그인 후 페이지 이동  

`url` 이동시에 로그인 필터에 걸려 로그인 페이지 이동시에는 상관없다.  
로그인 완료 후 원래 이동하려는 페이지로 이동한다.  

하지만 직접 `/login` url로 `GET Request` 요청후 로그인시에는 루트 디렉토리로 이동된다.  

이동할 페이지 지정을 위해 이동할 Url을 파라미터(`/login?dest=...`)로 같이 넘겨 세션에 저장하고 로그인 성공시 이동하도록 설정해보자.  

먼저 login 필터에 `successHandler` 메서드를 사용해 로그인 성공후 실행할 코드들이 정의되어 있는 클래스 `LoginSuccessHandler`를 설정한다.  
```java

@Log
@EnableWebSecurity
@EnableGlobalMethodSecurity(securedEnabled = true)
public class SecurityConfig extends WebSecurityConfigurerAdapter {

  @Override
  protected void configure(HttpSecurity http) throws Exception {
      log.info("SecurityConfig configure...");
      http.authorizeRequests()
        .antMatchers("/boards/list").permitAll()
        .antMatchers("/boards/register").hasAnyRole("BASIC", "MANAGER", "ADMIN");

      http.csrf().disable()
        .formLogin()
        .loginPage("/login")
        .successHandler(new LoginSuccessHandler());
      ...
      ...
  }
  ...
}
```
`LoginSuccessHandler` 는 `SavedRequestAwareAuthenticationSuccessHandler`의 여러가지 메서드중 로그인 성공시 이동경로를 설정할 `determineTargetUrl` 메서드를 오버라이딩 한다.  

```java
@Log
public class LoginSuccessHandler extends SavedRequestAwareAuthenticationSuccessHandler {

  @Override
  protected String determineTargetUrl(HttpServletRequest request, HttpServletResponse response) {
    log.info("determineTargetUrl....");
    Object dest = request.getSession().getAttribute("dest");
    String nextUrl = null;
    if (dest != null) {
      request.getSession().removeAttribute("dest");
      nextUrl = (String) dest;
    } else {
      nextUrl = super.determineTargetUrl(request, response);
    }
    log.info("determineTargetUrl nextUrl: " + nextUrl);
    return nextUrl;
  }
}
```

코드를 보면 `session` 객체 안의 `dest` 객체를 꺼내 `url` 로 설정한다.  

위의 로직이 가능하려면 먼저 `session` 에 `dest` 값을 삽입하는 과정이 있어야 하는데 인터셉터로 처리한다.  

```java
@Log
public class LoginCheckInterceptor extends HandlerInterceptorAdapter {

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        log.info("preHandle...");
        String dest = request.getParameter("dest");
        if (dest != null)
            request.getSession().setAttribute("dest", dest);
        return super.preHandle(request, response, handler);
    }
}
```
특정 요청이 들어오기 전에 호출되는 `preHandle` 정의  

해당 인터셉터를 `WebMvcConfigurer` 를 통해 요청 `url` 에 매핑  
```java
@Log
@Configuration
public class InterceptorConfig implements WebMvcConfigurer {
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new LoginCheckInterceptor()).addPathPatterns("/login");
        WebMvcConfigurer.super.addInterceptors(registry);
    }
}
```
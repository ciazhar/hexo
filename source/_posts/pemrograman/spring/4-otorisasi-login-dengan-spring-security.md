---
title: Otorisasi Login dengan spring security
date: 2017-04-23 14:17:37
categories:
  - Pemrograman
  - Spring
---
![](/images/springboot.png)
# Bikin Otorisasi Login #
- Tambahkan dependency (pom.xml)
  ```
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-security</artifactId>
    </dependency>
  ```
- Bikin KonfigurasiSecurity (main/java/domain/config/KonfigurasiSecurity.java)
  ```
    @Configuration
    @EnableWebSecurity
    @EnableGlobalMethodSecurity(prePostEnabled = true)
    public class KonfigurasiSecurity extends WebSecurityConfigurerAdapter{
        private static final String SQL_LOGIN
                = "SELECT username,password, enable " +
                "FROM s_users WHERE username = ?";

        private static final String SQL_PERMISSION
                = "SELECT u.username, r.nama as authority " +
                "FROM s_users u " +
                "JOIN s_user_role ur on u.id = ur.id_user " +
                "JOIN s_roles r on ur.id_role = r.id " +
                "WHERE u.username = ?";

        @Autowired
        private DataSource dataSource;

        @Autowired
        public void configureGlobal(AuthenticationManagerBuilder auth) throws Exception{

            //setting security non database
            auth
                    .inMemoryAuthentication()
                    .withUser("ciazhar")
                    .password("123")
                    .roles("apa");

            ///Setting security database
            /*auth
                    .jdbcAuthentication()
                    .dataSource(dataSource)
                    .usersByUsernameQuery(SQL_LOGIN)
                    .authoritiesByUsernameQuery(SQL_PERMISSION);*/
        }

        ///konfigurasi web mana yg boleh diakses admin staf user dll
        protected void configure(HttpSecurity http) throws Exception{
            http
                    .authorizeRequests()
                    .antMatchers("/css/**","/js/**").permitAll()
                    .anyRequest().authenticated()
                    .and()
                    .formLogin()
                    .loginPage("/login")
                    .defaultSuccessUrl("/")
                    .permitAll()
                    .and()
                    .logout();
        }
    }
  ```
- Register UI(src/main/java/domain/config/KonfigurasiWeb.java)
  Karena form login kita tidak menggunakan controller, maka harus didaftarkan terlebih dahulu.
  ```
    @Configuration
    public class KonfigurasiWeb extends WebMvcConfigurerAdapter{

        @Override
        public void addViewControllers(ViewControllerRegistry registry){
            registry.addViewController("/login").setViewName("login");
            registry.addViewController("/materi/list").setViewName("materi/listMateri");
        }
    }
  ```

- Bikin UI Login(main/resources/login.html)
  ```
    <html xmlns:th="http://www.thymeleaf.org">
      <head>

        <title>Log In</title>

        <!-- Bootstrap core CSS -->

        <link th:href="@{/css/bootstrap.min.css}" rel="stylesheet" />
        <link th:href="@{/css/bootstrap-theme.min.css}" rel="stylesheet" />
        <link th:href="@{/css/signin.css}" rel="stylesheet" />
      </head>

      <body>

        <div class="container">

          <form name="f" class="form-signin"  th:action="@{/login}" method="post">
            <div th:if="${param.error}" class="alert alert-error">
              Invalid username and password.
            </div>

            <div th:if="${param.logout}" class="alert alert-success">
              You have been logged out.
            </div>

            <h2 class="form-signin-heading">Please sign in</h2>
            <label for="username" class="sr-only">Username</label>
            <input type="text" name="username" id="username" class="form-control" placeholder="Username" required="true" autofocus="true" />
            <label for="password" class="sr-only">Password</label>
            <input type="password" name="password" id="password" class="form-control" placeholder="Password" required="true" />
            <button class="btn btn-lg btn-primary btn-block" type="submit">Sign in</button>
          </form>

        </div>

        <script th:src="@{/js/jquery.min.js}"></script>
        <script th:src="@{/js/bootstrap.min.js}"></script>
      </body>
    </html>
  ```

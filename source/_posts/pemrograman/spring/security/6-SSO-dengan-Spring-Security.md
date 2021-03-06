---
title: Single Signed On dengan Spring Security
date: 2017-05-27 17:22:29
categories:
  - Pemrograman
  - Spring
---
![](/images/springboot.png)

# Aplikasi Authorization Sever dengan Spring Security

#Setup Development Environment
- JDK 1.8
- Tomcat Sever
- MySQL

#Teknologi yang digunakan
- Spring Boot
- Spring Security
- AngularJS

#Setup Project
- Buka browser masukkan url
  ```
    http://start.spring.io/
  ```
- Masukkan data, sesuaikan dengan dependency yang dibutuhkan(web, security,oauth2) lalu download
- Add project ke text editor

Note :
Pada Project tersebut terdapat 3 buah file utama yaitu :
- pom.xml (konfigurasi maven)
- application.properties (konfigurasi database)
- Application.java (main class)

#Simple Oauth Facebook
- Konfigurasi Oauth (application.yml)
  Note : application.properties diganti application.yml
  ```
    security:
      oauth2:
        client:
          clientId: 233668646673605
          clientSecret: 33b17e044ee6a4fa383f46ec6e28ea1d
          accessTokenUri: https://graph.facebook.com/oauth/access_token
          userAuthorizationUri: https://www.facebook.com/dialog/oauth
          tokenName: oauth_token
          authenticationScheme: query
          clientAuthenticationScheme: form
        resource:
          userInfoUri: https://graph.facebook.com/me
  ```
- Kasih anotasi @EnableOAuth2Sso (Aplikasi.java)
  ```
    @EnableOAuth2Sso
    @SpringBootApplication
    public class Aplikasi {
    	public static void main(String[] args) {
    		SpringApplication.run(Aplikasi.class, args);
    	}
    }
  ```
- Bikin UI jika otorisari succes(resource/static/index.html)
  ```
    <!doctype html>
    <html lang="en">
    <head>
        <meta charset="utf-8" />
        <meta http-equiv="X-UA-Compatible" content="IE=edge" />
        <title>Demo</title>
        <meta name="description" content="" />
        <meta name="viewport" content="width=device-width" />
        <base href="/" />
        <link rel="stylesheet" type="text/css"
              href="/webjars/bootstrap/css/bootstrap.min.css" />
        <script type="text/javascript" src="/webjars/jquery/jquery.min.js"></script>
        <script type="text/javascript"
                src="/webjars/bootstrap/js/bootstrap.min.js"></script>
    </head>
    <body>
      <h1>SSO sukses</h1>
    </body>
    </html>
  ```
# Bikin UI untuk login
- Bikin UI (resource/static/index.html)
  ```
    ...
    <body ng-app="app" ng-controller="home as home">
      <h1>Login</h1>
      <div class="container" ng-show="!home.authenticated">
          With Facebook: <a href="/login">click here</a>
      </div>
      <div class="container" ng-show="home.authenticated">
          <h1>SSO berhasil</h1>
          Logged in as: <span ng-bind="home.user"></span>
      </div>
      <script type="text/javascript" src="/webjars/angularjs/angular.min.js"></script>
      <script type="text/javascript">
          angular.module("app", []).controller("home", function($http) {
              var self = this;
              $http.get("/user").success(function(data) {
                  self.user = data.userAuthentication.details.name;
                  self.authenticated = true;
              }).error(function() {
                  self.user = "N/A";
                  self.authenticated = false;
              });
          });
      </script>
    </body>
    ...
  ```
- Bikin RestController(java/domain/controllers/IndexController.java)
  ```
  @RestController
  public class IndexController {
      @RequestMapping("/user")
      public Principal user(Principal principal) {
          return principal;
      }
  }
  ```
- Bikin Konfigurasi Security(java/domain/config/KonfigurasiSecurity.java)
  ```
    @Configurable
    @EnableOAuth2Sso
    @EnableWebSecurity
    public class KonfigurasiSecurity extends WebSecurityConfigurerAdapter {
        @Override
        protected void configure(HttpSecurity http) throws Exception {
            http.antMatcher("/**").authorizeRequests().antMatchers("/", "/login**", "/webjars/**").permitAll().anyRequest()
                    .authenticated();
        }
    }
  ```
#Pretty print JSON
- Konfigurasi YAML (resources/application.yml)
  ```  
    spring:
      jackson:
        serialization:
          INDENT_OUTPUT: true
  ```

#Custom Logout
- Konfigurasi client Side (resources/static/index.html)
  ```
    angular
    .module("app", [])
    ...
    .controller("home", function($http, $location) {
      var self = this;
      ...
      self.logout = function() {
        $http.post('/logout', {}).success(function() {
          self.authenticated = false;
          $location.path("/");
        }).error(function(data) {
          console.log("Logout failed")
          self.authenticated = false;
        });
      };
    });
  ```
- Konfigurasi Server Side (java/domain/config/KonfigurasiSecurity.java)
  ```
    .and()
    .logout().logoutSuccessUrl("/").permitAll()
    .and()
    .csrf().csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse());
  ```

#Memisahkan Otorisasi Facebook dari Spring Security
- Mengubah anotasi @EnableOAuth2Sso dengan @EnableOAuth2Client (java/domain/config/KonfigurasiSecurity.java)

     Sejatinya adalah aplikasi yang telah kita buat tadi berdiri di persis di atas Spring Security. Hal ini dikarenakan kita menggunakan anotasi @EnableOAuth2Sso. Anotasi tersebut sendiri terdiri dari 2 fitur yaitu OAuth2 client dan Oauth2 authentification.
     Untuk OAuth2 client dia dapat berinteraksi dengan resource OAuth2 yang disediakan oleh Authorization Server (dalam konteks ini facebook Authorization Server).
     Sedangkan OAuth2 authentification dia berfungsi untuk menyelaraskan aplikasi kita dengan REST milik Spring Security
     Jadi ketika 2 fitur itu digunakan untuk SSO ke facebook saja, maka kita hanya dapat SSO ke facebook saja
     Oleh karena itu kita akan mengganti anotasi @EnableOAuth2Sso dengan @EnableOAuth2Client
  ```
    @Configurable
    @EnableOAuth2Client
    @EnableWebSecurity
    public class KonfigurasiSecurity extends WebSecurityConfigurerAdapter {

    }
  ```
- Membuat Filter authentification(java/domain/config/KonfigurasiSecurity.java)
  ```
    @Autowired
    OAuth2ClientContext oauth2ClientContext;

    ///Bean untuk memberitahu filter tentang registrasi client dengan facebook
    @Bean
    @ConfigurationProperties("facebook.client")
    public AuthorizationCodeResourceDetails facebook() {
      return new AuthorizationCodeResourceDetails();
    }

    ///Bean untuk memberitahu filter tentang dimana user end point di facebook
    @Bean
    @ConfigurationProperties("facebook.resource")
    public ResourceServerProperties facebookResource() {
      return new ResourceServerProperties();
    }

    private Filter ssoFilter() {
      OAuth2ClientAuthenticationProcessingFilter facebookFilter = new OAuth2ClientAuthenticationProcessingFilter("/login/facebook");
      OAuth2RestTemplate facebookTemplate = new OAuth2RestTemplate(facebook(), oauth2ClientContext);
      facebookFilter.setRestTemplate(facebookTemplate);
      UserInfoTokenServices tokenServices = new UserInfoTokenServices(facebookResource().getUserInfoUri(), facebook().getClientId());
      tokenServices.setRestTemplate(facebookTemplate);
      facebookFilter.setTokenServices(tokenServices);
      return facebookFilter;
    }


    @Override
    protected void configure(HttpSecurity http) throws Exception {
      http.antMatcher("/**")
      ...
      .addFilterBefore(ssoFilter(), BasicAuthenticationFilter.class);
    }
  ```
- Mengubah konfigurasi OAuth2(resources/application.yml)
  ```
    facebook:
      client:
        clientId: 233668646673605
        clientSecret: 33b17e044ee6a4fa383f46ec6e28ea1d
        accessTokenUri: https://graph.facebook.com/oauth/access_token
        userAuthorizationUri: https://www.facebook.com/dialog/oauth
        tokenName: oauth_token
        authenticationScheme: query
        clientAuthenticationScheme: form
      resource:
        userInfoUri: https://graph.facebook.com/me      
    logging:
      level:
        org.springframework.security: DEBUG
  ```
- Ganti URL pada UI(resources/static/index.html)
  ```
    <div class="container" ng-show="!home.authenticated">
      <div>
      With Facebook: <a href="/login/facebook">click here</a>
      </div>
    </div>
  ```
- Buat Konfigurasi untuk Redirect(java/domain/config/KonfigurasiSecurity.java)
  ```
    @Bean
    public FilterRegistrationBean oauth2ClientFilterRegistration(OAuth2ClientContextFilter filter) {
      FilterRegistrationBean registration = new FilterRegistrationBean();
      registration.setFilter(filter);
      registration.setOrder(-100);
      return registration;
    }

  ```
#Tambah Authentification untuk github
- Edit UI (resources/static/index.html)
  ```
    With Github: <a href="/login/github">click here</a>
  ```
- Tambah Konfigurasi untuk Github(java/domain/config/KonfigurasiSecurity.java)
  ```
    @Bean
    @ConfigurationProperties("github.client")
    public AuthorizationCodeResourceDetails github() {
        return new AuthorizationCodeResourceDetails();
    }

    @Bean
    @ConfigurationProperties("github.resource")
    public ResourceServerProperties githubResource() {
        return new ResourceServerProperties();
    }
  ```
- Edit Filter(java/domain/config/KonfigurasiSecurity.java)
  ```
    private Filter ssoFilter() {
        CompositeFilter filter = new CompositeFilter();
        List<Filter> filters = new ArrayList<>();
        OAuth2ClientAuthenticationProcessingFilter facebookFilter = new OAuth2ClientAuthenticationProcessingFilter("/login/facebook");
        OAuth2RestTemplate facebookTemplate = new OAuth2RestTemplate(facebook(), oauth2ClientContext);
        facebookFilter.setRestTemplate(facebookTemplate);
        UserInfoTokenServices tokenServices = new UserInfoTokenServices(facebookResource().getUserInfoUri(), facebook().getClientId());
        tokenServices.setRestTemplate(facebookTemplate);
        facebookFilter.setTokenServices(tokenServices);
        filters.add(facebookFilter);
        OAuth2ClientAuthenticationProcessingFilter githubFilter = new OAuth2ClientAuthenticationProcessingFilter("/login/github");
        OAuth2RestTemplate githubTemplate = new OAuth2RestTemplate(github(), oauth2ClientContext);
        githubFilter.setRestTemplate(githubTemplate);
        tokenServices = new UserInfoTokenServices(githubResource().getUserInfoUri(), github().getClientId());
        tokenServices.setRestTemplate(githubTemplate);
        githubFilter.setTokenServices(tokenServices);
        filters.add(githubFilter);
        filter.setFilters(filters);
        return filter;
    }
  ```
- Tambah Konfigurasi OAuth2 Github(resources/application.yml)
  ```
    github:
      client:
        clientId: bd1c0a783ccdd1c9b9e4
        clientSecret: 1a9030fbca47a5b2c28e92f19050bb77824b5ad1
        accessTokenUri: https://github.com/login/oauth/access_token
        userAuthorizationUri: https://github.com/login/oauth/authorize
        clientAuthenticationScheme: form
      resource:
        userInfoUri: https://api.github.com/user
  ```

#Membuat authorization server
- Merapikan ssoFilter

  Note : kita akan membagi menjadi 2 agar terlihat rapi   
  ```
    private Filter ssoFilter() {
    		CompositeFilter filter = new CompositeFilter();
    		List<Filter> filters = new ArrayList<>();
    		filters.add(ssoFilter(facebook(), "/login/facebook"));
    		filters.add(ssoFilter(github(), "/login/github"));
    		filter.setFilters(filters);
    		return filter;
    	}

    private Filter ssoFilter(ClientResources client, String path) {
        OAuth2ClientAuthenticationProcessingFilter filter = new OAuth2ClientAuthenticationProcessingFilter(
                path);
        OAuth2RestTemplate template = new OAuth2RestTemplate(client.getClient(), oauth2ClientContext);
        filter.setRestTemplate(template);
        UserInfoTokenServices tokenServices = new UserInfoTokenServices(
                client.getResource().getUserInfoUri(), client.getClient().getClientId());
        tokenServices.setRestTemplate(template);
        filter.setTokenServices(tokenServices);
        return filter;
    }

  ```
- membuat class ClientResources untuk mengatur otorisasi client dan akses resource yang pada tadinya terpisah
  ```
      class ClientResources {

        @NestedConfigurationProperty
        private AuthorizationCodeResourceDetails client = new AuthorizationCodeResourceDetails();

        @NestedConfigurationProperty
        private ResourceServerProperties resource = new ResourceServerProperties();

        public AuthorizationCodeResourceDetails getClient() {
          return client;
        }

        public ResourceServerProperties getResource() {
          return resource;
        }
      }
   ```
-

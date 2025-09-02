# CSRF och CORS Guide med Spring Boot

## Innehållsförteckning
1. [CSRF (Cross-Site Request Forgery)](#csrf-cross-site-request-forgery)
2. [CORS (Cross-Origin Resource Sharing)](#cors-cross-origin-resource-sharing)
3. [Spring Boot Skydd](#spring-boot-skydd)
4. [Praktiska Exempel](#praktiska-exempel)

---

## CSRF (Cross-Site Request Forgery)

### Vad är CSRF?
CSRF är en attack där en illvillig webbsida lurar en användares webbläsare att utföra oönskade åtgärder på en annan webbsida där användaren är inloggad.

### Så fungerar CSRF-attack:

```
1. Användaren loggar in på bank.se
2. Webbläsaren sparar sessionscookie för bank.se
3. Användaren besöker evil.com (utan att logga ut från banken)
4. evil.com innehåller dolt formulär som skickar pengar från användarens konto
5. Webbläsaren skickar automatiskt med sessionscookie till bank.se
6. Banken tror att det är användaren som gör överföringen
```

### Visuell representation:
```
Användare                Evil Site               Bank Site
    |                        |                      |
    |---(1) Besöker--------->|                      |
    |                        |                      |
    |<--(2) Sida med---------| <form action=        |
    |       dolt formulär    |  "bank.se/transfer"  |
    |                        |  method="post">      |
    |                        |  <input name="to"    |
    |                        |   value="hacker">    |
    |                        |  <input name="amount"|
    |                        |   value="1000">      |
    |                        |                      |
    |-----------(3) POST med sessionscookie-------->|
    |                                               |
    |<-----------(4) Pengar överförda---------------|
```

### Spring Boot CSRF-skydd

#### 1. Aktivera CSRF-skydd (aktiverat som standard)
```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf
                .csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse())
                .ignoringRequestMatchers("/api/public/**") // Undantag för vissa endpoints
            )
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/login", "/register").permitAll()
                .anyRequest().authenticated()
            );
        return http.build();
    }
}
```



#### 2. CSRF Token i JavaScript/AJAX
```javascript
// Hämta CSRF token från meta-tag eller cookie
const token = document.querySelector('meta[name="_csrf"]').getAttribute('content');
const header = document.querySelector('meta[name="_csrf_header"]').getAttribute('content');

// Använd i AJAX-anrop
fetch('/api/transfer', {
    method: 'POST',
    headers: {
        'Content-Type': 'application/json',
        [header]: token
    },
    body: JSON.stringify({
        amount: 1000,
        to: 'account123'
    })
});
```

#### 4. Controller-exempel
```java
@RestController
public class BankController {
    
    @PostMapping("/transfer")
    public ResponseEntity<String> transfer(
            @RequestParam String to,
            @RequestParam BigDecimal amount,
            HttpServletRequest request) {
        
        // Spring Security validerar automatiskt CSRF-token
        // Om token saknas eller är felaktig kastas InvalidCsrfTokenException
        
        transferService.transfer(to, amount);
        return ResponseEntity.ok("Transfer completed");
    }
}
```

---

## CORS (Cross-Origin Resource Sharing)

### Vad är CORS?
CORS är en säkerhetsmekanism som kontrollerar vilka domäner som får göra HTTP-anrop till din server från en webbläsare.

### Same-Origin Policy
Webbläsare blockerar som standard requests mellan olika origins:
- **Origin = protokoll + domän + port**
- `https://app.example.com:3000` och `https://api.example.com:8080` = olika origins

### CORS-flöde:

#### Simple Request:
```
Frontend (app.com)          Backend (api.com)
       |                          |
       |---(1) GET /data--------->|
       |    Origin: app.com       |
       |                          |
       |<--(2) Response-----------|
       |    Access-Control-       |
       |    Allow-Origin: app.com |
```

#### Preflight Request (för komplexa requests):
```
Frontend (app.com)          Backend (api.com)
       |                          |
       |---(1) OPTIONS /data----->|
       |    Origin: app.com       |
       |    Access-Control-       |
       |    Request-Method: POST  |
       |                          |
       |<--(2) Preflight OK-------|
       |    Access-Control-       |
       |    Allow-Origin: app.com |
       |    Allow-Methods: POST   |
       |                          |
       |---(3) POST /data-------->|
       |    Origin: app.com       |
       |                          |
       |<--(4) Actual Response----|
```

### Spring Boot CORS-konfiguration

#### 1. Global CORS-konfiguration (inte nödvändigt, bara exempel)
```java
@Configuration
public class CorsConfig implements WebMvcConfigurer {

    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**")
                .allowedOrigins("https://myapp.com", "http://localhost:3000")
                .allowedMethods("GET", "POST", "PUT", "DELETE", "OPTIONS")
                .allowedHeaders("*")
                .allowCredentials(true)
                .maxAge(3600); // Cache preflight för 1 timme
    }
}
```

#### 2. Controller-nivå CORS
```java
@RestController
@CrossOrigin(origins = {"https://myapp.com", "http://localhost:3000"})
public class ApiController {
    
    @GetMapping("/data")
    @CrossOrigin(origins = "https://trusted-app.com") // Specifik för denna endpoint
    public ResponseEntity<List<Data>> getData() {
        return ResponseEntity.ok(dataService.getAllData());
    }
}
```

#### 3. Security-baserad CORS (när Spring Security används)
```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .cors(cors -> cors.configurationSource(corsConfigurationSource()))
            .csrf(csrf -> csrf.disable()) // Ofta disabled för API:er
            .authorizeHttpRequests(auth -> auth
                .anyRequest().authenticated()
            );
        return http.build();
    }

    @Bean
    public CorsConfigurationSource corsConfigurationSource() {
        CorsConfiguration configuration = new CorsConfiguration();
        configuration.setAllowedOriginPatterns(Arrays.asList("https://*.mycompany.com"));
        configuration.setAllowedMethods(Arrays.asList("GET", "POST", "PUT", "DELETE"));
        configuration.setAllowedHeaders(Arrays.asList("*"));
        configuration.setAllowCredentials(true);
        
        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/api/**", configuration);
        return source;
    }
}
```

---

## Praktiska Exempel

### Komplett säkerhetskonfiguration
```java
@Configuration
@EnableWebSecurity
public class WebSecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            // CORS-konfiguration
            .cors(cors -> cors.configurationSource(corsConfigurationSource()))
            
            // CSRF-skydd - VIKTIGT: Disabled för JWT API:er!
            .csrf(csrf -> csrf
                .disable() // JWT är stateless och inte sårbart för CSRF
            )
            
            // Auktorisering
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/public/**").permitAll()
                .requestMatchers("/api/**").authenticated()
                .anyRequest().authenticated()
            )
            
            // JWT för API - stateless sessions
            .sessionManagement(session -> session
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
            )
            .oauth2ResourceServer(oauth2 -> oauth2.jwt(Customizer.withDefaults()));
            
        return http.build();
    }

    @Bean
    public CorsConfigurationSource corsConfigurationSource() {
        CorsConfiguration configuration = new CorsConfiguration();
        
        // Tillåt specifika domäner i produktion
        if (isProduction()) {
            configuration.setAllowedOrigins(Arrays.asList(
                "https://myapp.com",
                "https://admin.myapp.com"
            ));
        } else {
            // Utvecklingsläge - tillåt localhost
            configuration.setAllowedOriginPatterns(Arrays.asList("http://localhost:*"));
        }
        
        configuration.setAllowedMethods(Arrays.asList("GET", "POST", "PUT", "DELETE", "OPTIONS"));
        configuration.setAllowedHeaders(Arrays.asList("*"));
        configuration.setAllowCredentials(true);
        configuration.setMaxAge(3600L);
        
        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/api/**", configuration);
        return source;
    }
    
---

## JWT vs CSRF - Varför behöver JWT inte CSRF-skydd?

### Varför är JWT immunt mot CSRF?

**CSRF-attacken fungerar så här:**
1. Användaren är inloggad och har en sessionscookie
2. Webbläsaren skickar **automatiskt** cookies med varje request
3. En illvillig sida kan lura webbläsaren att skicka requests med cookien

**JWT fungerar annorlunda:**
1. JWT lagras vanligtvis i localStorage eller som variabel i JavaScript
2. JWT skickas **manuellt** i Authorization-headern
3. Webbläsaren skickar ALDRIG automatiskt Authorization-headers

### Visuell jämförelse:

#### Cookie-baserad autentisering (sårbar för CSRF):
```
Evil Site                    Browser                     Server
    |                          |                           |
    |---(1) <form>------------>|                           |
    |                          |---(2) POST med cookie---->|
    |                          |     Cookie: session=123   |
    |                          |                           |
    |                          |<--(3) Autentiserad--------|
```

#### JWT-baserad autentisering (INTE sårbar för CSRF):
```
Evil Site                    Browser                     Server
    |                          |                           |
    |---(1) <form>------------>|                           |
    |                          |---(2) POST utan header--->|
    |                          |     (Ingen Authorization) |
    |                          |                           |
    |                          |<--(3) 401 Unauthorized----|
```

### Säker JWT-implementation utan CSRF:
```java
@Configuration
@EnableWebSecurity
public class JWTSecurityConfig {

    @Bean
    public SecurityFilterChain apiFilterChain(HttpSecurity http) throws Exception {
        http
            // Endast CORS behövs för JWT API:er
            .cors(cors -> cors.configurationSource(corsConfigurationSource()))
            
            // CSRF disabled - JWT behöver det inte
            .csrf(csrf -> csrf.disable())
            
            // Stateless sessions
            .sessionManagement(session -> session
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
            )
            
            // JWT-validering
            .oauth2ResourceServer(oauth2 -> oauth2
                .jwt(jwt -> jwt
                    .jwtAuthenticationConverter(jwtAuthenticationConverter())
                )
            )
            
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/auth/**").permitAll()
                .requestMatchers("/api/**").authenticated()
                .anyRequest().denyAll()
            );
            
        return http.build();
    }
}
```

### Hybrid-lösning för både webb och API:
```java
@Configuration
@EnableWebSecurity
public class HybridSecurityConfig {

    // Separata filter chains för olika typer av endpoints
    
    @Bean
    @Order(1)
    public SecurityFilterChain apiFilterChain(HttpSecurity http) throws Exception {
        http
            .securityMatcher("/api/**")
            .cors(cors -> cors.configurationSource(corsConfigurationSource()))
            .csrf(csrf -> csrf.disable()) // JWT API - ingen CSRF
            .sessionManagement(session -> session
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
            )
            .oauth2ResourceServer(oauth2 -> oauth2.jwt(Customizer.withDefaults()))
            .authorizeHttpRequests(auth -> auth.anyRequest().authenticated());
            
        return http.build();
    }
    
    @Bean
    @Order(2)
    public SecurityFilterChain webFilterChain(HttpSecurity http) throws Exception {
        http
            .securityMatcher("/**")
            // Webb med sessions - CSRF behövs
            .csrf(csrf -> csrf
                .csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse())
            )
            .sessionManagement(session -> session
                .sessionCreationPolicy(SessionCreationPolicy.IF_REQUIRED)
                .maximumSessions(1)
            )
            .formLogin(form -> form
                .loginPage("/login")
                .defaultSuccessUrl("/dashboard")
            )
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/login", "/register").permitAll()
                .anyRequest().authenticated()
            );
            
        return http.build();
    }
}
```

### Frontend-exempel (React) - Endast CORS-hantering för JWT
```javascript
// API-klient för JWT - INGEN CSRF-token behövs!
class JWTApiClient {
    constructor(baseURL) {
        this.baseURL = baseURL;
        this.token = localStorage.getItem('jwt_token');
    }

    setToken(token) {
        this.token = token;
        localStorage.setItem('jwt_token', token);
    }

    clearToken() {
        this.token = null;
        localStorage.removeItem('jwt_token');
    }

    async makeRequest(url, options = {}) {
        const config = {
            ...options,
            headers: {
                'Content-Type': 'application/json',
                ...options.headers
            }
        };

        // Lägg till JWT i Authorization header (INTE cookie!)
        if (this.token) {
            config.headers.Authorization = `Bearer ${this.token}`;
        }

        // VIKTIGT: Använd INTE credentials: 'include' för JWT
        // Det skulle skicka cookies som kan vara sårbara för CSRF
        
        try {
            const response = await fetch(`${this.baseURL}${url}`, config);
            
            if (response.status === 401) {
                // JWT expired eller ogiltigt
                this.clearToken();
                throw new Error('Authentication failed');
            }
            
            return response;
        } catch (error) {
            console.error('API request failed:', error);
            throw error;
        }
    }

    async post(url, data) {
        return this.makeRequest(url, {
            method: 'POST',
            body: JSON.stringify(data)
        });
    }

    async get(url) {
        return this.makeRequest(url, {
            method: 'GET'
        });
    }
}

// Användning - Inga CSRF-tokens behövs!
const api = new JWTApiClient('https://api.myapp.com');

async function login(username, password) {
    try {
        const response = await api.post('/auth/login', { username, password });
        const data = await response.json();
        
        if (response.ok) {
            api.setToken(data.token);
            console.log('Login successful');
        }
    } catch (error) {
        console.error('Login failed:', error);
    }
}

async function transferMoney(amount, to) {
    try {
        // JWT skickas automatiskt i Authorization header
        const response = await api.post('/transfer', { amount, to });
        if (response.ok) {
            console.log('Transfer successful');
        }
    } catch (error) {
        console.error('Transfer failed:', error);
    }
}
```

---

## Sammanfattning av Säkerhetsåtgärder

### CSRF-skydd (endast för session/cookie-baserad autentisering):
✅ **Använd CSRF-tokens** för alla state-changing operationer  
✅ **Validera Referer/Origin headers**  
✅ **Kräv custom headers** för AJAX-requests  
✅ **Använd SameSite cookies**  

### CORS-skydd (gäller både session och JWT):
✅ **Specificera exakta allowed origins** (undvik wildcards)  
✅ **Begränsa allowed methods** till det som behövs  
✅ **Sätt lämplig maxAge** för preflight cache  
✅ **Var försiktig med allowCredentials** (endast för cookies)  

### JWT-specifika säkerhetsåtgärder:
✅ **Lagra JWT säkert** (ej i cookies för att undvika CSRF)  
✅ **Korta expiration times** (15-60 minuter)  
✅ **Refresh token rotation**  
✅ **Token revocation via blacklist**  
✅ **HTTPS required** för production  

### Bästa praxis:
- **JWT API:er**: Använd endast CORS, disable CSRF
- **Session-baserad webb**: Använd både CSRF och CORS
- **Hybrid-lösning**: Separata filter chains för API och webb
- Validera all input på server-sidan
- Implementera rate limiting
- Logga och övervaka säkerhetshändelser

### När ska du använda vad?

**Använd CSRF + CORS:**
- Traditionella webbapplikationer med server-side rendering
- Session-baserad autentisering med cookies
- Formulär som skickas via browser

**Använd endast CORS:**
- JWT-baserade API:er
- SPA (Single Page Applications) med JWT
- Mobila applikationer
- Microservices kommunikation

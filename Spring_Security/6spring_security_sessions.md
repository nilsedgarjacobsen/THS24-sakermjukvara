# Spring Security - Session-konfiguration

## Introduktion

I denna guide kommer vi att konfigurera sessions i detalj. Du kommer att lära dig:
- Hur sessions fungerar i Spring Security
- Konfigurera session timeout och säkerhet
- Hantera samtidiga inloggningar
- Implementera "Remember Me"-funktionalitet
- Övervaka och debugga sessions

**Förutsättningar:** Ett Spring Boot-projekt med Spring Security, databas-integration och BCrypt password encoding redan implementerat.

## 1. Vad är sessions?

**Sessions** är Spring Securitys sätt att "komma ihåg" att en användare är inloggad mellan HTTP-requests.

### Så här fungerar det:
```
1. Användaren loggar in framgångsrikt
2. Spring Security skapar en session på servern
3. Ett session-ID skickas till webbläsaren som en cookie
4. Vid nästa request skickar webbläsaren session-ID:t
5. Spring Security hittar sessionen och "kommer ihåg" användaren
```

### Visualisering:
```
Webbläsare                    Server
     │                           │
     │ POST /login (anna/pass)    │
     │ ────────────────────────► │ ✓ Korrekt lösenord
     │                           │ Skapar session: ABC123
     │ ◄──────────────────────── │ Set-Cookie: JSESSIONID=ABC123
     │                           │
     │ GET /dashboard             │
     │ Cookie: JSESSIONID=ABC123  │
     │ ────────────────────────► │ Hittar session ABC123
     │                           │ → Användare är inloggad!
     │ ◄──────────────────────── │ Skickar dashboard-sida
```

## 2. Spring Securitys standard session-konfiguration

Som standard konfigurerar Spring Security sessions så här:

```java
// Detta händer automatiskt
http.sessionManagement(session -> session
    .sessionCreationPolicy(SessionCreationPolicy.IF_REQUIRED)  // Skapa session vid behov
    .maximumSessions(1)                                         // En session per användare
    .maxSessionsPreventsLogin(false)                           // Ny inloggning kastar ut gammal
    .sessionFixation().changeSessionId()                       // Skydd mot session fixation
);
```

## 3. Grundläggande session-konfiguration

### Uppdatera SecurityConfig
```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {
    
    @Autowired
    private CustomUserDetailsService userDetailsService;
    
    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(authz -> authz
                .requestMatchers("/public/**", "/css/**", "/js/**").permitAll()
                .requestMatchers("/signup", "/register").permitAll()
                .anyRequest().authenticated()
            )
            .formLogin(form -> form
                .loginPage("/login")
                .defaultSuccessUrl("/dashboard")
                .permitAll()
            )
            .logout(logout -> logout
                .logoutSuccessUrl("/login?logout")
                .permitAll()
            )
            .sessionManagement(session -> session
                .sessionCreationPolicy(SessionCreationPolicy.IF_REQUIRED)
                .invalidSessionUrl("/login?expired")
                .maximumSessions(1)
                .maxSessionsPreventsLogin(false)
            )
            .userDetailsService(userDetailsService);
        
        return http.build();
    }
}
```

## 4. Session timeout-konfiguration

### I application.properties:
```properties
# Session timeout (30 minuter)
server.servlet.session.timeout=30m

# Cookie-inställningar
server.servlet.session.cookie.name=MYSESSIONID
server.servlet.session.cookie.http-only=true
server.servlet.session.cookie.secure=false
server.servlet.session.cookie.same-site=lax
```

**Förklaring:**
- `timeout=30m` - Session upphör efter 30 minuters inaktivitet
- `http-only=true` - Förhindrar JavaScript-åtkomst till cookie (XSS-skydd)
- `secure=false` - För utveckling med HTTP (sätt till `true` för HTTPS i produktion)
- `same-site=lax` - CSRF-skydd

### Programmatisk timeout-konfiguration:
```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http
        // ... andra konfigurationer
        .sessionManagement(session -> session
            .sessionCreationPolicy(SessionCreationPolicy.IF_REQUIRED)
            .invalidSessionUrl("/login?expired")
            .maximumSessions(1)
            .maxSessionsPreventsLogin(false)
            .and()
            .sessionFixation().changeSessionId()
        );
    
    return http.build();
}

@Bean
public HttpSessionEventPublisher httpSessionEventPublisher() {
    return new HttpSessionEventPublisher();
}
```

## 5. Hantera samtidiga sessioner

### Tillåt endast en session per användare:
```java
.sessionManagement(session -> session
    .maximumSessions(1)
    .maxSessionsPreventsLogin(true)  // Förhindra ny inloggning
    .expiredUrl("/login?concurrent")
)
```

### Tillåt flera sessioner men begränsa antalet:
```java
.sessionManagement(session -> session
    .maximumSessions(3)  // Max 3 samtidiga sessioner
    .maxSessionsPreventsLogin(false)  // Ny inloggning kastar ut äldsta
    .expiredUrl("/login?expired")
)
```

### Anpassad hantering av samtidiga sessioner:
```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http
        // ... andra konfigurationer
        .sessionManagement(session -> session
            .maximumSessions(2)
            .maxSessionsPreventsLogin(false)
            .sessionRegistry(sessionRegistry())  // Använd anpassat register
        );
    
    return http.build();
}

@Bean
public SessionRegistry sessionRegistry() {
    return new SessionRegistryImpl();
}
```

## 6. Session fixation-skydd

Session fixation är en attack där angripare försöker använda förutbestämda session-ID:n:

```java
.sessionManagement(session -> session
    .sessionFixation().changeSessionId()  // Rekommenderat (default)
    // .sessionFixation().migrateSession()  // Alternativ
    // .sessionFixation().newSession()      // Skapar helt ny session
    // .sessionFixation().none()            // Inget skydd (farligt!)
)
```

**Vad händer:**
1. Användaren loggar in med session-ID `OLD123`
2. Spring Security skapar nytt session-ID `NEW456`
3. Gammal session invalideras
4. Angripare kan inte använda gamla session-ID:t

## 7. "Remember Me"-funktionalitet

### Grundläggande Remember Me:
```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http
        // ... andra konfigurationer
        .rememberMe(remember -> remember
            .key("mySecretKey")
            .tokenValiditySeconds(86400 * 7)  // 7 dagar
            .userDetailsService(userDetailsService)
        );
    
    return http.build();
}
```

### Persistent Remember Me (med databas):
```java
@Bean
public PersistentTokenRepository persistentTokenRepository() {
    JdbcTokenRepositoryImpl tokenRepository = new JdbcTokenRepositoryImpl();
    tokenRepository.setDataSource(dataSource);
    return tokenRepository;
}

@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http
        // ... andra konfigurationer
        .rememberMe(remember -> remember
            .key("mySecretKey")
            .tokenValiditySeconds(86400 * 30)  // 30 dagar
            .tokenRepository(persistentTokenRepository())
            .userDetailsService(userDetailsService)
        );
    
    return http.build();
}
```

**Databastabell skapas automatiskt:**
```sql
CREATE TABLE persistent_logins (
    username VARCHAR(64) NOT NULL,
    series VARCHAR(64) PRIMARY KEY,
    token VARCHAR(64) NOT NULL,
    last_used TIMESTAMP NOT NULL
);
```

## 8. Session-information i controllers

### Få tag på session-information:
```java
@Controller
public class DashboardController {
    
    @GetMapping("/dashboard")
    public String dashboard(HttpServletRequest request, 
                           Authentication auth, 
                           Model model) {
        
        HttpSession session = request.getSession();
        
        // Session-information
        model.addAttribute("sessionId", session.getId());
        model.addAttribute("sessionCreated", new Date(session.getCreationTime()));
        model.addAttribute("sessionLastAccessed", new Date(session.getLastAccessedTime()));
        model.addAttribute("sessionMaxInactive", session.getMaxInactiveInterval());
        
        // Användar-information
        model.addAttribute("username", auth.getName());
        model.addAttribute("authorities", auth.getAuthorities());
        
        return "dashboard";
    }
    
    @GetMapping("/session-info")
    @ResponseBody
    public Map<String, Object> sessionInfo(HttpServletRequest request, Authentication auth) {
        HttpSession session = request.getSession();
        
        Map<String, Object> info = new HashMap<>();
        info.put("sessionId", session.getId());
        info.put("username", auth.getName());
        info.put("creationTime", new Date(session.getCreationTime()));
        info.put("lastAccessedTime", new Date(session.getLastAccessedTime()));
        info.put("maxInactiveInterval", session.getMaxInactiveInterval());
        info.put("isNew", session.isNew());
        
        return info;
    }
}
```

## 9. Session events och listeners

### Övervaka session-händelser:
```java
@Component
public class SessionEventListener {
    
    private static final Logger logger = LoggerFactory.getLogger(SessionEventListener.class);
    
    @EventListener
    public void handleSessionCreated(HttpSessionCreatedEvent event) {
        HttpSession session = event.getSession();
        logger.info("Session skapad: {}", session.getId());
    }
    
    @EventListener
    public void handleSessionDestroyed(HttpSessionDestroyedEvent event) {
        HttpSession session = event.getSession();
        logger.info("Session förstörd: {}", session.getId());
    }
    
    @EventListener
    public void handleSessionAuthentication(InteractiveAuthenticationSuccessEvent event) {
        Authentication auth = event.getAuthentication();
        logger.info("Användare inloggad: {}", auth.getName());
    }
    
    @EventListener
    public void handleLogout(LogoutSuccessEvent event) {
        Authentication auth = event.getAuthentication();
        logger.info("Användare utloggad: {}", auth.getName());
    }
}
```

## 10. Anpassad session-hantering

### Session-controller för administratörer:
```java
@Controller
@RequestMapping("/admin")
public class SessionAdminController {
    
    @Autowired
    private SessionRegistry sessionRegistry;
    
    @GetMapping("/sessions")
    public String viewActiveSessions(Model model) {
        List<Object> allPrincipals = sessionRegistry.getAllPrincipals();
        
        Map<Object, List<SessionInformation>> principalSessions = new HashMap<>();
        for (Object principal : allPrincipals) {
            List<SessionInformation> sessions = sessionRegistry.getAllSessions(principal, false);
            principalSessions.put(principal, sessions);
        }
        
        model.addAttribute("principalSessions", principalSessions);
        return "admin/sessions";
    }
    
    @PostMapping("/sessions/{sessionId}/invalidate")
    public String invalidateSession(@PathVariable String sessionId) {
        SessionInformation sessionInfo = sessionRegistry.getSessionInformation(sessionId);
        if (sessionInfo != null) {
            sessionInfo.expireNow();
        }
        return "redirect:/admin/sessions";
    }
}
```

## 11. Testa session-konfigurationen

### Test 1: Basic session-funktionalitet
1. Logga in på `/login`
2. Gå till `/session-info` - se session-data
3. Vänta tills session timeout
4. Försök komma åt skyddad sida - omdirigeras till login

### Test 2: Concurrent sessions
1. Logga in i en webbläsare
2. Logga in med samma användare i annan webbläsare
3. Första sessionen ska bli ogiltig (beroende på konfiguration)

### Test 3: Remember Me
1. Logga in med "Remember me" ikryssad
2. Stäng webbläsaren
3. Öppna igen - du ska fortfarande vara inloggad

### Test 4: Session timeout
```properties
# Sätt kort timeout för testning
server.servlet.session.timeout=2m
```

## 12. Session i kluster-miljöer (avancerat)

För applikationer som körs på flera servrar behöver du extern session-lagring:

### Redis session store:
```properties
spring.session.store-type=redis
spring.redis.host=localhost
spring.redis.port=6379
```

### JDBC session store:
```properties
spring.session.store-type=jdbc
spring.session.jdbc.initialize-schema=always
```

## Vanliga problem och lösningar

### Problem 1: Session upphör för snabbt
**Orsak:** För kort timeout i `application.properties`  
**Lösning:** Öka `server.servlet.session.timeout=60m`

### Problem 2: "Remember Me" fungerar inte
**Orsak:** Glömt lägga till checkbox i login-formulär  
**Lösning:** Lägg till `<input type="checkbox" name="remember-me">` i login-form

### Problem 3: Användare kan logga in flera gånger
**Orsak:** `maxSessionsPreventsLogin` är false  
**Lösning:** Ändra till `maxSessionsPreventsLogin(true)`

### Problem 4: Session-information visas inte
**Orsak:** Glömt `HttpSessionEventPublisher` bean  
**Lösning:** Lägg till bean i konfiguration

### Problem 5: CSRF-fel efter session timeout
**Orsak:** Session upphör men CSRF-token finns kvar  
**Lösning:** Konfigurera `.invalidSessionUrl("/login?expired")`

## Sammanfattning

Nu har du:
- ✅ Konfigurerat session timeout och säkerhet
- ✅ Implementerat hantering av samtidiga sessioner
- ✅ Lagt till "Remember Me"-funktionalitet
- ✅ Skapat session-övervakning och events
- ✅ Byggt administrativa verktyg för session-hantering

## Nästa steg

I nästa guide kommer vi att:
- Byta från sessions till JWT (JSON Web Tokens)
- Förstå skillnaderna mellan stateful och stateless authentication
- Konfigurera JWT för REST API:er
- Hantera token refresh och säkerhet

---

**Viktigt:** Sessions är bra för traditionella webbappar, men för REST API:er och mikroservices är JWT ofta ett bättre val.
# OWASP Top 10 - De vanligaste säkerhetshoten

## Vad är OWASP?

**OWASP** (Open Web Application Security Project) är en global ideell organisation som arbetar för att förbättra säkerheten för mjukvara. Sedan 2003 har de publicerat **OWASP Top 10** - en lista över de 10 allvarligaste säkerhetriskerna för webbapplikationer.

### Varför är OWASP Top 10 viktigt?

OWASP Top 10 är:
- **Branschstandard** - Används av säkerhetsexperter världen över
- **Baserat på verklig data** - Analyserar miljontals applikationer
- **Praktiskt användbart** - Ger konkret vägledning för utvecklare
- **Regelbundet uppdaterat** - Speglar dagens hotbild

## Hur hoten har förändrats över tid

### 2003-2010: Grundläggande säkerhet
- **SQL Injection** dominerade
- **Cross-Site Scripting (XSS)** blev ett stort problem
- Fokus på **input-validering**

### 2010-2017: Mer sofistikerade attacker
- **Broken Authentication** blev vanligare
- **Sensitive Data Exposure** ökade
- **Cross-Site Request Forgery (CSRF)** var ett stort hot

### 2017-2021: Moderna webbapplikationer
- **Broken Access Control** blev #1-hotet
- **Security Misconfiguration** ökade dramatiskt
- **Insecure Design** lades till som nytt hot

### 2021-idag: Cloud och API-säkerhet
- **Software Supply Chain** attacker (som SolarWinds)
- **API-specifika sårbarheter** ökar
- **Cloud-konfigurationsfel** blir vanligare

## OWASP Top 10 (2021)

### De 5 viktigaste hoten (som vi fokuserar på):

1. **A01: Broken Access Control** ⚠️ KRITISK
2. **A03: Injection** ⚠️ KRITISK
3. **A07: Cross-Site Scripting (XSS)** ⚠️ KRITISK
4. **A04: Insecure Design** ⚠️ VIKTIGT
5. **A05: Security Misconfiguration** ⚠️ VIKTIGT

### Övriga 5 hot (som vi nämner):

6. **A02: Cryptographic Failures** - Dålig kryptering
7. **A06: Vulnerable and Outdated Components** - Gamla bibliotek
8. **A08: Software and Data Integrity Failures** - Opålitlig kod
9. **A09: Security Logging and Monitoring Failures** - Dålig övervakning
10. **A10: Server-Side Request Forgery (SSRF)** - Serverförfalskning

---

# A01: Broken Access Control

## Vad är det?

Broken Access Control betyder att användare kan komma åt resurser eller utföra åtgärder som de inte ska ha tillgång till.

**Enkelt förklarat:** Det är som att ha en nyckel som öppnar alla dörrar i byggnaden, inte bara din egen lägenhet.

## Så här ser det ut i kod

### ❌ Sårbar kod:
```java
@GetMapping("/user/{userId}/profile")
public String getUserProfile(@PathVariable Long userId, Model model) {
    User user = userService.findById(userId);
    model.addAttribute("user", user);
    return "profile";
}
```

**Problem:** Vem som helst kan ändra URL:en från `/user/123/profile` till `/user/456/profile` och se andras profiler.

### ✅ Säker kod:
```java
@GetMapping("/user/{userId}/profile")
public String getUserProfile(@PathVariable Long userId, 
                           Authentication auth, Model model) {
    User currentUser = (User) auth.getPrincipal();
    
    // Kontrollera att användaren bara ser sin egen profil
    if (!currentUser.getId().equals(userId)) {
        throw new AccessDeniedException("Inte din profil!");
    }
    
    User user = userService.findById(userId);
    model.addAttribute("user", user);
    return "profile";
}
```

## Hur Spring Security hjälper

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(authz -> authz
                .requestMatchers("/admin/**").hasRole("ADMIN")
                .requestMatchers("/user/**").hasRole("USER")
                .anyRequest().authenticated()
            );
        return http.build();
    }
}
```

## Vanliga exempel på Broken Access Control

- **Ändra URL-parametrar:** `/account/123` → `/account/456`
- **Gissa API-endpoints:** `/api/admin/users` (utan att vara admin)
- **Escalera privilegier:** Vanlig användare blir admin
- **Komma åt filer:** `/files/secret.pdf` (ska inte vara tillgänglig)

---

# A03: Injection

## Vad är det?

Injection-attacker händer när skadlig kod "injiceras" i din applikation genom användarinput som behandlas som kod istället för data.

**Enkelt förklarat:** Det är som att be någon säga sitt namn och de svarar med instruktioner som din app följer.

## SQL Injection - vanligaste typen

### ❌ Sårbar kod:
```java
@GetMapping("/search")
public List<Product> searchProducts(@RequestParam String query) {
    String sql = "SELECT * FROM products WHERE name = '" + query + "'";
    return jdbcTemplate.query(sql, new ProductMapper());
}
```

**Attack:** Användaren skickar: `'; DROP TABLE products; --`
**Resultat:** Hela produkttabellen raderas!

### ✅ Säker kod:
```java
@GetMapping("/search")
public List<Product> searchProducts(@RequestParam String query) {
    String sql = "SELECT * FROM products WHERE name = ?";
    return jdbcTemplate.query(sql, new ProductMapper(), query);
}
```

## Andra typer av injection

### Command Injection
```java
// ❌ Farligt
Runtime.getRuntime().exec("ping " + userInput);

// ✅ Säkert - validera input
if (userInput.matches("^[a-zA-Z0-9.-]+$")) {
    Runtime.getRuntime().exec(new String[]{"ping", userInput});
}
```

### LDAP Injection
```java
// ❌ Farligt
String filter = "(&(uid=" + username + ")(password=" + password + "))";

// ✅ Säkert - använd parametrar
String filter = "(&(uid={0})(password={1}))";
```

## Hur Spring förhindrar injection

### JPA/Hibernate (automatiskt säkert):
```java
@Repository
public interface ProductRepository extends JpaRepository<Product, Long> {
    List<Product> findByNameContaining(String name); // Säkert automatiskt
}
```

### Spring Data Query Methods:
```java
@Query("SELECT p FROM Product p WHERE p.name LIKE %?1%")
List<Product> findByNameCustom(String name); // Säkert med ?1 parametrar
```

---

# A07: Cross-Site Scripting (XSS)

## Vad är det?

XSS-attacker händer när skadlig JavaScript-kod injiceras i webbsidor och körs i andra användares webbläsare.

**Enkelt förklarat:** Det är som att lämna ett meddelande på en anslagstavla som gör att alla som läser det utför en handling de inte vet om.

## Typer av XSS

### Reflected XSS
```java
// ❌ Sårbar kod
@GetMapping("/search")
public String search(@RequestParam String q, Model model) {
    model.addAttribute("query", q);  // Direkt till template
    return "search";
}
```

```html
<!-- I template -->
<p>Du sökte efter: <span th:utext="${query}"></span></p>
```

**Attack:** `?q=<script>alert('XSS')</script>`

### Stored XSS
```java
// ❌ Sårbar kod - sparar oskyddat innehåll
@PostMapping("/comment")
public String addComment(@RequestParam String content) {
    Comment comment = new Comment(content);  // Sparar HTML/JS direkt
    commentService.save(comment);
    return "redirect:/comments";
}
```

## Säkert hantering av XSS

### ✅ Använd Thymeleaf's säkra rendering:
```html
<!-- Säkert - escaper automatiskt -->
<p>Du sökte efter: <span th:text="${query}"></span></p>

<!-- Farligt - escaper INTE -->
<p>Du sökte efter: <span th:utext="${query}"></span></p>
```

### ✅ Input-validering:
```java
@PostMapping("/comment")
public String addComment(@RequestParam String content) {
    // Validera och sanitera input
    String cleanContent = sanitizeInput(content);
    Comment comment = new Comment(cleanContent);
    commentService.save(comment);
    return "redirect:/comments";
}

private String sanitizeInput(String input) {
    return input.replaceAll("<script.*?>.*?</script>", "")
               .replaceAll("<.*?>", ""); // Ta bort alla HTML-taggar
}
```

## Hur Spring Security hjälper mot XSS

```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http
        .headers(headers -> headers
            .contentTypeOptions()  // Förhindrar MIME-sniffing
            .xssProtection()       // Aktiverar XSS-skydd i webbläsare
        );
    return http.build();
}
```

---

# A04: Insecure Design

## Vad är det?

Insecure Design handlar om säkerhetsbrister som är inbyggda i applikationens grundläggande design och arkitektur.

**Enkelt förklarat:** Det är som att bygga ett hus utan att tänka på säkerhet - även om du sätter bra lås efteråt, är grunddesignen osäker.

## Exempel på Insecure Design

### Dålig lösenordsåterställning
```java
// ❌ Osäker design
@PostMapping("/reset-password")
public String resetPassword(@RequestParam String email) {
    User user = userService.findByEmail(email);
    if (user != null) {
        // Skickar lösenordet i klartext via email!
        emailService.sendPassword(email, user.getPassword());
    }
    return "password-sent";
}
```

### ✅ Säker design:
```java
@PostMapping("/reset-password")
public String resetPassword(@RequestParam String email) {
    User user = userService.findByEmail(email);
    if (user != null) {
        // Skapa en säker reset-token
        String resetToken = generateSecureToken();
        user.setResetToken(resetToken);
        user.setResetTokenExpiry(LocalDateTime.now().plusHours(1));
        userService.save(user);
        
        // Skicka länk med token
        String resetUrl = "https://oursite.com/reset?token=" + resetToken;
        emailService.sendResetLink(email, resetUrl);
    }
    return "reset-link-sent";
}
```

## Vanliga designproblem

### Ingen rate limiting
```java
// ❌ Tillåter brute force-attacker
@PostMapping("/login")
public String login(@RequestParam String username, 
                   @RequestParam String password) {
    // Ingen begränsning av antal försök
    if (userService.authenticate(username, password)) {
        return "dashboard";
    }
    return "login-error";
}
```

### Ingen logging av säkerhetshändelser
```java
// ❌ Ingen övervakning
@PostMapping("/admin/delete-user")
public String deleteUser(@RequestParam Long userId) {
    userService.delete(userId);  // Loggas inte!
    return "user-deleted";
}
```

## Säker design-principer

1. **Fail securely** - När något går fel, var säker
2. **Defense in depth** - Flera säkerhetslager
3. **Least privilege** - Minimal behörighet
4. **Separation of duties** - Dela upp känsliga operationer

---

# A05: Security Misconfiguration

## Vad är det?

Security Misconfiguration händer när säkerhetsinställningar är felaktiga, inkompletta eller använder osäkra standardinställningar.

**Enkelt förklarat:** Det är som att ha ett bra lås men glömma att använda det, eller ha larmet påslaget men med fel kod.

## Vanliga konfigurationsfel

### Utvecklingsläge i produktion
```properties
# ❌ application.properties i produktion
spring.profiles.active=dev
spring.h2.console.enabled=true
server.error.include-stacktrace=always
logging.level.org.springframework.security=DEBUG
```

### ✅ Säker produktionskonfiguration:
```properties
# application-prod.properties
spring.profiles.active=prod
server.error.include-stacktrace=never
logging.level.org.springframework.security=WARN
management.endpoints.enabled-by-default=false
```

### Osäkra Spring Security-inställningar
```java
// ❌ Osäker utvecklingskonfiguration
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http
        .csrf().disable()  // Stänger av CSRF-skydd
        .authorizeHttpRequests(authz -> authz
            .anyRequest().permitAll()  // Alla får komma åt allt
        );
    return http.build();
}
```

## Hur Spring Security hjälper

### Säkra standardinställningar
```java
// ✅ Spring Security är säkert som standard
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http
        .authorizeHttpRequests(authz -> authz
            .anyRequest().authenticated()  // Kräver inloggning som standard
        )
        .formLogin()  // CSRF-skydd aktiverat automatiskt
        .headers(headers -> headers
            .frameOptions().deny()  // Förhindrar clickjacking
            .contentTypeOptions()   // Förhindrar MIME-sniffing
        );
    return http.build();
}
```

### Säkra headers automatiskt
Spring Security lägger automatiskt till:
- `X-Frame-Options: DENY`
- `X-Content-Type-Options: nosniff`
- `X-XSS-Protection: 1; mode=block`
- `Referrer-Policy: strict-origin-when-cross-origin`

---

# Övriga 5 hot (kort översikt)

## A02: Cryptographic Failures
**Problem:** Svag kryptering, lösenord i klartext, osäkra protokoll  
**Lösning:** Använd HTTPS, BCrypt för lösenord, starka krypteringsalgoritmer

## A06: Vulnerable and Outdated Components
**Problem:** Gamla bibliotek med kända säkerhetshål  
**Lösning:** Håll dependencies uppdaterade, använd sårbarhetsscanning

## A08: Software and Data Integrity Failures
**Problem:** Opålitlig kod från tredje part, osäkra updates  
**Lösning:** Verifiera signaturer, använd Subresource Integrity

## A09: Security Logging and Monitoring Failures
**Problem:** Ingen övervakning av säkerhetshändelser  
**Lösning:** Logga säkerhetsrelevanta händelser, sätt upp varningar

## A10: Server-Side Request Forgery (SSRF)
**Problem:** Servern kan luras göra requests åt angripare  
**Lösning:** Validera och begränsa utgående requests

---

# Sammanfattning

## Viktigast att komma ihåg:

1. **Broken Access Control** - Kontrollera behörigheter på server-sidan
2. **Injection** - Använd parametriserade queries och validera input
3. **XSS** - Escapa output och validera input
4. **Insecure Design** - Tänk säkerhet från början
5. **Security Misconfiguration** - Använd säkra inställningar i produktion

## Spring Security hjälper med:
- Säkra standardinställningar
- Automatiska säkerhetsheaders
- CSRF-skydd
- Session-hantering
- Input-validering stöd

## Nästa steg:
- Implementera konkreta säkerhetslösningar
- Testa din applikation mot OWASP Top 10
- Sätt upp säkerhetsmonitoring
- Regelbundna säkerhetsgranskningar

---

**Kom ihåg:** OWASP Top 10 är inte en komplett säkerhetslösning, men en bra utgångspunkt för att bygga säkra webbapplikationer!
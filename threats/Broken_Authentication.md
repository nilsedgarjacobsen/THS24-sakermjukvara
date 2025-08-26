# Vanliga Säkerhetsattacker och Skydd - Spring Boot

## 1. Credential Stuffing

### Vad är det?
Angripare använder stulna användarnamn/lösenord från andra webbplatser och testar dem på din sida. Om användare återanvänder lösenord lyckas attacken.

**Exempel:** Användaren har samma lösenord på både Gmail och din app. Gmail får dataintrång → angriparen testar samma combo på din sida → lyckas!

### Så Ser Attacken Ut
```
POST /login
username: john@email.com
password: password123

POST /login  
username: mary@email.com
password: qwerty123

POST /login
username: admin@company.com  
password: admin123
// ... tusentals försök med stulna credentials
```

### Skydd med Spring Boot
```java
@Component
public class LoginAttemptService {
    
    private Map<String, Integer> attempts = new ConcurrentHashMap<>();
    
    public void loginFailed(String username) {
        // Räkna misslyckade försök per användare
        attempts.put(username, attempts.getOrDefault(username, 0) + 1);
    }
    
    public boolean isBlocked(String username) {
        // Blockera efter 5 misslyckade försök
        return attempts.getOrDefault(username, 0) >= 5;
    }
    
    public void loginSucceeded(String username) {
        // Nollställ räknaren vid lyckad inloggning
        attempts.remove(username);
    }
}

@RestController
public class LoginController {
    
    @Autowired
    private LoginAttemptService attemptService;
    
    @PostMapping("/login")
    public ResponseEntity<?> login(@RequestBody LoginRequest request) {
        
        // Kolla om kontot är blockerat
        if (attemptService.isBlocked(request.getUsername())) {
            return ResponseEntity.status(429) // Too Many Requests
                .body("Kontot är temporärt blockerat");
        }
        
        // Försök logga in användaren
        if (authService.authenticate(request.getUsername(), request.getPassword())) {
            attemptService.loginSucceeded(request.getUsername());
            return ResponseEntity.ok("Inloggning lyckades");
        } else {
            attemptService.loginFailed(request.getUsername());
            return ResponseEntity.status(401).body("Fel användarnamn eller lösenord");
        }
    }
}
```

## 2. Brute Force Attacker

### Vad är det?
Angriparen testar systematiskt tusentals lösenordskombinationer för att gissa rätt lösenord.

**Exempel:** Testar "password1", "password2", "123456", "qwerty" osv. tills rätt lösenord hittas.

### Så Ser Attacken Ut
```
POST /login
username: admin
password: admin

POST /login
username: admin  
password: password

POST /login
username: admin
password: 123456
// ... kontinuerligt tills rätt lösenord hittas
```

### Skydd med Rate Limiting
```java
@Component
public class RateLimitService {
    
    private Map<String, List<Long>> requestTimes = new ConcurrentHashMap<>();
    private final int MAX_REQUESTS = 10;        // Max 10 försök
    private final long TIME_WINDOW = 60000;     // Per minut
    
    public boolean isAllowed(String clientIP) {
        long currentTime = System.currentTimeMillis();
        
        // Hämta tidigare requests från denna IP
        List<Long> times = requestTimes.getOrDefault(clientIP, new ArrayList<>());
        
        // Ta bort gamla requests (äldre än 1 minut)
        times.removeIf(time -> currentTime - time > TIME_WINDOW);
        
        // Kolla om vi är under gränsen
        if (times.size() >= MAX_REQUESTS) {
            return false; // För många requests
        }
        
        // Lägg till denna request
        times.add(currentTime);
        requestTimes.put(clientIP, times);
        
        return true;
    }
}

@RestController
public class LoginController {
    
    @PostMapping("/login")
    public ResponseEntity<?> login(HttpServletRequest request, 
                                  @RequestBody LoginRequest loginReq) {
        
        String clientIP = getClientIP(request);
        
        // Kolla rate limit
        if (!rateLimitService.isAllowed(clientIP)) {
            return ResponseEntity.status(429)
                .body("För många inloggningsförsök. Försök igen senare.");
        }
        
        // Fortsätt med normal inloggning...
    }
    
    private String getClientIP(HttpServletRequest request) {
        // Hantera proxy/load balancer headers
        String xForwardedFor = request.getHeader("X-Forwarded-For");
        if (xForwardedFor != null) {
            return xForwardedFor.split(",")[0].trim();
        }
        return request.getRemoteAddr();
    }
}
```

## 3. Session Hijacking

### Vad är det?
Angriparen stjäl en användares session-ID och kan då agera som den användaren utan att känna till lösenordet.

**Exempel:** Användaren loggar in på osäkert WiFi → angriparen sniffar session-cookien → använder den för att komma åt kontot.

### Så Fungerar Attacken
```
1. Användare loggar in → får session-ID: JSESSIONID=ABC123
2. Angriparen stjäl ABC123 (via WiFi sniffing, XSS, etc.)
3. Angriparen skickar requests med Cookie: JSESSIONID=ABC123
4. Servern tror angriparen är den riktiga användaren
```

### Skydd mot Session Hijacking
```java
@Configuration
public class SecurityConfig {
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .sessionManagement(session -> session
                // Skapa ny session vid inloggning (förhindrar session fixation)
                .sessionFixation().migrateSession()
                
                // Max en aktiv session per användare
                .maximumSessions(1)
                .maxSessionsPreventsLogin(false)
            )
            
            // Säkra cookies
            .headers(headers -> headers
                .httpStrictTransportSecurity(hsts -> hsts
                    .maxAgeInSeconds(31536000)        // 1 år
                    .includeSubdomains(true)
                )
            );
            
        return http.build();
    }
    
    @Bean
    public CookieSerializer cookieSerializer() {
        DefaultCookieSerializer serializer = new DefaultCookieSerializer();
        serializer.setUseSecureCookie(true);      // Bara över HTTPS
        serializer.setUseHttpOnlyCookie(true);    // Inte tillgänglig via JavaScript
        serializer.setSameSite("Strict");         // CSRF-skydd
        return serializer;
    }
}

@Component
public class SessionSecurityService {
    
    public void validateSession(HttpServletRequest request) {
        HttpSession session = request.getSession(false);
        
        if (session != null) {
            // Kolla IP-address (enkel kontroll)
            String currentIP = request.getRemoteAddr();
            String sessionIP = (String) session.getAttribute("IP_ADDRESS");
            
            if (sessionIP != null && !sessionIP.equals(currentIP)) {
                // IP har ändrats - möjlig hijacking
                session.invalidate();
                throw new SecurityException("Säkerhetsvarning: Session ogiltig");
            }
            
            // Uppdatera senaste aktivitet
            session.setAttribute("LAST_ACTIVITY", System.currentTimeMillis());
        }
    }
}
```

## 4. Weak Passwords (Svaga Lösenord)

### Vad är problemet?
Användare väljer lösenord som är lätta att gissa eller knäcka: "password", "123456", "qwerty".

**Vanliga svaga lösenord:**
- Korta lösenord (under 8 tecken)
- Bara siffror eller bara bokstäver
- Vanliga ord eller fraser
- Personlig information (födelsedatum, namn)

### Lösenordsskydd
```java
@Component
public class PasswordStrengthValidator {
    
    // Lista över vanliga dåliga lösenord
    private Set<String> commonPasswords = Set.of(
        "password", "123456", "qwerty", "admin", "letmein",
        "welcome", "monkey", "dragon", "pass", "master"
    );
    
    public PasswordValidationResult validatePassword(String password) {
        List<String> errors = new ArrayList<>();
        
        // Längd
        if (password.length() < 8) {
            errors.add("Lösenordet måste vara minst 8 tecken");
        }
        
        // Komplexitet
        if (!password.matches(".*[A-Z].*")) {
            errors.add("Lösenordet måste innehålla minst en stor bokstav");
        }
        
        if (!password.matches(".*[a-z].*")) {
            errors.add("Lösenordet måste innehålla minst en liten bokstav");
        }
        
        if (!password.matches(".*[0-9].*")) {
            errors.add("Lösenordet måste innehålla minst en siffra");
        }
        
        if (!password.matches(".*[!@#$%^&*()_+\\-=\\[\\]{};':\"\\\\|,.<>\\?].*")) {
            errors.add("Lösenordet måste innehålla minst ett specialtecken");
        }
        
        // Vanliga lösenord
        if (commonPasswords.contains(password.toLowerCase())) {
            errors.add("Detta lösenord är för vanligt och osäkert");
        }
        
        // Repetition
        if (hasRepeatingPatterns(password)) {
            errors.add("Undvik upprepande mönster (aaa, 123, abc)");
        }
        
        return new PasswordValidationResult(errors.isEmpty(), errors);
    }
    
    private boolean hasRepeatingPatterns(String password) {
        // Kolla efter upprepningar som "aaa" eller "123"
        for (int i = 0; i <= password.length() - 3; i++) {
            String substring = password.substring(i, i + 3);
            
            // Samma tecken upprepat
            if (substring.charAt(0) == substring.charAt(1) && 
                substring.charAt(1) == substring.charAt(2)) {
                return true;
            }
            
            // Sekventiella tecken
            if (isSequential(substring)) {
                return true;
            }
        }
        return false;
    }
    
    private boolean isSequential(String str) {
        return str.equals("123") || str.equals("abc") || str.equals("xyz") ||
               str.equals("789") || str.equals("456");
    }
}

@RestController
public class PasswordController {
    
    @PostMapping("/change-password")
    public ResponseEntity<?> changePassword(@RequestBody ChangePasswordRequest request) {
        
        // Validera lösenordsstyrka
        PasswordValidationResult result = passwordValidator.validatePassword(request.getNewPassword());
        
        if (!result.isValid()) {
            return ResponseEntity.badRequest()
                .body(Map.of("errors", result.getErrors()));
        }
        
        // Fortsätt med lösenordsändring...
        userService.changePassword(request.getUsername(), request.getNewPassword());
        
        return ResponseEntity.ok("Lösenord uppdaterat");
    }
}
```

## Sammanfattning - Skydda Dig

### ✅ Generella Skydd
- **Rate limiting** - begränsa antal requests per IP/användare
- **Account lockout** - blockera konton efter för många misslyckade försök
- **Strong password policies** - kräv komplexa lösenord
- **HTTPS överallt** - kryptera all trafik
- **Secure session handling** - säkra cookies och session-hantering

### ✅ Monitoring och Logging
```java
@Component
@Slf4j
public class SecurityMonitor {
    
    public void logSuspiciousActivity(String event, String username, String ip) {
        log.warn("SECURITY: {} - User: {} - IP: {} - Time: {}", 
            event, username, ip, LocalDateTime.now());
        
        // Skicka till säkerhetsmonitoring-system
        alertService.sendSecurityAlert(event, username, ip);
    }
}
```

### 🚨 Tecken på Attacker
- Många inloggningsförsök från samma IP
- Inloggningsförsök med vanliga lösenord
- Sessioner från olika IP-adresser samtidigt
- Ovanliga aktivitetsmönster från kända användare

**Kom ihåg:** Säkerhet handlar om lager. Ingen enskild åtgärd är tillräcklig - kombinera flera skydd för bästa resultat!
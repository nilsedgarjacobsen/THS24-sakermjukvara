# XSS-skydd i Spring Boot Backend - Praktisk guide

## Vad är XSS?

XSS (Cross-Site Scripting) uppstår när en webbapplikation tar användardata och visar det som HTML/JavaScript utan att kontrollera att det är säkert först.

**Enkelt exempel:**
1. Användare skriver: `<script>alert('Hacked!')</script>` i ett formulär
2. Om backend visar detta direkt som HTML: `return "<h1>" + userInput + "</h1>";`
3. Webbläsaren kör JavaScript-koden och visar popup

**Varför är det farligt?**
- Stjäla användarnas cookies/sessioner
- Omdirigera till skadliga sidor
- Läsa känslig information från sidan
- Utföra handlingar åt användaren (skicka pengar, ändra lösenord)

**Viktigt:** XSS är bara ett problem när din app genererar HTML/JavaScript. JSON APIs har inte detta problem.

## När är XSS verkligen ett hot?

XSS är **ENDAST** ett problem när din backend skickar användardata direkt till webbläsaren som HTML/JavaScript. 

### ❌ XSS-risk: När backend renderar HTML

```java
// FARLIGT - Backend genererar HTML med användardata
@GetMapping("/profile/{userId}")
public String getProfile(@PathVariable Long userId) {
    User user = userRepository.findById(userId);
    return "<h1>Välkommen " + user.getName() + "</h1>"; // XSS-risk!
}

// FARLIGT - Bygger JavaScript med användardata
@GetMapping("/dashboard.js")
public ResponseEntity<String> getDashboardScript() {
    String userName = getCurrentUser().getName();
    String js = "var currentUser = '" + userName + "';"; // XSS-risk!
    return ResponseEntity.ok()
        .contentType(MediaType.valueOf("application/javascript"))
        .body(js);
}
```

### ✅ INGEN XSS-risk: Modern JSON API

```java
// SÄKERT - Returnerar bara JSON
@RestController
public class UserController {
    
    @GetMapping("/api/users/{id}")
    public ResponseEntity<UserDto> getUser(@PathVariable Long id) {
        User user = userRepository.findById(id);
        return ResponseEntity.ok(new UserDto(user));
        // JSON serialisering är automatiskt säker
    }
    
    @PostMapping("/api/comments")
    public ResponseEntity<CommentDto> createComment(@RequestBody CommentDto comment) {
        Comment saved = commentRepository.save(new Comment(comment.getText()));
        return ResponseEntity.ok(new CommentDto(saved));
        // Även skadlig data är säker som JSON
    }
}
```

## Verkliga XSS-hot scenarion

### Scenario 1: Email med användardata

```java
// FARLIGT - Om emailen innehåller HTML
@Service
public class NotificationService {
    
    public void sendWelcomeEmail(User user) {
        String htmlContent = "<h1>Hej " + user.getName() + "!</h1>"; // XSS-risk
        emailService.sendHtml(user.getEmail(), htmlContent);
    }
}

// SÄKERT - Escape användardata
public void sendWelcomeEmail(User user) {
    String safeName = HtmlUtils.htmlEscape(user.getName());
    String htmlContent = "<h1>Hej " + safeName + "!</h1>";
    emailService.sendHtml(user.getEmail(), htmlContent);
}
```

### Scenario 2: Felmeddelanden som HTML

```java
// FARLIGT - Visa användarinput i felmeddelanden
@ExceptionHandler(ValidationException.class)
public ResponseEntity<String> handleValidation(ValidationException e) {
    return ResponseEntity.badRequest()
        .contentType(MediaType.TEXT_HTML)
        .body("<div class='error'>Fel i fält: " + e.getFieldName() + "</div>");
}

// SÄKERT - JSON felmeddelanden  
@ExceptionHandler(ValidationException.class)
public ResponseEntity<ErrorResponse> handleValidation(ValidationException e) {
    return ResponseEntity.badRequest()
        .body(new ErrorResponse("Validering misslyckades", e.getFieldName()));
}
```

### Scenario 3: Admin-interface som genererar HTML

```java
// FARLIGT - Admin ser användardata som HTML
@GetMapping("/admin/users")
public String getUserTable() {
    List<User> users = userRepository.findAll();
    StringBuilder html = new StringBuilder("<table>");
    
    for (User user : users) {
        html.append("<tr><td>")
            .append(user.getName()) // XSS-risk om användare har skadligt namn
            .append("</td></tr>");
    }
    return html.toString();
}
```

## När du INTE behöver tänka på XSS

### ✅ JSON API (99% av moderna appar)

```java
@RestController
public class ApiController {
    
    // Alla dessa är automatiskt säkra
    @GetMapping("/api/posts")
    public List<PostDto> getPosts() { ... }
    
    @PostMapping("/api/posts") 
    public PostDto createPost(@RequestBody PostDto post) { ... }
    
    @GetMapping("/api/search")
    public SearchResult search(@RequestParam String query) { ... }
}
```

**Varför är JSON säkert?**
- Webbläsaren tolkar JSON som data, inte kod
- Jackson serialisering escapar automatiskt
- Frontend (React/Vue/Angular) hanterar XSS-skydd

### ✅ Ren datalagring

```java
@Entity
public class Comment {
    private String text; // Kan innehålla vad som helst
    // JPA sparar raw data säkert
}

@Repository
public interface CommentRepository extends JpaRepository<Comment, Long> {
    // Alla JPA operationer är säkra
    List<Comment> findByTextContaining(String text);
}
```

### ✅ Intern API-kommunikation

```java
// Service-to-service kommunikation
@Service  
public class UserService {
    
    public void processUserData(String userData) {
        // Intern logik - ingen XSS-risk
        logRepository.save(new LogEntry(userData));
        cacheService.store("user_" + userData, someValue);
    }
}
```

## Praktisk checklista

### 🚨 Kontrollera för XSS-risk:

1. **Returnerar du HTML från endpoints?**
   ```java
   return "<html>..."; // RISK
   ```

2. **Bygger du JavaScript dynamiskt?**
   ```java
   return "var data = '" + userInput + "';"; // RISK
   ```

3. **Skickar du HTML-email med användardata?**
   ```java
   emailService.sendHtml(htmlWithUserData); // RISK
   ```

4. **Har du servlets som skriver direkt till response?**
   ```java
   response.getWriter().write("<h1>" + userInput + "</h1>"); // RISK
   ```

### ✅ Inga XSS-problem om du:

1. **Bara returnerar JSON**
2. **Låter frontend hantera all rendering**  
3. **Använder JPA för datalagring**
4. **Skickar plain text emails**

## Spring Security konfiguration för JSON API

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
            .headers(headers -> headers
                .contentSecurityPolicy("default-src 'self'")
                .and()
                .frameOptions().deny()
            )
            .sessionManagement(session -> 
                session.sessionCreationPolicy(SessionCreationPolicy.STATELESS)
            )
            .build();
    }
}
```

## Sammanfattning

**XSS är BARA ett problem när:**
- Din Spring Boot app genererar HTML/JavaScript med användardata
- Du bygger old-school server-rendered webbsidor

**XSS är INTE ett problem när:**
- Du bygger JSON REST API (moderna appar)
- Frontend (React/Vue/Angular) hanterar all rendering
- Du bara sparar och hämtar data via JPA

**Bottom line:** Om du bygger en modern JSON API backend, fokusera på andra säkerhetsproblem som authentication, authorization och SQL injection istället.

# Spring Security - Från sessions till JWT

## Introduktion

I denna guide kommer vi att byta från session-baserad autentisering till JWT (JSON Web Tokens). Du kommer att lära dig:
- Skillnaden mellan stateful (sessions) och stateless (JWT) authentication
- Vad JWT är och hur det fungerar
- Implementera JWT i Spring Security
- Skapa JWT-baserade REST API:er
- Hantera token refresh och säkerhet

**Förutsättningar:** Ett Spring Boot-projekt med Spring Security, databas-integration, password encoding och session-konfiguration redan implementerat.

## 1. Sessions vs JWT - Vad är skillnaden?

### Sessions (Stateful)
```
Webbläsare                    Server
     │                           │
     │ POST /login               │
     │ ────────────────────────► │ Sparar session i minne/databas
     │ ◄──────────────────────── │ Set-Cookie: JSESSIONID=ABC123
     │                           │
     │ GET /api/users             │
     │ Cookie: JSESSIONID=ABC123  │
     │ ────────────────────────► │ Slår upp session ABC123
     │                           │ Hittar användare i session-store
     │ ◄──────────────────────── │ Returnerar data
```

### JWT (Stateless)
```
Klient                        Server
     │                           │
     │ POST /login               │
     │ ────────────────────────► │ Validerar lösenord
     │ ◄──────────────────────── │ Returnerar JWT token
     │                           │
     │ GET /api/users             │
     │ Authorization: Bearer JWT  │
     │ ────────────────────────► │ Validerar JWT signature
     │                           │ Läser användarinfo från token
     │ ◄──────────────────────── │ Returnerar data
```

### Fördelar och nackdelar

**Sessions (Stateful):**
- ✅ Enkel att implementera
- ✅ Kan enkelt invalideras (logout)
- ✅ Mindre data över nätverket
- ❌ Kräver server-side lagring
- ❌ Svårt att skala horisontellt
- ❌ Fungerar dåligt för mobile apps

**JWT (Stateless):**
- ✅ Ingen server-side lagring
- ✅ Perfekt för mikroservices
- ✅ Fungerar bra för mobile apps
- ✅ Kan innehålla användardata
- ❌ Svårare att invalidera
- ❌ Större tokens över nätverket
- ❌ Mer komplext att implementera

## 2. Vad är JWT?

JWT (JSON Web Token) är ett säkert sätt att skicka information mellan parter som en JSON-objekt.

### JWT-struktur:
```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJhbm5hIiwiaWF0IjoxNjE2MjM5MDIyfQ.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c

│─────────── HEADER ──────────│─────────── PAYLOAD ─────────│─────── SIGNATURE ──────│
```

### Dekoderat:
```json
// HEADER
{
  "alg": "HS256",
  "typ": "JWT"
}

// PAYLOAD
{
  "sub": "anna",
  "iat": 1616239022,
  "exp": 1616325422,
  "authorities": ["ROLE_USER"]
}

// SIGNATURE
HMACSHA256(
  base64UrlEncode(header) + "." + base64UrlEncode(payload),
  secret
)
```

## 3. Lägg till JWT-dependencies

I din `pom.xml`, lägg till:
```xml
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-api</artifactId>
    <version>0.11.5</version>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-impl</artifactId>
    <version>0.11.5</version>
    <scope>runtime</scope>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-jackson</artifactId>
    <version>0.11.5</version>
    <scope>runtime</scope>
</dependency>
```

## 4. Skapa JWT-verktygsklassen

```java
@Component
public class JwtUtils {
    
    private static final Logger logger = LoggerFactory.getLogger(JwtUtils.class);
    
    @Value("${app.jwtSecret:mySecretKey}")
    private String jwtSecret;
    
    @Value("${app.jwtExpirationMs:86400000}")
    private int jwtExpirationMs; // 24 timmar
    
    public String generateJwtToken(Authentication authentication) {
        UserDetails userPrincipal = (UserDetails) authentication.getPrincipal();
        
        return Jwts.builder()
                .setSubject(userPrincipal.getUsername())
                .setIssuedAt(new Date())
                .setExpiration(new Date(System.currentTimeMillis() + jwtExpirationMs))
                .signWith(SignatureAlgorithm.HS512, jwtSecret)
                .compact();
    }
    
    public String getUserNameFromJwtToken(String token) {
        return Jwts.parser()
                .setSigningKey(jwtSecret)
                .parseClaimsJws(token)
                .getBody()
                .getSubject();
    }
    
    public boolean validateJwtToken(String authToken) {
        try {
            Jwts.parser().setSigningKey(jwtSecret).parseClaimsJws(authToken);
            return true;
        } catch (SignatureException e) {
            logger.error("Invalid JWT signature: {}", e.getMessage());
        } catch (MalformedJwtException e) {
            logger.error("Invalid JWT token: {}", e.getMessage());
        } catch (ExpiredJwtException e) {
            logger.error("JWT token is expired: {}", e.getMessage());
        } catch (UnsupportedJwtException e) {
            logger.error("JWT token is unsupported: {}", e.getMessage());
        } catch (IllegalArgumentException e) {
            logger.error("JWT claims string is empty: {}", e.getMessage());
        }
        return false;
    }
}
```

## 5. Skapa JWT Authentication Filter

```java
public class AuthTokenFilter extends OncePerRequestFilter {
    
    @Autowired
    private JwtUtils jwtUtils;
    
    @Autowired
    private CustomUserDetailsService userDetailsService;
    
    private static final Logger logger = LoggerFactory.getLogger(AuthTokenFilter.class);
    
    @Override
    protected void doFilterInternal(HttpServletRequest request, 
                                   HttpServletResponse response, 
                                   FilterChain filterChain) throws ServletException, IOException {
        try {
            String jwt = parseJwt(request);
            if (jwt != null && jwtUtils.validateJwtToken(jwt)) {
                String username = jwtUtils.getUserNameFromJwtToken(jwt);
                
                UserDetails userDetails = userDetailsService.loadUserByUsername(username);
                UsernamePasswordAuthenticationToken authentication = 
                    new UsernamePasswordAuthenticationToken(userDetails, null, userDetails.getAuthorities());
                authentication.setDetails(new WebAuthenticationDetailsSource().buildDetails(request));
                
                SecurityContextHolder.getContext().setAuthentication(authentication);
            }
        } catch (Exception e) {
            logger.error("Cannot set user authentication: {}", e.getMessage());
        }
        
        filterChain.doFilter(request, response);
    }
    
    private String parseJwt(HttpServletRequest request) {
        String headerAuth = request.getHeader("Authorization");
        
        if (StringUtils.hasText(headerAuth) && headerAuth.startsWith("Bearer ")) {
            return headerAuth.substring(7);
        }
        
        return null;
    }
}
```

## 6. Uppdatera SecurityConfig för JWT

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {
    
    @Autowired
    private CustomUserDetailsService userDetailsService;
    
    @Autowired
    private AuthEntryPointJwt unauthorizedHandler;
    
    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
    
    @Bean
    public AuthTokenFilter authenticationJwtTokenFilter() {
        return new AuthTokenFilter();
    }
    
    @Bean
    public DaoAuthenticationProvider authenticationProvider() {
        DaoAuthenticationProvider authProvider = new DaoAuthenticationProvider();
        authProvider.setUserDetailsService(userDetailsService);
        authProvider.setPasswordEncoder(passwordEncoder());
        return authProvider;
    }
    
    @Bean
    public AuthenticationManager authenticationManager(AuthenticationConfiguration authConfig) throws Exception {
        return authConfig.getAuthenticationManager();
    }
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http.cors().and().csrf().disable()
            .exceptionHandling().authenticationEntryPoint(unauthorizedHandler).and()
            .sessionManagement().sessionCreationPolicy(SessionCreationPolicy.STATELESS).and()
            .authorizeHttpRequests(authz -> authz
                .requestMatchers("/api/auth/**").permitAll()
                .requestMatchers("/public/**").permitAll()
                .anyRequest().authenticated()
            );
        
        http.authenticationProvider(authenticationProvider());
        http.addFilterBefore(authenticationJwtTokenFilter(), UsernamePasswordAuthenticationFilter.class);
        
        return http.build();
    }
}
```

**Viktiga ändringar:**
- `.sessionCreationPolicy(SessionCreationPolicy.STATELESS)` - Ingen session skapas
- `.csrf().disable()` - CSRF inte nödvändigt för stateless API:er
- `addFilterBefore()` - JWT-filter körs före standard authentication

## 7. Skapa Authentication Entry Point

```java
@Component
public class AuthEntryPointJwt implements AuthenticationEntryPoint {
    
    private static final Logger logger = LoggerFactory.getLogger(AuthEntryPointJwt.class);
    
    @Override
    public void commence(HttpServletRequest request, HttpServletResponse response,
                        AuthenticationException authException) throws IOException, ServletException {
        logger.error("Unauthorized error: {}", authException.getMessage());
        response.sendError(HttpServletResponse.SC_UNAUTHORIZED, "Error: Unauthorized");
    }
}
```

## 8. Skapa JWT Authentication Controller

```java
@RestController
@RequestMapping("/api/auth")
public class AuthController {
    
    @Autowired
    private AuthenticationManager authenticationManager;
    
    @Autowired
    private UserRepository userRepository;
    
    @Autowired
    private PasswordEncoder encoder;
    
    @Autowired
    private JwtUtils jwtUtils;
    
    @PostMapping("/login")
    public ResponseEntity<?> authenticateUser(@RequestBody LoginRequest loginRequest) {
        
        Authentication authentication = authenticationManager.authenticate(
                new UsernamePasswordAuthenticationToken(loginRequest.getUsername(), loginRequest.getPassword()));
        
        SecurityContextHolder.getContext().setAuthentication(authentication);
        String jwt = jwtUtils.generateJwtToken(authentication);
        
        UserDetails userDetails = (UserDetails) authentication.getPrincipal();
        List<String> roles = userDetails.getAuthorities().stream()
                .map(item -> item.getAuthority())
                .collect(Collectors.toList());
        
        return ResponseEntity.ok(new JwtResponse(jwt,
                                               userDetails.getUsername(),
                                               roles));
    }
    
    @PostMapping("/register")
    public ResponseEntity<?> registerUser(@RequestBody SignupRequest signUpRequest) {
        if (userRepository.existsByUsername(signUpRequest.getUsername())) {
            return ResponseEntity.badRequest()
                    .body(new MessageResponse("Error: Username is already taken!"));
        }
        
        // Skapa ny användare
        User user = new User(signUpRequest.getUsername(),
                           encoder.encode(signUpRequest.getPassword()),
                           "USER");
        
        userRepository.save(user);
        
        return ResponseEntity.ok(new MessageResponse("User registered successfully!"));
    }
}
```

## 9. Skapa Request och Response-klasser

### LoginRequest:
```java
public class LoginRequest {
    private String username;
    private String password;
    
    // Konstruktorer, getters och setters
    public LoginRequest() {}
    
    public String getUsername() { return username; }
    public void setUsername(String username) { this.username = username; }
    
    public String getPassword() { return password; }
    public void setPassword(String password) { this.password = password; }
}
```

### JwtResponse:
```java
public class JwtResponse {
    private String token;
    private String type = "Bearer";
    private String username;
    private List<String> roles;
    
    public JwtResponse(String accessToken, String username, List<String> roles) {
        this.token = accessToken;
        this.username = username;
        this.roles = roles;
    }
    
    // Getters och setters
    public String getAccessToken() { return token; }
    public void setAccessToken(String accessToken) { this.token = accessToken; }
    
    public String getTokenType() { return type; }
    public void setTokenType(String tokenType) { this.type = tokenType; }
    
    public String getUsername() { return username; }
    public void setUsername(String username) { this.username = username; }
    
    public List<String> getRoles() { return roles; }
    public void setRoles(List<String> roles) { this.roles = roles; }
}
```

## 10. Skapa skyddade API-endpoints

```java
@RestController
@RequestMapping("/api")
public class UserController {
    
    @Autowired
    private UserRepository userRepository;
    
    @GetMapping("/users")
    public ResponseEntity<List<User>> getAllUsers() {
        List<User> users = userRepository.findAll();
        return ResponseEntity.ok(users);
    }
    
    @GetMapping("/profile")
    public ResponseEntity<?> getUserProfile(Authentication authentication) {
        String username = authentication.getName();
        User user = userRepository.findByUsername(username)
                .orElseThrow(() -> new RuntimeException("User not found"));
        
        return ResponseEntity.ok(new UserInfoResponse(user.getUsername(), user.getRole()));
    }
    
    @PreAuthorize("hasRole('ADMIN')")
    @DeleteMapping("/users/{id}")
    public ResponseEntity<?> deleteUser(@PathVariable Long id) {
        userRepository.deleteById(id);
        return ResponseEntity.ok(new MessageResponse("User deleted successfully!"));
    }
}
```

## 11. Konfigurera JWT-inställningar

### application.properties:
```properties
# JWT Configuration
app.jwtSecret=mySecretKey
app.jwtExpirationMs=86400000

# Databas (samma som tidigare)
spring.datasource.url=jdbc:sqlite:users.db
spring.datasource.driver-class-name=org.sqlite.JDBC
spring.jpa.database-platform=org.hibernate.dialect.SQLiteDialect
spring.jpa.hibernate.ddl-auto=update
```

## 12. Testa JWT-implementationen

### Test 1: Logga in och få JWT token
```bash
curl -X POST http://localhost:8080/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"anna","password":"password123"}'

# Response:
{
  "token": "eyJhbGciOiJIUzUxMiJ9...",
  "type": "Bearer",
  "username": "anna",
  "roles": ["ROLE_USER"]
}
```

### Test 2: Använd token för att komma åt skyddad endpoint
```bash
curl -X GET http://localhost:8080/api/profile \
  -H "Authorization: Bearer eyJhbGciOiJIUzUxMiJ9..."

# Response:
{
  "username": "anna",
  "role": "USER"
}
```

### Test 3: Registrera ny användare
```bash
curl -X POST http://localhost:8080/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{"username":"bertil","password":"newpassword123"}'
```

## 13. Frontend-integration (JavaScript exempel)

### Login och spara token:
```javascript
async function login(username, password) {
    const response = await fetch('/api/auth/login', {
        method: 'POST',
        headers: {
            'Content-Type': 'application/json',
        },
        body: JSON.stringify({ username, password })
    });
    
    if (response.ok) {
        const data = await response.json();
        localStorage.setItem('jwt_token', data.token);
        return data;
    }
    throw new Error('Login failed');
}

async function fetchProtectedData() {
    const token = localStorage.getItem('jwt_token');
    
    const response = await fetch('/api/profile', {
        headers: {
            'Authorization': `Bearer ${token}`
        }
    });
    
    if (response.ok) {
        return await response.json();
    }
    throw new Error('Unauthorized');
}
```

## 14. JWT Refresh Tokens (avancerat)

För längre säkerhet kan du implementera refresh tokens:

```java
@Component
public class JwtUtils {
    
    // ... befintlig kod
    
    @Value("${app.jwtRefreshExpirationMs:604800000}")
    private Long refreshTokenDurationMs; // 7 dagar
    
    public String generateRefreshToken(Authentication authentication) {
        UserDetails userPrincipal = (UserDetails) authentication.getPrincipal();
        
        return Jwts.builder()
                .setSubject(userPrincipal.getUsername())
                .setIssuedAt(new Date())
                .setExpiration(new Date(System.currentTimeMillis() + refreshTokenDurationMs))
                .signWith(SignatureAlgorithm.HS512, jwtSecret)
                .compact();
    }
}
```

## Vanliga problem och lösningar

### Problem 1: "JWT token is expired"
**Orsak:** Token har gått ut  
**Lösning:** Implementera refresh tokens eller öka `jwtExpirationMs`

### Problem 2: "Invalid JWT signature"
**Orsak:** Fel secret key eller korrupt token  
**Lösning:** Kontrollera `app.jwtSecret` i properties

### Problem 3: CORS-fel från frontend
**Orsak:** Cross-origin requests blockeras  
**Lösning:** Konfigurera CORS i SecurityConfig:
```java
@Bean
public CorsConfigurationSource corsConfigurationSource() {
    CorsConfiguration configuration = new CorsConfiguration();
    configuration.setAllowedOrigins(Arrays.asList("http://localhost:3000"));
    configuration.setAllowedMethods(Arrays.asList("GET", "POST", "PUT", "DELETE"));
    configuration.setAllowedHeaders(Arrays.asList("*"));
    configuration.setAllowCredentials(true);
    
    UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
    source.registerCorsConfiguration("/**", configuration);
    return source;
}
```

### Problem 4: "Cannot set user authentication"
**Orsak:** JWT filter konfigurerat fel  
**Lösning:** Kontrollera att `AuthTokenFilter` är konfigurerad som `@Bean`

## Sammanfattning

Nu har du:
- ✅ Bytt från sessions till JWT-baserad authentication
- ✅ Implementerat stateless REST API:er
- ✅ Skapat JWT token generation och validation
- ✅ Byggt login/register endpoints
- ✅ Konfigurerat JWT security filter chain

## Nästa steg

I nästa guide kommer vi att:
- Implementera rollbaserad auktorisering i detalj
- Skapa dynamiska behörigheter från databasen
- Använda method-level security (@PreAuthorize)
- Bygga admin-gränssnitt för användarhantering

---

**Viktigt:** JWT är perfekt för REST API:er och mikroservices, men sessions kan fortfarande vara bättre för traditionella webbapplikationer med server-side rendering.
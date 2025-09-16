# Spring Security Logout Guide

## Introduktion

Spring Security hanterar logout på olika sätt beroende på autentiseringsmetod. Anledningen till att vi behöver explicit logout-hantering är att:

- **Säkerhet**: Förhindra obehörig åtkomst med stulna credentials
- **Sessionhantering**: Frigöra serverresurser (för sessions)
- **Användarupplevelse**: Tydlig indikation på att användaren är utloggad
- **Compliance**: Många säkerhetsstandarder kräver proper logout

### Behöver vi alltid en controller endpoint?

**För sessions**: Nej, Spring Security kan hantera logout automatiskt via konfiguration
**För JWT**: Ja, vi behöver nästan alltid en explicit endpoint för token-invalidering
**För renodlad backend (REST API)**: Ja, vi behöver endpoints som frontend kan anropa

## Session-baserad Logout

### Automatisk Konfiguration (utan controller)

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .logout(logout -> logout
                .logoutUrl("/logout")                    // URL som triggar logout
                .logoutSuccessUrl("/login?logout")       // Redirect efter logout
                .invalidateHttpSession(true)             // Invalidera HTTP-session
                .deleteCookies("JSESSIONID")            // Ta bort session-cookies
                .clearAuthentication(true)               // Rensa authentication context
                .permitAll()                            // Tillåt alla att accessa logout
            )
            .sessionManagement(session -> session
                .maximumSessions(1)                     // Begränsa samtidiga sessioner
                .maxSessionsPreventsLogin(false)        // Tillåt ny login, kicka gamla
            );
        return http.build();
    }
}
```

### Med Custom Controller (för REST API)

```java
@RestController
public class AuthController {
    
    @PostMapping("/api/logout")
    public ResponseEntity<?> logout(HttpServletRequest request, HttpServletResponse response) {
        // Hämta nuvarande authentication
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        
        if (auth != null) {
            // Spring Security's logout handler
            new SecurityContextLogoutHandler().logout(request, response, auth);
        }
        
        return ResponseEntity.ok(new MessageResponse("Logout successful"));
    }
}

// Response DTO
public class MessageResponse {
    private String message;
    
    public MessageResponse(String message) {
        this.message = message;
    }
    
    // getters/setters...
}
```

### Hur session-baserad logout fungerar

1. **Request kommer in** på `/logout` endpoint
2. **Spring Security interceptar** requesten automatiskt
3. **Session invalideras** - alla session-data raderas
4. **Cookies raderas** - JSESSIONID och andra konfigurerade cookies
5. **SecurityContext rensas** - authentication information tas bort
6. **Redirect/Response** - användaren omdirigeras eller får JSON-response

## JWT-baserad Logout

### JWT Token Service

```java
@Service
public class JwtTokenService {
    
    // I produktion: använd Redis eller databas för distributed systems
    private final Set<String> blacklistedTokens = new ConcurrentHashMap<>.newKeySet();
    
    /**
     * Lägg till token i blacklist för att invalidera den
     */
    public void blacklistToken(String token) {
        blacklistedTokens.add(token);
        // TODO: Implementera cleanup av gamla tokens för att undvika memory leak
    }
    
    /**
     * Kontrollera om token finns i blacklist
     */
    public boolean isTokenBlacklisted(String token) {
        return blacklistedTokens.contains(token);
    }
    
    /**
     * Extrahera JWT token från Authorization header
     */
    public String extractTokenFromHeader(String authHeader) {
        if (authHeader != null && authHeader.startsWith("Bearer ")) {
            return authHeader.substring(7); // Ta bort "Bearer " prefix
        }
        return null;
    }
    
    /**
     * Hämta expiration time från token (för cleanup)
     */
    public Date getExpirationFromToken(String token) {
        // Implementera JWT parsing för att få expiration
        // return Jwts.parser()...
        return null; // Placeholder
    }
}
```

### Logout Controller för JWT

```java
@RestController
@RequestMapping("/api/auth")
public class AuthController {
    
    @Autowired
    private JwtTokenService tokenService;
    
    /**
     * Logout endpoint som invaliderar JWT token
     */
    @PostMapping("/logout")
    public ResponseEntity<?> logout(HttpServletRequest request) {
        try {
            // Hämta token från Authorization header
            String authHeader = request.getHeader("Authorization");
            String token = tokenService.extractTokenFromHeader(authHeader);
            
            if (token != null) {
                // Lägg till token i blacklist
                tokenService.blacklistToken(token);
                
                // Rensa SecurityContext för denna request
                SecurityContextHolder.clearContext();
                
                return ResponseEntity.ok(new MessageResponse("Logout successful"));
            }
            
            // Ingen token hittades i request
            return ResponseEntity.badRequest()
                .body(new MessageResponse("No token found in request"));
                
        } catch (Exception e) {
            // Logga fel men avslöja inte känslig information
            return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
                .body(new MessageResponse("Logout failed"));
        }
    }
    
    /**
     * Optional: Logout från alla enheter (revoke all tokens för användare)
     */
    @PostMapping("/logout-all")
    public ResponseEntity<?> logoutAll(Authentication authentication) {
        // Implementera: invalidera alla tokens för denna användare
        // Kräver att tokens innehåller användar-ID och databas/cache lookup
        
        String username = authentication.getName();
        // tokenService.blacklistAllTokensForUser(username);
        
        return ResponseEntity.ok(new MessageResponse("Logged out from all devices"));
    }
}
```

### JWT Filter med Blacklist-kontroll

```java
@Component
public class JwtAuthenticationFilter extends OncePerRequestFilter {
    
    @Autowired
    private JwtTokenService tokenService;
    
    @Autowired
    private JwtUtil jwtUtil; // Din JWT utility class
    
    @Override
    protected void doFilterInternal(HttpServletRequest request, 
                                  HttpServletResponse response, 
                                  FilterChain filterChain) throws ServletException, IOException {
        
        try {
            // Hämta token från request
            String authHeader = request.getHeader("Authorization");
            String token = tokenService.extractTokenFromHeader(authHeader);
            
            if (token != null) {
                // VIKTIGT: Kontrollera blacklist INNAN token-validering
                if (tokenService.isTokenBlacklisted(token)) {
                    // Token är invaliderad, skicka 401 Unauthorized
                    response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
                    response.getWriter().write("Token has been invalidated");
                    return;
                }
                
                // Om token inte är blacklistad, fortsätt med normal validering
                if (jwtUtil.validateToken(token)) {
                    // Sätt authentication i SecurityContext
                    String username = jwtUtil.getUsernameFromToken(token);
                    // ... standard JWT authentication logic
                }
            }
            
        } catch (Exception e) {
            // Logga fel men fortsätt filter chain
            logger.error("JWT Authentication error: ", e);
        }
        
        // Fortsätt med nästa filter i kedjan
        filterChain.doFilter(request, response);
    }
    
    /**
     * Skippa JWT-kontroll för vissa endpoints (som login)
     */
    @Override
    protected boolean shouldNotFilter(HttpServletRequest request) {
        String path = request.getRequestURI();
        return path.equals("/api/auth/login") || path.equals("/api/auth/register");
    }
}
```

### Alternativ JWT Logout Strategi: Refresh Token Revocation

```java
@Service
public class RefreshTokenService {
    
    @Autowired
    private RefreshTokenRepository refreshTokenRepository;
    
    /**
     * Revoke refresh token vid logout
     * Access tokens expirerar naturligt (kortare livslängd)
     */
    public void revokeRefreshToken(String username) {
        // Hitta och ta bort refresh token från databas
        RefreshToken refreshToken = refreshTokenRepository.findByUsername(username);
        if (refreshToken != null) {
            refreshTokenRepository.delete(refreshToken);
        }
    }
    
    /**
     * Revoke alla refresh tokens för användare (logout från alla enheter)
     */
    public void revokeAllRefreshTokensForUser(String username) {
        List<RefreshToken> tokens = refreshTokenRepository.findAllByUsername(username);
        refreshTokenRepository.deleteAll(tokens);
    }
}

// I logout controller
@PostMapping("/logout")
public ResponseEntity<?> logoutWithRefreshToken(HttpServletRequest request, Authentication auth) {
    // Revoke refresh token istället för att blacklista access token
    refreshTokenService.revokeRefreshToken(auth.getName());
    
    // Access token kommer expirerar naturligt (sätt kort expiration tid)
    return ResponseEntity.ok(new MessageResponse("Logout successful"));
}
```

## Frontend Integration

### Session-baserad logout (traditionell webbapp)

```javascript
// Form-baserad logout (automatisk av Spring Security)
document.getElementById('logout-form').addEventListener('submit', function(e) {
    // Spring Security hanterar POST till /logout automatiskt
    // Ingen extra JavaScript behövs
});
```

### REST API logout (SPA/Mobile app)

```javascript
// Session-baserad REST logout
async function sessionLogout() {
    try {
        const response = await fetch('/api/logout', {
            method: 'POST',
            credentials: 'include',  // Inkludera cookies
            headers: {
                'Content-Type': 'application/json'
            }
        });
        
        if (response.ok) {
            // Omdirigera till login
            window.location.href = '/login';
        }
    } catch (error) {
        console.error('Logout failed:', error);
    }
}

// JWT-baserad logout
async function jwtLogout() {
    try {
        const token = localStorage.getItem('accessToken');
        
        const response = await fetch('/api/auth/logout', {
            method: 'POST',
            headers: {
                'Authorization': `Bearer ${token}`,
                'Content-Type': 'application/json'
            }
        });
        
        if (response.ok) {
            // Rensa lokala tokens
            localStorage.removeItem('accessToken');
            localStorage.removeItem('refreshToken');
            
            // Omdirigera till login
            window.location.href = '/login';
        }
    } catch (error) {
        // Även om logout misslyckas, rensa lokala tokens
        localStorage.removeItem('accessToken');
        localStorage.removeItem('refreshToken');
        window.location.href = '/login';
    }
}
```

## Jämförelse och Rekommendationer

| Aspekt | Sessions | JWT |
|--------|----------|-----|
| **Controller endpoint** | Optional (automatisk) | Krävs nästan alltid |
| **Serverstatus** | Kräver session-lagring | Stateless (utom blacklist) |
| **Logout-komplexitet** | Automatisk via konfiguration | Kräver explicit hantering |
| **Skalbarhet** | Begränsad av session-lagring | Hög skalbarhet |
| **Säkerhet** | Omedelbar invalidering | Kräver blacklist för omedelbar invalidering |

### För Renodlad Backend (REST API)

**Rekommendation**: Skapa alltid explicita logout endpoints även för sessions, eftersom:
- Frontend behöver programmatisk åtkomst
- Bättre kontroll över response format (JSON)
- Enhetlig API-design
- Möjlighet att logga logout-events

```java
// Minimal REST logout för sessions
@PostMapping("/api/logout") 
public ResponseEntity<?> logout(HttpServletRequest request, HttpServletResponse response) {
    Authentication auth = SecurityContextHolder.getContext().getAuthentication();
    if (auth != null) {
        new SecurityContextLogoutHandler().logout(request, response, auth);
    }
    return ResponseEntity.ok().build();
}
```

## Säkerhetsöverväganden

1. **CSRF Protection**: Aktivera för session-baserad auth
2. **HTTPS Only**: Alltid för produktion
3. **Token Cleanup**: Implementera för JWT blacklist
4. **Audit Logging**: Logga logout-events
5. **Rate Limiting**: Förhindra logout abuse
6. **Secure Headers**: Sätt proper cache-control headers

Denna guide visar att logout-hantering varierar kraftigt beroende på autentiseringsmetod, men med rätt implementation kan båda metoderna användas säkert i produktionssystem.
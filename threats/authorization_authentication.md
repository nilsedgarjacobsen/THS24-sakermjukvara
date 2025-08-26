# Authentication & Authorization Guide för Spring Boot

## Grunderna - Vad är Skillnaden?

Tänk på säkerhet som att komma in på en nattklubb:

**Authentication (Autentisering)** = "Vem är du?"
- Visar legitimation vid dörren
- Bevisar din identitet
- "Ja, du är verkligen John Doe"

**Authorization (Auktorisering)** = "Vad får du göra?"
- Kollar om du får gå in i VIP-rummet
- Kontrollerar dina rättigheter
- "Du är John, men du får inte vara här"

## Varför är Detta Grunden för Säkerhet?

Utan proper authentication och authorization:
- Vem som helst kan logga in som admin
- Användare kan komma åt andras data  
- Känslig information läcker ut
- System kan förstöras eller manipuleras

Det är som att ha ett hus utan lås och bara lita på att folk inte går in!

## Authentication - Att Bevisa Vem Du Är

### Vanliga Metoder

**1. Användarnamn & Lösenord**
- Mest vanliga metoden
- Du vet något endast du ska veta

**2. Multi-Factor Authentication (MFA)**
- Kombinerar flera metoder
- Något du vet + något du har
- T.ex. lösenord + SMS-kod

**3. Tokens (JWT, OAuth)**
- Modern webbutveckling
- Ingen session på servern
- Token innehåller användarinfo

### Spring Security Authentication

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            // Säg åt Spring att vi vill ha inloggning
            .formLogin(form -> form
                .loginPage("/login")           // Vår egen inloggningssida
                .defaultSuccessUrl("/home")    // Dit användaren ska efter inloggning
                .permitAll()                   // Alla får komma åt login-sidan
            )
            // Logga ut användare
            .logout(logout -> logout
                .logoutSuccessUrl("/")         // Dit de ska efter utloggning
                .permitAll()
            );
        
        return http.build();
    }
}
```

### Hantera Användare

```java
@Service
public class UserService implements UserDetailsService {
    
    @Autowired
    private UserRepository userRepo;
    
    @Override
    public UserDetails loadUserByUsername(String username) {
        // Hitta användaren i databasen
        User user = userRepo.findByUsername(username);
        
        if (user == null) {
            throw new UsernameNotFoundException("Användare hittades inte");
        }
        
        // Skapa Spring Security användare
        return User.builder()
            .username(user.getUsername())
            .password(user.getPassword())        // Måste vara hashad!
            .roles(user.getRoles())              // Användarens roller
            .build();
    }
}
```

## Authorization - Vad Får Du Göra?

Authorization händer EFTER authentication. Du har bevisat vem du är, nu kollar systemet vad du får göra.

### Roller vs Rättigheter

**Roller** = Samlingar av rättigheter
- ADMIN, USER, MODERATOR
- Enkelt att hantera

**Rättigheter** = Specifika tillstånd  
- READ_USERS, DELETE_POSTS, EDIT_PROFILE
- Mer flexibelt men komplexare

### Spring Security Authorization

```java
@Configuration
public class SecurityConfig {
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                // Publika sidor - alla får komma åt
                .requestMatchers("/", "/public/**", "/css/**").permitAll()
                
                // Admin-sidor - bara administratörer
                .requestMatchers("/admin/**").hasRole("ADMIN")
                
                // User-sidor - inloggade användare
                .requestMatchers("/user/**").hasRole("USER")
                
                // Allt annat kräver inloggning
                .anyRequest().authenticated()
            );
        
        return http.build();
    }
}
```

### Method-Level Security

```java
@RestController
public class UserController {
    
    // Bara admins kan hämta alla användare
    @PreAuthorize("hasRole('ADMIN')")
    @GetMapping("/admin/users")
    public List<User> getAllUsers() {
        return userService.findAll();
    }
    
    // Användare kan bara se sin egen profil
    @PreAuthorize("#username == authentication.name or hasRole('ADMIN')")
    @GetMapping("/user/{username}")
    public User getUser(@PathVariable String username) {
        return userService.findByUsername(username);
    }
    
    // Bara ägaren av inlägget kan redigera det
    @PreAuthorize("@postService.isOwner(#postId, authentication.name)")
    @PutMapping("/posts/{postId}")
    public Post updatePost(@PathVariable Long postId, @RequestBody Post post) {
        return postService.update(postId, post);
    }
}
```

## Säkra Lösenord

```java
@Configuration
public class SecurityConfig {
    
    @Bean
    public PasswordEncoder passwordEncoder() {
        // BCrypt är säker och standard
        return new BCryptPasswordEncoder();
    }
}

@Service
public class UserService {
    
    @Autowired
    private PasswordEncoder passwordEncoder;
    
    public User createUser(String username, String rawPassword) {
        User user = new User();
        user.setUsername(username);
        
        // Hasha lösenordet innan sparande - VIKTIGT!
        String hashedPassword = passwordEncoder.encode(rawPassword);
        user.setPassword(hashedPassword);
        
        return userRepo.save(user);
    }
    
    public boolean checkPassword(String rawPassword, String hashedPassword) {
        // BCrypt kollar automatiskt om lösenorden matchar
        return passwordEncoder.matches(rawPassword, hashedPassword);
    }
}
```

## JWT Tokens - Modern Authentication

```java
@Component
public class JwtService {
    
    private String secretKey = "mySecretKey"; // I verkligheten: från config
    
    public String generateToken(String username) {
        return Jwts.builder()
            .setSubject(username)                    // Vem token gäller för
            .setIssuedAt(new Date())                 // När den skapades
            .setExpiration(new Date(System.currentTimeMillis() + 86400000)) // 24h
            .signWith(SignatureAlgorithm.HS256, secretKey)  // Signera säkert
            .compact();
    }
    
    public String getUsernameFromToken(String token) {
        // Hämta användarnamn från token
        return Jwts.parser()
            .setSigningKey(secretKey)
            .parseClaimsJws(token)
            .getBody()
            .getSubject();
    }
    
    public boolean isTokenValid(String token) {
        try {
            // Om parsing fungerar är token giltig
            Jwts.parser().setSigningKey(secretKey).parseClaimsJws(token);
            return true;
        } catch (JwtException e) {
            return false;  // Token är trasig eller utgången
        }
    }
}
```

## Säkerhetslagen - Defense in Depth

Säkerhet fungerar bäst med flera lager:

### 1. Nätverkssäkerhet
- HTTPS överallt
- Brandväggar
- Ingen känslig info i URLs

### 2. Applikationssäkerhet  
- Authentication & Authorization
- Input-validering
- SQL injection-skydd

### 3. Datasäkerhet
- Hashade lösenord
- Krypterad data
- Säkra databaskonfigurationer

## Vanliga Säkerhetsmissar

### ❌ Dåliga Lösenordskrav
```java
// Dåligt - för svaga krav
if (password.length() >= 4) {
    // Detta är inte säkert!
}
```

### ✅ Bra Lösenordskrav  
```java
@Component
public class PasswordValidator {
    
    public boolean isValidPassword(String password) {
        return password.length() >= 8 &&           // Minst 8 tecken
               password.matches(".*[A-Z].*") &&    // Stor bokstav
               password.matches(".*[a-z].*") &&    // Liten bokstav  
               password.matches(".*[0-9].*") &&    // Siffra
               password.matches(".*[!@#$%^&*].*"); // Specialtecken
    }
}
```

### ❌ Session Fixation
```java
// Dåligt - använd aldrig samma session efter inloggning
// Spring Security fixar detta automatiskt, men bra att veta
```

### ✅ Säker Session-hantering
```java
@Configuration
public class SecurityConfig {
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .sessionManagement(session -> session
                .sessionCreationPolicy(SessionCreationPolicy.IF_REQUIRED)
                .maximumSessions(1)                    // Max 1 session per användare
                .maxSessionsPreventsLogin(false)       // Nya login kickar ut gamla
            );
        
        return http.build();
    }
}
```

## Best Practices Sammanfattning

### Authentication ✅
- Använd BCrypt för lösenord
- Implementera proper session-hantering  
- Använd HTTPS
- Lägg till MFA för viktiga konton
- Ha lockout efter failed attempts

### Authorization ✅
- Minsta möjliga rättigheter (Principle of Least Privilege)
- Kolla rättigheter på varje request
- Använd roller för enklare hantering
- Validera på både frontend OCH backend
- Logga säkerhetsrelevanta events

### Generellt ✅
- Defense in depth - flera säkerhetslager
- Håll bibliotek uppdaterade
- Testa säkerheten regelbundet
- Utbilda utvecklare om säkerhet
- Planera för incident response

## Slutsats

Authentication och authorization är fundamenten i din applikationssäkerhet. Utan dem har du ingen kontroll över vem som gör vad i ditt system.

**Kom ihåg:**
1. **Authentication först** - bevisa vem användaren är
2. **Authorization sedan** - kontrollera vad de får göra  
3. **Aldrig lita på klienten** - validera alltid på servern
4. **Flera säkerhetslager** - om ett misslyckas, finns andra
5. **Logga och övervaka** - upptäck problem tidigt

Med Spring Security får du mycket säkerhet "gratis", men du måste fortfarande förstå grunderna och konfigurera det rätt för din applikation!
# Spring Security - Databas-integration för användare

## Introduktion

I denna guide flyttar vi användardata från hårdkodade användare till databasen. Du kommer att lära dig:
- Skapa User-entitet och repository
- Koppla Spring Security till databasen
- Implementera grundläggande registrering
- Använda NoOpPasswordEncoder för utveckling

**Förutsättningar:** Ett Spring Boot-projekt med Spring Security och JPA redan konfigurerat.

## 1. Konfigurera databasen

### För SQLite (application.properties):
```properties
# SQLite-konfiguration
spring.datasource.url=jdbc:sqlite:users.db
spring.datasource.driver-class-name=org.sqlite.JDBC
spring.jpa.database-platform=org.hibernate.dialect.SQLiteDialect

# Skapa tabeller automatiskt
spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true
```

### För MySQL (application.properties):
```properties
# MySQL-konfiguration
spring.datasource.url=jdbc:mysql://localhost:3306/myapp
spring.datasource.username=root
spring.datasource.password=yourpassword
spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver

# Skapa tabeller automatiskt
spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true
```

**Förklaring:**
- `ddl-auto=update` betyder att Spring skapar/uppdaterar tabeller automatiskt
- `show-sql=true` visar SQL-queries i konsollen (bra för utveckling)

## 2. Skapa User-entiteten

```java
@Entity
@Table(name = "users")
public class User {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(unique = true, nullable = false)
    private String username;
    
    @Column(nullable = false)
    private String password;
    
    @Column(nullable = false)
    private String role;
    
    @Column(nullable = false)
    private boolean enabled = true;
    
    // Konstruktorer
    public User() {}
    
    public User(String username, String password, String role) {
        this.username = username;
        this.password = password;
        this.role = role;
        this.enabled = true;
    }
    
    // Getters och Setters
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    
    public String getUsername() { return username; }
    public void setUsername(String username) { this.username = username; }
    
    public String getPassword() { return password; }
    public void setPassword(String password) { this.password = password; }
    
    public String getRole() { return role; }
    public void setRole(String role) { this.role = role; }
    
    public boolean isEnabled() { return enabled; }
    public void setEnabled(boolean enabled) { this.enabled = enabled; }
}
```

**Viktiga detaljer:**
- `@Entity` gör klassen till en databastabell
- `@Table(name = "users")` anger tabellnamnet
- `@Column(unique = true)` betyder att username måste vara unikt
- `enabled` låter oss "stänga av" användare utan att radera dem

## 3. Skapa UserRepository

```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    
    Optional<User> findByUsername(String username);
    
    boolean existsByUsername(String username);
}
```

**Förklaring:**
- `JpaRepository` ger oss grundläggande CRUD-operationer gratis
- `findByUsername()` låter oss hitta användare via användarnamn
- `existsByUsername()` kollar om användarnamn redan finns

## 4. Skapa CustomUserDetailsService

Spring Security behöver veta hur det ska hämta användare från databasen:

```java
@Service
public class CustomUserDetailsService implements UserDetailsService {
    
    @Autowired
    private UserRepository userRepository;
    
    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        User user = userRepository.findByUsername(username)
            .orElseThrow(() -> new UsernameNotFoundException("Användare inte hittad: " + username));
        
        return org.springframework.security.core.userdetails.User.builder()
            .username(user.getUsername())
            .password(user.getPassword())
            .authorities("ROLE_" + user.getRole())
            .disabled(!user.isEnabled())
            .build();
    }
}
```

**Vad händer här:**
1. Spring Security anropar `loadUserByUsername()` när någon försöker logga in
2. Vi söker efter användaren i databasen
3. Om användaren finns, konverterar vi den till Spring Securitys `UserDetails`
4. `"ROLE_" + user.getRole()` lägger till "ROLE_"-prefix som Spring Security förväntar sig

## 5. Uppdatera SecurityConfig

Nu ska vi koppla vår databas-service till Spring Security. **Viktigt:** Vi använder `NoPasswordEncoder` tillfälligt för att få allt att fungera först:

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {
    
    @Autowired
    private CustomUserDetailsService userDetailsService;
    
    @Bean
    public PasswordEncoder passwordEncoder() {
        // ENDAST FÖR UTVECKLING - ersätts i nästa guide!
        return NoOpPasswordEncoder.getInstance();
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
            .userDetailsService(userDetailsService);  // Använd vår databas-service
        
        return http.build();
    }
}
```

**Viktiga ändringar:**
- `.userDetailsService(userDetailsService)` kopplar vår databas-service
- `NoOpPasswordEncoder.getInstance()` tillåter lösenord i klartext (ENDAST för utveckling!)
- Vi har lagt till `/signup` och `/register` som publika endpoints
- Tog bort den gamla `InMemoryUserDetailsManager`

**Varför NoOpPasswordEncoder?** 
- Låter oss testa databas-integrationen utan att krångla med lösenordshashning än
- Gör det enkelt att komma igång och se att allting fungerar
- Vi byter till säker BCrypt i nästa guide

## 6. Lägg till testdata

För att testa systemet, låt oss skapa några användare när applikationen startar:

```java
@Component
public class DataLoader implements ApplicationRunner {
    
    @Autowired
    private UserRepository userRepository;
    
    @Override
    public void run(ApplicationArguments args) throws Exception {
        // Skapa testanvändare endast om inga användare finns
        if (userRepository.count() == 0) {
            User user1 = new User("anna", "password123", "USER");
            User user2 = new User("admin", "admin123", "ADMIN");
            
            userRepository.save(user1);
            userRepository.save(user2);
            
            System.out.println("Testanvändare skapade:");
            System.out.println("- anna / password123 (USER)");
            System.out.println("- admin / admin123 (ADMIN)");
        }
    }
}
```

**Viktigt:** Detta är bara för testning! Lösenord sparas i klartext vilket är FARLIGT i produktion. Nästa guide fixar detta säkerhetsproblem.

## Varför vi börjar med NoOpPasswordEncoder

`NoOpPasswordEncoder` är Spring Securitys sätt att säga "använd lösenord i klartext". Vi använder det tillfälligt eftersom:

- **Enklare att komma igång** - Ingen risk för encoding-problem
- **Lättare att debugga** - Du ser lösenorden direkt i databasen  
- **Fokus på databas-integrationen** - Vi lär oss en sak i taget
- **Byggd för detta** - Spring tillhandahåller den just för utvecklingsfasen

**Nästa guide** kommer vi byta till `BCryptPasswordEncoder` för riktig säkerhet.

## 7. Skapa en enkel registreringscontroller

```java
@Controller
public class AuthController {
    
    @Autowired
    private UserRepository userRepository;
    
    @GetMapping("/signup")
    public String showSignupForm() {
        return "signup";
    }
    
    @PostMapping("/register")
    public String registerUser(@RequestParam String username, 
                              @RequestParam String password,
                              Model model) {
        
        // Kolla om användarnamnet redan finns
        if (userRepository.existsByUsername(username)) {
            model.addAttribute("error", "Användarnamnet finns redan");
            return "signup";
        }
        
        // Skapa ny användare
        User newUser = new User(username, password, "USER");
        userRepository.save(newUser);
        
        return "redirect:/login?registered";
    }
    
    @GetMapping("/login")
    public String showLoginForm() {
        return "login";
    }
    
    @GetMapping("/dashboard")
    public String dashboard(Authentication auth, Model model) {
        model.addAttribute("username", auth.getName());
        return "dashboard";
    }
}
```

## 8. Testa din databas-integration

### Starta applikationen:
1. Spring kommer skapa tabellen `users` automatiskt
2. Testanvändare läggs till via `DataLoader`
3. Du ser SQL-queries i konsollen (om `show-sql=true`)

### Logga in:
- Gå till `http://localhost:8080/login`
- Testa: `anna` / `password123`
- Testa: `admin` / `admin123`

### Registrera ny användare:
- Gå till `http://localhost:8080/signup`
- Skapa ett nytt konto
- Logga in med det nya kontot

### Kolla databasen:
**För SQLite:** Filen `users.db` skapas i projektmappen
**För MySQL:** Logga in och kör `SELECT * FROM users;`

## Vanliga problem och lösningar

### Problem 1: "Table 'users' doesn't exist"
**Lösning:** Kontrollera att `spring.jpa.hibernate.ddl-auto=update` finns i konfigurationen

### Problem 2: SQLite-filen hittas inte
**Lösning:** Kontrollera sökvägen i `spring.datasource.url=jdbc:sqlite:users.db`

### Problem 3: "Access denied for user"
**Lösning:** Kontrollera MySQL-användarnamn och lösenord i `application.properties`

### Problem 4: Användarnamn fungerar inte
**Lösning:** Kontrollera att `username` är unikt i databasen

### Problem 5: "There is no PasswordEncoder mapped for the id null"
**Lösning:** Du har glömt att lägga till `PasswordEncoder`-beanen. Lägg till:
```java
@Bean
public PasswordEncoder passwordEncoder() {
    return NoOpPasswordEncoder.getInstance();
}
```

### Problem 6: Lösenord funkar inte efter databas-byte
**Orsak:** Du har bytt från hårdkodade användare till databas utan PasswordEncoder  
**Lösning:** Se Problem 5 ovan

## Sammanfattning

Nu har du:
- ✅ Kopplat Spring Security till en databas
- ✅ Skapat User-entitet och UserRepository
- ✅ Implementerat CustomUserDetailsService
- ✅ Lagt till grundläggande registrering
- ✅ Testat med både SQLite och MySQL

## Nästa steg

I nästa guide kommer vi att:
- Hasha lösenord säkert med BCrypt
- Förklara olika password encoders
- Uppdatera registrering och inloggning
- Hantera lösenordsvalidering

---

**Viktigt:** Lösenord sparas just nu i klartext! Detta är endast för testning. Nästa guide fixar detta säkerhetsproblem.
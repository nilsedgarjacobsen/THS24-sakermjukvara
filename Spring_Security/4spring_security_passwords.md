# Spring Security - Password encoding och hashning

## Introduktion

I denna guide kommer vi att ersätta den osäkra `NoOpPasswordEncoder` med säker lösenordshashning. Du kommer att lära dig:
- Varför lösenordshashning är kritiskt viktigt
- Skillnaden mellan olika password encoders
- Implementera BCrypt för säker lösenordshashning
- Uppdatera registrering och testdata
- Hantera lösenordsvalidering

**Förutsättningar:** Ett Spring Boot-projekt med Spring Security och databas-integration redan implementerat (med `NoOpPasswordEncoder`).

## 1. Varför behöver vi password encoding?

Just nu sparas lösenord i klartext i databasen:

```
Databas: users
| username | password  | role |
|----------|-----------|------|
| anna     | secret123 | USER |  ❌ FARLIGT!
| admin    | admin123  | ADMIN| ❌ FARLIGT!
```

**Problem med klartext-lösenord:**
- **Databasläckor** - Om någon får åtkomst till databasen ser de alla lösenord
- **Insiders** - Databasadministratörer kan se användarnas lösenord
- **Loggar** - Lösenord kan hamna i felloggar eller debug-utskrifter
- **Backups** - Osäkra backups exponerar alla lösenord

**Lösningen:** Hasha lösenorden innan de sparas:

```
Databas: users
| username | password                           | role |
|----------|------------------------------------|------|
| anna     | $2a$10$N.zmdr9k7uOCQb376NoUnu...      | USER |  ✅ SÄKERT!
| admin    | $2a$10$92IXUNpkjO0rOQ5byMi.Ye...      | ADMIN|  ✅ SÄKERT!
```

## 2. Vad är hashning?

**Hashning** är en envägsprocess som omvandlar lösenord till en oläslig sträng:

```
Input: "secret123"
Hash:  "$2a$10$N.zmdr9k7uOCQb376NoUnuTJ8iAt6Z5EHsM7rc.faVHnr6H55gNlW"
```

**Viktiga egenskaper:**
- **Envägs** - Omöjligt att få tillbaka originalet från hashen
- **Deterministisk** - Samma input ger alltid samma hash (med salt)
- **Känslig** - Liten ändring i input ger helt olika hash
- **Snabb att verifiera** - Kan snabbt jämföra input med lagrad hash

## 3. Olika Password Encoders

### NoOpPasswordEncoder (nuvarande)
```java
@Bean
public PasswordEncoder passwordEncoder() {
    return NoOpPasswordEncoder.getInstance(); // ENDAST FÖR UTVECKLING!
}
```
**Användning:** Utveckling/testning  
**Säkerhet:** Ingen - sparar lösenord i klartext

### BCryptPasswordEncoder (rekommenderat)
```java
@Bean
public PasswordEncoder passwordEncoder() {
    return new BCryptPasswordEncoder();
}
```
**Användning:** Produktion  
**Säkerhet:** Mycket hög - industry standard

### Argon2PasswordEncoder (modernast)
```java
@Bean
public PasswordEncoder passwordEncoder() {
    return new Argon2PasswordEncoder();
}
```
**Användning:** Nya projekt med högsta säkerhetskrav  
**Säkerhet:** Högst - vinnare av Password Hashing Competition

### StandardPasswordEncoder (föråldrad)
```java
// ❌ ANVÄND INTE - deprecated!
return new StandardPasswordEncoder();
```

## 4. Byta till BCryptPasswordEncoder

### Uppdatera SecurityConfig
```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {
    
    @Autowired
    private CustomUserDetailsService userDetailsService;
    
    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder(); // Byt från NoOpPasswordEncoder
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
            .userDetailsService(userDetailsService);
        
        return http.build();
    }
}
```

**Vad händer nu:**
- Spring Security använder BCrypt automatiskt för att jämföra lösenord
- Nya lösenord som sparas kommer att hashas
- **Viktigt:** Gamla lösenord i databasen kommer INTE att fungera!

## 5. Uppdatera registreringscontroller

Nu måste vi hasha lösenord innan vi sparar dem:

```java
@Controller
public class AuthController {
    
    @Autowired
    private UserRepository userRepository;
    
    @Autowired
    private PasswordEncoder passwordEncoder; // Injicera PasswordEncoder
    
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
        
        // Hasha lösenordet innan sparande
        String hashedPassword = passwordEncoder.encode(password);
        
        // Skapa ny användare med hashat lösenord
        User newUser = new User(username, hashedPassword, "USER");
        userRepository.save(newUser);
        
        return "redirect:/login?registered";
    }
    
    // Resten av metoderna...
}
```

**Viktiga ändringar:**
- `@Autowired PasswordEncoder` - Spring injicerar BCrypt automatiskt
- `passwordEncoder.encode(password)` - Hashar lösenordet före sparning
- Nu sparas hashade lösenord i databasen

## 6. Uppdatera testdata

Eftersom vi bytt till BCrypt måste vi även uppdatera testdatan:

```java
@Component
public class DataLoader implements ApplicationRunner {
    
    @Autowired
    private UserRepository userRepository;
    
    @Autowired
    private PasswordEncoder passwordEncoder; // Lägg till PasswordEncoder
    
    @Override
    public void run(ApplicationArguments args) throws Exception {
        // Skapa testanvändare endast om inga användare finns
        if (userRepository.count() == 0) {
            // Hasha lösenorden
            String annaPassword = passwordEncoder.encode("password123");
            String adminPassword = passwordEncoder.encode("admin123");
            
            User user1 = new User("anna", annaPassword, "USER");
            User user2 = new User("admin", adminPassword, "ADMIN");
            
            userRepository.save(user1);
            userRepository.save(user2);
            
            System.out.println("Testanvändare skapade:");
            System.out.println("- anna / password123 (USER)");
            System.out.println("- admin / admin123 (ADMIN)");
        }
    }
}
```

## 7. Testa den nya password encoding

### Starta applikationen:
1. **Rensa gammal data** - Ta bort gamla användare med klartext-lösenord
2. **Nya användare skapas** - Med BCrypt-hashade lösenord
3. **Kolla databasen** - Lösenord ser nu ut som `$2a$10$...`

### Testa inloggning:
- Logga in med `anna` / `password123`
- Logga in med `admin` / `admin123`
- Fungerar precis som tidigare, men lösenorden är nu säkra!

### Testa registrering:
- Gå till `/signup`
- Skapa ett nytt konto
- Kolla databasen - det nya lösenordet är hashat

## 8. Så här fungerar BCrypt bakom kulisserna

### Vid registrering:
```java
String rawPassword = "mypassword";
String hash = passwordEncoder.encode(rawPassword);
// hash = "$2a$10$N.zmdr9k7uOCQb..."
```

### Vid inloggning:
```java
String inputPassword = "mypassword";        // Vad användaren skrev
String storedHash = "$2a$10$N.zmdr9k7uOC..."; // Från databasen

boolean matches = passwordEncoder.matches(inputPassword, storedHash);
// matches = true om lösenordet stämmer
```

### BCrypt-hashens struktur:
```
$2a$10$N.zmdr9k7uOCQb376NoUnuTJ8iAt6Z5EHsM7rc.faVHnr6H55gNlW
│││ │  │                                   │
│││ │  └─ Salt (22 tecken)                 └─ Hash (31 tecken)
│││ └─ Cost factor (10 = 2^10 = 1024 rounds)
││└─ Minor version (a)
│└─ Major version (2)
└─ Identifier ($)
```

## 9. Lösenordsvalidering (frivilligt)

Du kan lägga till lösenordskrav för starkare säkerhet:

```java
@PostMapping("/register")
public String registerUser(@RequestParam String username, 
                          @RequestParam String password,
                          Model model) {
    
    // Validera lösenordsstyrka
    if (password.length() < 8) {
        model.addAttribute("error", "Lösenordet måste vara minst 8 tecken");
        return "signup";
    }
    
    if (!password.matches(".*[A-Z].*")) {
        model.addAttribute("error", "Lösenordet måste innehålla minst en stor bokstav");
        return "signup";
    }
    
    if (!password.matches(".*[0-9].*")) {
        model.addAttribute("error", "Lösenordet måste innehålla minst en siffra");
        return "signup";
    }
    
    // Fortsätt med registrering...
    if (userRepository.existsByUsername(username)) {
        model.addAttribute("error", "Användarnamnet finns redan");
        return "signup";
    }
    
    String hashedPassword = passwordEncoder.encode(password);
    User newUser = new User(username, hashedPassword, "USER");
    userRepository.save(newUser);
    
    return "redirect:/login?registered";
}
```

## 10. Anpassa BCrypt-inställningar (avancerat)

Du kan justera BCrypt-styrkan om behövs:

```java
@Bean
public PasswordEncoder passwordEncoder() {
    // Högre tal = säkrare men långsammare
    return new BCryptPasswordEncoder(12); // Standard är 10
}
```

**Kostnadsnivåer:**
- **4-8:** Snabb, lämplig för utveckling
- **10:** Standard, bra balans (Spring Securitys default)
- **12-15:** Långsam, extra säkerhet för känsliga system

## Vanliga problem och lösningar

### Problem 1: "Bad credentials" efter byte till BCrypt
**Orsak:** Gamla lösenord i databasen är i klartext, inte BCrypt-hashade  
**Lösning:** Rensa användartabellen och låt `DataLoader` skapa nya användare

### Problem 2: Inloggning tar för lång tid
**Orsak:** För hög cost factor i BCryptPasswordEncoder  
**Lösning:** Sänk från `new BCryptPasswordEncoder(15)` till `new BCryptPasswordEncoder(10)`

### Problem 3: "There is no PasswordEncoder mapped for the id null"
**Orsak:** Glömt att uppdatera `@Bean PasswordEncoder`  
**Lösning:** Kontrollera att du har `return new BCryptPasswordEncoder();`

### Problem 4: Nya användare kan inte logga in
**Orsak:** Glömt att hasha lösenord i registreringscontroller  
**Lösning:** Använd `passwordEncoder.encode(password)` före sparning

## Sammanfattning

Nu har du:
- ✅ Implementerat säker lösenordshashning med BCrypt
- ✅ Uppdaterat registrering för att hasha nya lösenord
- ✅ Uppdaterat testdata med hashade lösenord
- ✅ Förstått hur BCrypt fungerar bakom kulisserna
- ✅ Lagt till grundläggande lösenordsvalidering

## Nästa steg

I nästa guide kommer vi att:
- Konfigurera sessions i detalj
- Hantera session timeout
- Implementera "Remember Me"-funktionalitet
- Hantera samtidiga sessioner

---

**Viktigt:** Nu är dina lösenord säkra! Även om någon får åtkomst till databasen kan de inte se användarnas riktiga lösenord.
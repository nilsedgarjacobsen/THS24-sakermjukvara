# Spring Security - Första implementation

## Introduktion

Nu ska vi gå från teori till praktik! I denna guide kommer vi att:
1. Lägga till Spring Security till ett projekt
2. Förstå vad som händer automatiskt
3. Skapa vår första SecurityConfig-klass
4. Konfigurera Security Filter Chain steg för steg
5. Anpassa inloggningsprocessen

## 1. Lägga till Spring Security

### Maven (pom.xml)
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```

### Gradle (build.gradle)
```gradle
implementation 'org.springframework.boot:spring-boot-starter-security'
```

### Vad händer när du startar applikationen?

Så fort du lägger till dependency och startar om din app kommer du se något liknande i konsollen:

```
Using generated security password: a1b2c3d4-e5f6-7890-abcd-ef1234567890

This generated password is for development use only.
```

**Detta betyder:** Spring Security har automatiskt skapat en användare åt dig!
- **Användarnamn:** `user`
- **Lösenord:** `a1b2c3d4-e5f6-7890-abcd-ef1234567890` (ändras varje gång du startar om)

**Testa:** Gå till `http://localhost:8080` - du omdirigeras till en inloggningssida!

## 2. Vad Spring Security gör automatiskt

Bakom kulisserna konfigurerar Spring Security följande:
- **Alla endpoints** kräver autentisering
- **Inloggningsformulär** på `/login`
- **CSRF-skydd** är aktiverat
- **Säkra HTTP-headers** skickas automatiskt
- **Sessionhantering** är påslagen

## 3. Skapa din första SecurityConfig-klass

Nu ska vi ta kontroll över konfigurationen. Skapa denna klass:

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(authz -> authz
                .anyRequest().authenticated()
            )
            .formLogin();
        
        return http.build();
    }
}
```

**Vad händer nu:**
- Du har tagit kontroll över Security-konfigurationen
- Konfigurationen gör **exakt samma sak** som tidigare
- Inget har ändrats - du ser bara konfigurationen nu

## 4. Förstå Security Filter Chain

Security Filter Chain är **hjärtat** i Spring Security. Varje HTTP-request går igenom denna kedja:

```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http
        // Steg 1: Bestäm vilka requests som kräver autentisering
        .authorizeHttpRequests(authz -> authz
            .anyRequest().authenticated()  // "Alla requests kräver inloggning"
        )
        // Steg 2: Konfigurera hur inloggning ska fungera
        .formLogin();  // "Använd standard inloggningsformulär"
    
    return http.build();
}
```

### Visualisering av vad som händer

```
HTTP Request till /dashboard
      ↓
[CSRF-kontroll] ✓ 
      ↓
[Session-kontroll] ✓ 
      ↓
[Autentisering] ✓ Användare inloggad?
      ↓
[Auktorisering] ✓ Får komma åt /dashboard?
      ↓
Request når din @Controller
```

## 5. Konfigurera endpoints steg för steg

### Steg 1: Tillåt publika endpoints

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(authz -> authz
                .requestMatchers("/public/**").permitAll()
                .anyRequest().authenticated()
            )
            .formLogin();
        
        return http.build();
    }
}
```

**Vad detta gör:**
- `http://localhost:8080/public/info` → Ingen inloggning krävs
- `http://localhost:8080/dashboard` → Kräver inloggning

### Steg 2: Lägga till fler regler

```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http
        .authorizeHttpRequests(authz -> authz
            .requestMatchers("/public/**").permitAll()
            .requestMatchers("/css/**", "/js/**").permitAll()
            .requestMatchers("/api/**").authenticated()
            .anyRequest().authenticated()
        )
        .formLogin();
    
    return http.build();
}
```

**Ordning spelar roll!** Spring Security kollar reglerna uppifrån och ner. Första matchande regel vinner.

### Steg 3: Använda HTTP-metoder

```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http
        .authorizeHttpRequests(authz -> authz
            .requestMatchers("/public/**").permitAll()
            .requestMatchers(HttpMethod.GET, "/api/products").permitAll()
            .requestMatchers(HttpMethod.POST, "/api/products").authenticated()
            .anyRequest().authenticated()
        )
        .formLogin();
    
    return http.build();
}
```

**Vad detta gör:**
- `GET /api/products` → Alla kan läsa produkter
- `POST /api/products` → Bara inloggade kan skapa produkter

## 6. Anpassa inloggningsprocessen

### Grundläggande anpassning

```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http
        .authorizeHttpRequests(authz -> authz
            .requestMatchers("/public/**").permitAll()
            .anyRequest().authenticated()
        )
        .formLogin(form -> form
            .loginPage("/signin")
            .defaultSuccessUrl("/dashboard")
            .failureUrl("/signin?error")
            .permitAll()
        );
    
    return http.build();
}
```

**Vad detta gör:**
- Inloggningssida: `/signin` istället för `/login`  
- Efter lyckad inloggning → `/dashboard`
- Efter misslyckad inloggning → `/signin?error`

### Konfigurera utloggning

```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http
        .authorizeHttpRequests(authz -> authz
            .requestMatchers("/public/**").permitAll()
            .anyRequest().authenticated()
        )
        .formLogin(form -> form
            .loginPage("/signin")
            .defaultSuccessUrl("/dashboard")
            .permitAll()
        )
        .logout(logout -> logout
            .logoutUrl("/logout")
            .logoutSuccessUrl("/public/goodbye")
            .permitAll()
        );
    
    return http.build();
}
```

### Komplett exempel

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(authz -> authz
                .requestMatchers("/public/**", "/css/**", "/js/**").permitAll()
                .requestMatchers("/signin").permitAll()
                .requestMatchers("/api/**").authenticated()
                .anyRequest().authenticated()
            )
            .formLogin(form -> form
                .loginPage("/signin")
                .defaultSuccessUrl("/dashboard")
                .failureUrl("/signin?error")
                .permitAll()
            )
            .logout(logout -> logout
                .logoutUrl("/logout")
                .logoutSuccessUrl("/public/goodbye")
                .permitAll()
            );
        
        return http.build();
    }
}
```

## Testa din konfiguration

1. **Starta applikationen**
2. **Testa publika endpoints:** `http://localhost:8080/public/info` (ingen inloggning)
3. **Testa skyddade endpoints:** `http://localhost:8080/dashboard` (omdirigeras till `/signin`)
4. **Logga in** med `user` och lösenordet från konsollen
5. **Testa utloggning:** Gå till `http://localhost:8080/logout`

## Vanliga fel och lösningar

### Problem 1: "There was an unexpected error (type=Forbidden, status=403)"
**Orsak:** CSRF-token saknas för POST-requests  
**Lösning:** Lägg till CSRF-token i formulär eller stäng av CSRF för utveckling:
```java
.csrf(csrf -> csrf.disable())  // Endast för utveckling!
```

### Problem 2: "Circular view path [signin]"
**Orsak:** Din controller returnerar "signin" från URL `/signin`  
**Lösning:** Returnera "redirect:/signin" eller använd annan URL

### Problem 3: Redirectar i evighet
**Orsak:** Din anpassade login-sida kräver autentisering  
**Lösning:** Glöm inte `.permitAll()` efter `.loginPage()`:
```java
.formLogin(form -> form
    .loginPage("/signin")
    .permitAll()  // Viktigt!
)
```

### Problem 4: CSS/JS fungerar inte
**Orsak:** Statiska filer kräver autentisering  
**Lösning:**
```java
.requestMatchers("/css/**", "/js/**", "/images/**").permitAll()
```

## Sammanfattning

Du har nu lärt dig att:
- ✅ Lägga till Spring Security till ett projekt
- ✅ Skapa en SecurityConfig-klass som visar standardkonfigurationen
- ✅ Konfigurera Security Filter Chain steg för steg
- ✅ Skapa regler för publika och skyddade endpoints
- ✅ Anpassa inloggnings- och utloggningsprocessen

## Nästa steg

I nästa guide kommer vi att:
- Integrera med databas för användarhantering
- Implementera rollbaserad auktorisering
- Skapa säker lösenordshantering
- Bygga anpassade inloggningsformulär

---

**Kom ihåg:** Använd alltid `user` och lösenordet från konsollen för att testa denna konfiguration!
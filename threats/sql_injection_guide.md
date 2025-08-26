# SQL Injection Guide för Spring Boot med JPA

## Vad är SQL Injection?

SQL Injection är när en angripare kan manipulera dina databasfrågor genom att skicka in skadlig kod via formulär eller URL-parametrar. Det är som att låta en främling skriva delar av din SQL-kod.

### Enkelt Exempel

Tänk dig att du har en inloggning som kollar:
```sql
SELECT * FROM users WHERE username = 'admin' AND password = 'hemligt123'
```

Om koden bygger denna fråga genom att bara sätta ihop strängar, kan en angripare skicka in:
```
Användarnamn: admin'; DROP TABLE users; --
```

Då blir frågan:
```sql
SELECT * FROM users WHERE username = 'admin'; DROP TABLE users; --' AND password = 'hemligt123'
```

Resultat: Din användartabell försvinner! 😱

## Varför Händer Detta?

Problemet uppstår när din kod inte kan skilja mellan **data** (användarinput) och **kod** (SQL-kommandon). Databasen ser allting som kommandon att köra.

## JPA och Säkerhet

Den goda nyheten: JPA skyddar dig automatiskt i de flesta fall! Men du måste veta när skyddet fungerar och när det inte gör det.

### ✅ Automatiskt Säkert med JPA

```java
// Spring Data JPA query methods - alltid säkra
User findByUsername(String username);
List<User> findByEmailContaining(String email);
User findByUsernameAndPassword(String username, String password);
```

Spring genererar säker SQL automatiskt. Du kan skicka in vad som helst i `username` - ingen risk!

### ✅ Säkert med @Query och Parametrar

```java
@Query("SELECT u FROM User u WHERE u.username = :username")
User findUser(@Param("username") String username);

// Även med LIKE
@Query("SELECT u FROM User u WHERE u.name LIKE %:searchTerm%")
List<User> searchUsers(@Param("searchTerm") String searchTerm);
```

Symbolen `:username` säger åt JPA: "Detta är data, inte kod". Perfekt säkert!

### ❌ FARLIGT - När JPA Inte Skyddar

```java
// String concatenation = DÅLIGT
@Query("SELECT u FROM User u WHERE u.username = '" + "#{username}" + "'")
User farligMetod(@Param("username") String username);

// Native SQL utan parametrar = DÅLIGT  
@Query(value = "SELECT * FROM users WHERE name = '" + "#{searchTerm}" + "'", 
       nativeQuery = true)
List<User> änMerFarligt(@Param("searchTerm") String searchTerm);
```

## Praktiska Riktlinjer

### Använd Query Methods Först
```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    
    // Perfekt för enkla sökningar
    User findByUsername(String username);
    List<User> findByAgeGreaterThan(Integer age);
    List<User> findByEmailContainingIgnoreCase(String email);
    
    // Kombinationer fungerar också
    User findByUsernameAndActiveTrue(String username);
}
```

### För Komplexare Frågor - @Query
```java
@Repository  
public interface UserRepository extends JpaRepository<User, Long> {
    
    @Query("SELECT u FROM User u WHERE u.createdDate BETWEEN :start AND :end")
    List<User> findUsersInDateRange(@Param("start") LocalDate start, 
                                   @Param("end") LocalDate end);
    
    @Query("SELECT u FROM User u WHERE u.department.name = :deptName AND u.active = true")
    List<User> findActiveUsersByDepartment(@Param("deptName") String deptName);
}
```

### Native SQL - Var Extra Försiktig
```java
// OK - med parametrar
@Query(value = "SELECT * FROM users WHERE LOWER(email) = LOWER(:email)", 
       nativeQuery = true)
User findByEmailIgnoreCase(@Param("email") String email);
```

## Vanliga Misstag att Undvika

### 1. String Building i Services
```java
// ❌ Gör ALDRIG så här
@Service
public class UserService {
    
    public List<User> sökAnvändare(String sökterm) {
        // FARLIGT - bygg inte SQL-strängar manuellt
        String jpql = "SELECT u FROM User u WHERE u.name = '" + sökterm + "'";
        // Detta kan exploateras!
    }
}
```

### 2. Dynamiska Queries Fel
```java  
// ❌ Farligt sätt
public List<User> dynamiskSökning(String namn, String email) {
    String jpql = "SELECT u FROM User u WHERE 1=1";
    if (namn != null) {
        jpql += " AND u.name = '" + namn + "'";  // DÅLIGT
    }
    // ...
}
```

## Säker Dynamisk Sökning

Använd `Specification` för komplexa, dynamiska queries:

```java
@Service
public class UserSearchService {
    
    public List<User> sökAnvändare(String namn, String email, Integer minÅlder) {
        
        Specification<User> spec = Specification.where(null);
        
        if (namn != null) {
            spec = spec.and((root, query, cb) -> 
                cb.like(cb.lower(root.get("name")), "%" + namn.toLowerCase() + "%"));
        }
        
        if (email != null) {
            spec = spec.and((root, query, cb) -> 
                cb.equal(root.get("email"), email));
        }
        
        if (minÅlder != null) {
            spec = spec.and((root, query, cb) -> 
                cb.greaterThanOrEqualTo(root.get("age"), minÅlder));
        }
        
        return userRepository.findAll(spec);
    }
}
```

## Input-Validering - Extra Skydd

```java
@RestController
public class UserController {
    
    @GetMapping("/users/search")
    public List<User> sökAnvändare(
            @RequestParam @Size(min = 1, max = 50) String namn,
            @RequestParam @Email String email) {
        
        // Validering händer automatiskt med @Valid/@Size/@Email
        return userService.sökAnvändare(namn, email);
    }
}
```

## Sammanfattning - Enkla Regler

### ✅ Gör Detta:
1. **Använd Spring Data query methods** - alltid säkra
2. **Använd @Query med :parametrar** - aldrig string concatenation  
3. **Validera input** med Spring Validation
4. **Använd Specification** för komplexa dynamiska queries

### ❌ Undvik Detta:
1. **String concatenation** i JPQL eller native SQL
2. **Att bygga queries manuellt** i service-lager
3. **Native SQL utan parametrar**
4. **Att lita blindt på frontend-validering**

## Slutsats

Med JPA är du redan skyddad i 90% av fallen! Huvudregeln är enkel: **Använd alltid parametrar (`:namn`) istället för string concatenation (`"+ input +"`).**

Spring Data query methods och @Query med parametrar är dina bästa vänner. De är både säkra och enkla att använda.

Kommer du ihåg denna enkla regel är du trygg: **Data ska vara parametrar, inte del av SQL-koden!**
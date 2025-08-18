# Spring Security - Roller och behörigheter

## Introduktion

I denna guide kommer vi att implementera ett komplett system för roller och behörigheter. Du kommer att lära dig:
- Skillnaden mellan roller och behörigheter (authorities)
- Skapa dynamiska roller från databasen
- Implementera method-level security
- Bygga ett admin-gränssnitt för användarhantering
- Hantera komplexa behörighetshierarkier

**Förutsättningar:** Ett Spring Boot-projekt med Spring Security, JWT-authentication och databas-integration redan implementerat.

## 1. Roller vs Behörigheter - Vad är skillnaden?

### Roller (Roles)
**Roller** är högnivå-kategorier som beskriver vad en användare är:
- `USER` - Vanlig användare
- `ADMIN` - Administratör  
- `MANAGER` - Chef
- `MODERATOR` - Moderator

### Behörigheter (Authorities/Permissions)
**Behörigheter** är specifika åtgärder som en användare kan utföra:
- `READ_USERS` - Kan läsa användardata
- `WRITE_USERS` - Kan skapa/uppdatera användare
- `DELETE_USERS` - Kan radera användare
- `MANAGE_POSTS` - Kan hantera inlägg

### Praktiskt exempel:
```
ADMIN-roll har behörigheter:
- READ_USERS
- WRITE_USERS  
- DELETE_USERS
- MANAGE_POSTS

USER-roll har behörigheter:
- READ_USERS (endast sin egen data)
```

## 2. Skapa Role och Permission-entiteter

### Role-entitet:
```java
@Entity
@Table(name = "roles")
public class Role {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(unique = true, nullable = false)
    private String name;
    
    @Column
    private String description;
    
    @ManyToMany(fetch = FetchType.EAGER)
    @JoinTable(
        name = "role_permissions",
        joinColumns = @JoinColumn(name = "role_id"),
        inverseJoinColumns = @JoinColumn(name = "permission_id")
    )
    private Set<Permission> permissions = new HashSet<>();
    
    // Konstruktorer, getters och setters
    public Role() {}
    
    public Role(String name, String description) {
        this.name = name;
        this.description = description;
    }
    
    // Getters och setters
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    
    public String getDescription() { return description; }
    public void setDescription(String description) { this.description = description; }
    
    public Set<Permission> getPermissions() { return permissions; }
    public void setPermissions(Set<Permission> permissions) { this.permissions = permissions; }
}
```

### Permission-entitet:
```java
@Entity
@Table(name = "permissions")
public class Permission {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(unique = true, nullable = false)
    private String name;
    
    @Column
    private String description;
    
    // Konstruktorer, getters och setters
    public Permission() {}
    
    public Permission(String name, String description) {
        this.name = name;
        this.description = description;
    }
    
    // Getters och setters
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    
    public String getDescription() { return description; }
    public void setDescription(String description) { this.description = description; }
}
```

## 3. Uppdatera User-entiteten

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
    private boolean enabled = true;
    
    @ManyToMany(fetch = FetchType.EAGER)
    @JoinTable(
        name = "user_roles",
        joinColumns = @JoinColumn(name = "user_id"),
        inverseJoinColumns = @JoinColumn(name = "role_id")
    )
    private Set<Role> roles = new HashSet<>();
    
    // Konstruktorer
    public User() {}
    
    public User(String username, String password) {
        this.username = username;
        this.password = password;
        this.enabled = true;
    }
    
    // Getters och setters
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    
    public String getUsername() { return username; }
    public void setUsername(String username) { this.username = username; }
    
    public String getPassword() { return password; }
    public void setPassword(String password) { this.password = password; }
    
    public boolean isEnabled() { return enabled; }
    public void setEnabled(boolean enabled) { this.enabled = enabled; }
    
    public Set<Role> getRoles() { return roles; }
    public void setRoles(Set<Role> roles) { this.roles = roles; }
    
    // Hjälpmetoder
    public void addRole(Role role) {
        this.roles.add(role);
    }
    
    public void removeRole(Role role) {
        this.roles.remove(role);
    }
}
```

## 4. Skapa repositories

### RoleRepository:
```java
@Repository
public interface RoleRepository extends JpaRepository<Role, Long> {
    Optional<Role> findByName(String name);
    boolean existsByName(String name);
}
```

### PermissionRepository:
```java
@Repository
public interface PermissionRepository extends JpaRepository<Permission, Long> {
    Optional<Permission> findByName(String name);
    boolean existsByName(String name);
}
```

## 5. Uppdatera CustomUserDetailsService

```java
@Service
public class CustomUserDetailsService implements UserDetailsService {
    
    @Autowired
    private UserRepository userRepository;
    
    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        User user = userRepository.findByUsername(username)
            .orElseThrow(() -> new UsernameNotFoundException("Användare inte hittad: " + username));
        
        return buildUserDetails(user);
    }
    
    private UserDetails buildUserDetails(User user) {
        Set<GrantedAuthority> authorities = new HashSet<>();
        
        // Lägg till roller
        for (Role role : user.getRoles()) {
            authorities.add(new SimpleGrantedAuthority("ROLE_" + role.getName()));
            
            // Lägg till behörigheter från rollerna
            for (Permission permission : role.getPermissions()) {
                authorities.add(new SimpleGrantedAuthority(permission.getName()));
            }
        }
        
        return org.springframework.security.core.userdetails.User.builder()
            .username(user.getUsername())
            .password(user.getPassword())
            .authorities(authorities)
            .disabled(!user.isEnabled())
            .build();
    }
}
```

**Vad händer här:**
- Varje `Role` läggs till som `ROLE_ROLENAME` (Spring Security-konvention)
- Varje `Permission` från rollerna läggs till som authority
- Användaren får både roller och behörigheter som authorities

## 6. Uppdatera JWT-implementationen

### Uppdatera JwtUtils för att inkludera roller:
```java
@Component
public class JwtUtils {
    
    // ... befintlig kod
    
    public String generateJwtToken(Authentication authentication) {
        UserDetails userPrincipal = (UserDetails) authentication.getPrincipal();
        
        // Samla roller och behörigheter
        Set<String> authorities = userPrincipal.getAuthorities().stream()
            .map(GrantedAuthority::getAuthority)
            .collect(Collectors.toSet());
        
        return Jwts.builder()
                .setSubject(userPrincipal.getUsername())
                .claim("authorities", authorities)
                .setIssuedAt(new Date())
                .setExpiration(new Date(System.currentTimeMillis() + jwtExpirationMs))
                .signWith(SignatureAlgorithm.HS512, jwtSecret)
                .compact();
    }
    
    public Collection<? extends GrantedAuthority> getAuthoritiesFromJwtToken(String token) {
        Claims claims = Jwts.parser()
                .setSigningKey(jwtSecret)
                .parseClaimsJws(token)
                .getBody();
        
        @SuppressWarnings("unchecked")
        List<String> authorities = (List<String>) claims.get("authorities");
        
        return authorities.stream()
                .map(SimpleGrantedAuthority::new)
                .collect(Collectors.toList());
    }
}
```

## 7. Aktivera Method-Level Security

### SecurityConfig:
```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity(prePostEnabled = true) // Aktivera @PreAuthorize
public class SecurityConfig {
    
    // ... befintlig kod som tidigare
}
```

## 8. Använda behörigheter i controllers

### Rollbaserad säkerhet:
```java
@RestController
@RequestMapping("/api/admin")
public class AdminController {
    
    @Autowired
    private UserRepository userRepository;
    
    @Autowired
    private RoleRepository roleRepository;
    
    @GetMapping("/users")
    @PreAuthorize("hasRole('ADMIN')")
    public ResponseEntity<List<User>> getAllUsers() {
        List<User> users = userRepository.findAll();
        return ResponseEntity.ok(users);
    }
    
    @PostMapping("/users")
    @PreAuthorize("hasAuthority('WRITE_USERS')")
    public ResponseEntity<?> createUser(@RequestBody CreateUserRequest request) {
        // Skapa användare
        User user = new User(request.getUsername(), passwordEncoder.encode(request.getPassword()));
        
        // Lägg till roll
        Role userRole = roleRepository.findByName("USER")
            .orElseThrow(() -> new RuntimeException("Role not found"));
        user.addRole(userRole);
        
        userRepository.save(user);
        return ResponseEntity.ok(new MessageResponse("User created successfully"));
    }
    
    @DeleteMapping("/users/{id}")
    @PreAuthorize("hasAuthority('DELETE_USERS')")
    public ResponseEntity<?> deleteUser(@PathVariable Long id) {
        userRepository.deleteById(id);
        return ResponseEntity.ok(new MessageResponse("User deleted successfully"));
    }
}
```

### Användarspecifik säkerhet:
```java
@RestController
@RequestMapping("/api/users")
public class UserController {
    
    @Autowired
    private UserRepository userRepository;
    
    @GetMapping("/{id}")
    @PreAuthorize("hasRole('ADMIN') or (hasRole('USER') and #id == authentication.principal.id)")
    public ResponseEntity<User> getUser(@PathVariable Long id, Authentication authentication) {
        User user = userRepository.findById(id)
            .orElseThrow(() -> new RuntimeException("User not found"));
        return ResponseEntity.ok(user);
    }
    
    @PutMapping("/{id}")
    @PreAuthorize("hasAuthority('WRITE_USERS') or #id == authentication.principal.id")
    public ResponseEntity<?> updateUser(@PathVariable Long id, 
                                       @RequestBody UpdateUserRequest request,
                                       Authentication authentication) {
        // Uppdatera användare
        User user = userRepository.findById(id)
            .orElseThrow(() -> new RuntimeException("User not found"));
        
        user.setUsername(request.getUsername());
        userRepository.save(user);
        
        return ResponseEntity.ok(new MessageResponse("User updated successfully"));
    }
}
```

## 9. Skapa testdata med roller och behörigheter

```java
@Component
public class RoleDataLoader implements ApplicationRunner {
    
    @Autowired
    private UserRepository userRepository;
    
    @Autowired
    private RoleRepository roleRepository;
    
    @Autowired
    private PermissionRepository permissionRepository;
    
    @Autowired
    private PasswordEncoder passwordEncoder;
    
    @Override
    public void run(ApplicationArguments args) throws Exception {
        if (permissionRepository.count() == 0) {
            createPermissions();
            createRoles();
            createUsers();
        }
    }
    
    private void createPermissions() {
        Permission readUsers = new Permission("READ_USERS", "Can read user data");
        Permission writeUsers = new Permission("WRITE_USERS", "Can create and update users");
        Permission deleteUsers = new Permission("DELETE_USERS", "Can delete users");
        Permission managePosts = new Permission("MANAGE_POSTS", "Can manage posts");
        
        permissionRepository.saveAll(Arrays.asList(readUsers, writeUsers, deleteUsers, managePosts));
        System.out.println("Permissions created");
    }
    
    private void createRoles() {
        // Hämta behörigheter
        Permission readUsers = permissionRepository.findByName("READ_USERS").orElseThrow();
        Permission writeUsers = permissionRepository.findByName("WRITE_USERS").orElseThrow();
        Permission deleteUsers = permissionRepository.findByName("DELETE_USERS").orElseThrow();
        Permission managePosts = permissionRepository.findByName("MANAGE_POSTS").orElseThrow();
        
        // Skapa USER-roll
        Role userRole = new Role("USER", "Standard user");
        userRole.setPermissions(Set.of(readUsers));
        
        // Skapa MODERATOR-roll
        Role moderatorRole = new Role("MODERATOR", "Content moderator");
        moderatorRole.setPermissions(Set.of(readUsers, managePosts));
        
        // Skapa ADMIN-roll
        Role adminRole = new Role("ADMIN", "System administrator");
        adminRole.setPermissions(Set.of(readUsers, writeUsers, deleteUsers, managePosts));
        
        roleRepository.saveAll(Arrays.asList(userRole, moderatorRole, adminRole));
        System.out.println("Roles created");
    }
    
    private void createUsers() {
        Role userRole = roleRepository.findByName("USER").orElseThrow();
        Role moderatorRole = roleRepository.findByName("MODERATOR").orElseThrow();
        Role adminRole = roleRepository.findByName("ADMIN").orElseThrow();
        
        // Skapa vanlig användare
        User user = new User("anna", passwordEncoder.encode("password123"));
        user.addRole(userRole);
        
        // Skapa moderator
        User moderator = new User("moderator", passwordEncoder.encode("mod123"));
        moderator.addRole(moderatorRole);
        
        // Skapa admin
        User admin = new User("admin", passwordEncoder.encode("admin123"));
        admin.addRole(adminRole);
        
        userRepository.saveAll(Arrays.asList(user, moderator, admin));
        System.out.println("Users created with roles");
    }
}
```

## 10. Rollhantering via REST API

### RoleController:
```java
@RestController
@RequestMapping("/api/admin/roles")
@PreAuthorize("hasRole('ADMIN')")
public class RoleController {
    
    @Autowired
    private RoleRepository roleRepository;
    
    @Autowired
    private PermissionRepository permissionRepository;
    
    @Autowired
    private UserRepository userRepository;
    
    @GetMapping
    public ResponseEntity<List<Role>> getAllRoles() {
        List<Role> roles = roleRepository.findAll();
        return ResponseEntity.ok(roles);
    }
    
    @PostMapping
    public ResponseEntity<?> createRole(@RequestBody CreateRoleRequest request) {
        if (roleRepository.existsByName(request.getName())) {
            return ResponseEntity.badRequest()
                .body(new MessageResponse("Role already exists"));
        }
        
        Role role = new Role(request.getName(), request.getDescription());
        
        // Lägg till behörigheter
        if (request.getPermissionIds() != null) {
            Set<Permission> permissions = new HashSet<>();
            for (Long permissionId : request.getPermissionIds()) {
                Permission permission = permissionRepository.findById(permissionId)
                    .orElseThrow(() -> new RuntimeException("Permission not found"));
                permissions.add(permission);
            }
            role.setPermissions(permissions);
        }
        
        roleRepository.save(role);
        return ResponseEntity.ok(new MessageResponse("Role created successfully"));
    }
    
    @PutMapping("/{id}")
    public ResponseEntity<?> updateRole(@PathVariable Long id, @RequestBody UpdateRoleRequest request) {
        Role role = roleRepository.findById(id)
            .orElseThrow(() -> new RuntimeException("Role not found"));
        
        role.setName(request.getName());
        role.setDescription(request.getDescription());
        
        // Uppdatera behörigheter
        if (request.getPermissionIds() != null) {
            Set<Permission> permissions = new HashSet<>();
            for (Long permissionId : request.getPermissionIds()) {
                Permission permission = permissionRepository.findById(permissionId)
                    .orElseThrow(() -> new RuntimeException("Permission not found"));
                permissions.add(permission);
            }
            role.setPermissions(permissions);
        }
        
        roleRepository.save(role);
        return ResponseEntity.ok(new MessageResponse("Role updated successfully"));
    }
    
    @PostMapping("/{roleId}/users/{userId}")
    public ResponseEntity<?> assignRoleToUser(@PathVariable Long roleId, @PathVariable Long userId) {
        Role role = roleRepository.findById(roleId)
            .orElseThrow(() -> new RuntimeException("Role not found"));
        User user = userRepository.findById(userId)
            .orElseThrow(() -> new RuntimeException("User not found"));
        
        user.addRole(role);
        userRepository.save(user);
        
        return ResponseEntity.ok(new MessageResponse("Role assigned to user"));
    }
}
```

## 11. Komplexa behörighetskontroller

### Hierarkiska roller:
```java
@Service
public class AuthorityService {
    
    @Autowired
    private RoleRepository roleRepository;
    
    public boolean hasPermission(Authentication authentication, String permission) {
        return authentication.getAuthorities().stream()
            .anyMatch(authority -> authority.getAuthority().equals(permission));
    }
    
    public boolean canAccessUser(Authentication authentication, Long userId) {
        // Admin kan komma åt alla användare
        if (hasRole(authentication, "ADMIN")) {
            return true;
        }
        
        // Användare kan bara komma åt sin egen data
        if (hasRole(authentication, "USER")) {
            String username = authentication.getName();
            // Hämta användarens ID från databasen och jämför
            // Implementation beror på din UserDetails-implementation
            return true; // Förenklad logik
        }
        
        return false;
    }
    
    private boolean hasRole(Authentication authentication, String role) {
        return authentication.getAuthorities().stream()
            .anyMatch(authority -> authority.getAuthority().equals("ROLE_" + role));
    }
}
```

### Använda custom authority service:
```java
@RestController
@RequestMapping("/api/users")
public class UserController {
    
    @Autowired
    private AuthorityService authorityService;
    
    @GetMapping("/{id}")
    @PreAuthorize("@authorityService.canAccessUser(authentication, #id)")
    public ResponseEntity<User> getUser(@PathVariable Long id) {
        // Implementation
        return ResponseEntity.ok(user);
    }
}
```

## 12. Frontend-integration

### JavaScript för rollbaserat UI:
```javascript
class AuthService {
    constructor() {
        this.token = localStorage.getItem('jwt_token');
        this.userRoles = this.getRolesFromToken();
    }
    
    getRolesFromToken() {
        if (!this.token) return [];
        
        try {
            const payload = JSON.parse(atob(this.token.split('.')[1]));
            return payload.authorities || [];
        } catch (e) {
            return [];
        }
    }
    
    hasRole(role) {
        return this.userRoles.includes('ROLE_' + role);
    }
    
    hasPermission(permission) {
        return this.userRoles.includes(permission);
    }
    
    canDeleteUsers() {
        return this.hasPermission('DELETE_USERS');
    }
    
    isAdmin() {
        return this.hasRole('ADMIN');
    }
}

// Användning
const auth = new AuthService();

if (auth.isAdmin()) {
    document.getElementById('admin-menu').style.display = 'block';
}

if (auth.canDeleteUsers()) {
    document.getElementById('delete-button').style.display = 'block';
}
```

## 13. Testa rollsystemet

### Test 1: Logga in med olika roller
```bash
# Logga in som admin
curl -X POST http://localhost:8080/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"admin123"}'

# Använd token för admin-åtgärder
curl -X GET http://localhost:8080/api/admin/users \
  -H "Authorization: Bearer [admin-token]"
```

### Test 2: Testa behörigheter
```bash
# Försök radera användare (kräver DELETE_USERS)
curl -X DELETE http://localhost:8080/api/admin/users/1 \
  -H "Authorization: Bearer [token]"
```

### Test 3: Method-level security
```bash
# Vanlig användare försöker komma åt admin-endpoint
curl -X GET http://localhost:8080/api/admin/users \
  -H "Authorization: Bearer [user-token]"
# Ska ge 403 Forbidden
```

## Vanliga problem och lösningar

### Problem 1: "Access Denied" trots rätt roll
**Orsak:** Glömt `ROLE_`-prefix eller felstavad roll  
**Lösning:** Kontrollera att roller sparas som `ROLE_ADMIN` i authorities

### Problem 2: Behörigheter fungerar inte i JWT
**Orsak:** Authorities inkluderas inte i JWT-token  
**Lösning:** Uppdatera `generateJwtToken()` för att inkludera authorities

### Problem 3: @PreAuthorize fungerar inte
**Orsak:** Glömt aktivera method security  
**Lösning:** Lägg till `@EnableMethodSecurity(prePostEnabled = true)`

### Problem 4: Cirkulära referenser i JSON
**Orsak:** User → Role → Permission → Role  
**Lösning:** Använd `@JsonIgnore` eller DTO-klasser för API-responses

## Sammanfattning

Nu har du:
- ✅ Implementerat komplett rollsystem med databas
- ✅ Skapat dynamiska behörigheter och roller
- ✅ Använt method-level security (@PreAuthorize)
- ✅ Byggt admin-API för rollhantering
- ✅ Integrerat roller med JWT-authentication
- ✅ Skapat hierarkiska behörighetskontroller

## Fortsatt utveckling

För att vidareutveckla systemet kan du:
- Implementera rollhierarkier (ADMIN > MODERATOR > USER)
- Skapa resursberoende behörigheter (äga en post för att redigera den)
- Bygga ett webb-UI för rollhantering
- Lägga till audit logging för säkerhetsändringar
- Implementera temporära roller med utgångsdatum

---

**Gratulationer!** Du har nu ett komplett, produktionsklart säkerhetssystem med Spring Security!
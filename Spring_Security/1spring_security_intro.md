# Vad är Spring Security?

## Introduktion

Spring Security är som en säkerhetsvakt för din Spring Boot-applikation. När du lägger till Spring Security till ditt projekt får du plötsligt en massa säkerhetsfunktioner automatiskt - utan att du behöver skriva en rad kod!

Men vad händer egentligen bakom kulisserna? Och vad kan du använda Spring Security till? Det ska vi utforska i denna guide.

## Vad är Spring Security?

Spring Security är ett ramverk som hanterar **autentisering** och **auktorisering** i Spring-applikationer.

### Enkelt förklarat:
- **Autentisering** = "Vem är du?" (inloggning)
- **Auktorisering** = "Vad får du göra?" (behörigheter)

Spring Security fungerar som en **säkerhetsvakt på en nattklubb**:
1. Kollar ditt ID vid entrén (autentisering)
2. Bestämmer vilka områden du får komma in i (auktorisering)
3. Följer med och kollar varje gång du försöker gå någonstans (varje HTTP-request)

## Vad händer när du lägger till Spring Security?

Så fort du lägger till Spring Security som dependency händer **magiska saker automatiskt**:

### 1. All trafik blockeras som standard
```
Innan Security: Alla kan komma åt allt
Efter Security: Ingen kan komma åt något (utan inloggning)
```

Det är som att sätta en vakt vid alla dörrar i byggnaden.

### 2. En automatisk inloggningssida skapas
Spring Security skapar en enkel inloggningssida åt dig på `/login`. Du behöver inte skriva HTML eller CSS - den bara finns där!

### 3. Ett standardanvändarkonto skapas
```
Användarnamn: user
Lösenord: [slumpmässigt genererat - visas i konsollen]
```

### 4. Alla endpoints skyddas
Varje URL i din applikation kräver nu inloggning. Även dina REST API:er!

### 5. Sessionshantering aktiveras
Spring Security skapar automatiskt **sessioner** för att komma ihåg att användare är inloggade.

**Så här fungerar det:**
```
1. Användaren loggar in framgångsrikt
2. Spring Security skapar en session på servern
3. Ett session-ID skickas till webbläsaren som en cookie
4. Vid nästa request skickar webbläsaren session-ID:t
5. Spring Security hittar sessionen och "kommer ihåg" användaren
```

**Enkelt förklarat:** Det är som att få en nummerlapp på en restaurang - servern "kommer ihåg" din beställning genom numret, och du visar lappen varje gång du hämtar något.

**Varför behövs sessions?** HTTP är "glömsk" - varje request är som en ny konversation. Sessions gör så att servern kan komma ihåg tidigare requests.

### 6. CSRF-skydd aktiveras
Skydd mot Cross-Site Request Forgery är påslaget från början.

### 7. Säkra HTTP-headers läggs till
Headers som skyddar mot XSS och andra attacker skickas automatiskt med varje svar.

## Vad du får "gratis" med Spring Security

### Autentisering
- Form-baserad inloggning, HTTP Basic, JWT, OAuth 2.0, LDAP

### Lösenordshantering  
- Automatisk säker hashning (BCrypt, Argon2)
- Lösenordspolicys och reset-funktioner

### Sessionshantering
- Automatiska sessioner med timeout
- Skydd mot session fixation
- Begränsning av samtidiga inloggningar

### Säkerhetsheaders
- XSS-skydd, clickjacking-skydd, CSP

### Integrationsmöjligheter
- Databaser, externa providers, method-level security

## Vad kommer vi använda Spring Security till?

I vårt Spring Boot-projekt kommer vi att använda Spring Security för:

### 1. Grundläggande autentisering
- Skapa användarkonton
- Hantera inloggning och utloggning
- Spara användardata i databas

### 2. Rollbaserad auktorisering
```
Roller vi kommer arbeta med:
- USER: Grundläggande användare
- ADMIN: Administratörer
- MODERATOR: Mellannivå
```

### 3. Endpoint-säkerhet
- Vissa endpoints endast för inloggade användare
- Admin-endpoints endast för administratörer
- Publika endpoints som alla kan komma åt

### 4. REST API-säkerhet
- Skydda våra API:er
- Använda JWT-tokens för API-åtkomst
- Hantera API-nycklar och rate limiting

### 5. Formulärsäkerhet
- Säkra inloggningsformulär
- CSRF-skydd för alla formulär
- Input-validering och säkerhetsheaders

### 6. Sessionhantering
- Kontrollera session-längd
- Hantera samtidiga sessioner
- Säker utloggning

## Spring Securitys arkitektur (förenklad)

## Spring Securitys arkitektur (förenklad)

Spring Security fungerar genom **filters** som kollar varje HTTP-request:

```
HTTP Request → CSRF-kontroll → Autentisering → Auktorisering → Din controller
```

Tänk på det som säkerhetskontroller på en flygplats - varje kontroll måste passeras.

**Security Context** håller koll på vem som är inloggad i varje request.

## Vanliga komponenter du kommer stöta på

## Vanliga komponenter

**UserDetailsService** - Hämtar användarinformation  
**PasswordEncoder** - Hashar och verifierar lösenord  
**AuthenticationProvider** - Kontrollerar inloggningsuppgifter  
**AccessDecisionManager** - Bestämmer behörigheter

## Konfiguration vs Konvention

### Konvention (Convention)
Spring Security kommer med smarta standardinställningar:
- Alla endpoints kräver inloggning
- Formulärbaserad inloggning aktiverad
- CSRF-skydd påslaget
- Säkra headers skickas automatiskt

### Konfiguration (Configuration)
Du kan anpassa allt genom konfiguration:
- Vilka endpoints som ska vara publika
- Vilken typ av autentisering som ska användas
- Hur lösenord ska hashas
- Vilka roller som finns

## Fördelar med Spring Security

## Fördelar med Spring Security

- **Säkerhet som standard** - Mycket säkerhet utan att tänka
- **Flexibilitet** - Anpassas från enkla till komplexa system  
- **Vältestad** - Miljontals applikationer, alla vanliga problem lösta
- **Integration** - Fungerar perfekt med Spring-ekosystemet

## Vanliga missförstånd

### ❌ "Spring Security är komplext"
**Sanningen:** Grundfunktionerna är enkla att komma igång med. Komplexiteten kommer när du behöver avancerade funktioner.

### ❌ "Jag kan bygga säkerhet själv"
**Sanningen:** Säkerhet är extremt svårt att få rätt. Spring Security har löst problem du inte ens vet att du har.

### ❌ "Spring Security gör appen långsam"
**Sanningen:** Den säkerhetsöverhead som finns är försumbar jämfört med fördelarna.

### ❌ "Jag behöver inte säkerhet i utvecklingsfasen"
**Sanningen:** Det är mycket lättare att bygga in säkerhet från början än att lägga till det senare.

## Nästa steg

Nu när du förstår vad Spring Security är och vad det gör automatiskt, är du redo att börja konfigurera det för dina specifika behov.

I nästa guide kommer vi att:
1. Lägga till Spring Security till ett projekt
2. Konfigurera grundläggande inställningar
3. Skapa våra första användare
4. Skydda specifika endpoints

## Sammanfattning

Spring Security är som att anlita en professionell säkerhetsfirma för din webbapplikation:

- **Du får massor av säkerhet automatiskt** när du lägger till det
- **Det skyddar alla dina endpoints** från början
- **Du kan anpassa det** efter dina behov
- **Det hanterar komplexa säkerhetsproblem** som du annars skulle behöva lösa själv

**Kom ihåg:** Spring Security är inte något som "läggs till" på slutet - det är en grundläggande del av hur din applikation fungerar från dag ett.

---

**Reflektion:** Tänk på webbapplikationer du använder dagligen. Vilka säkerhetsfunktioner märker du som användare? Hur tror du Spring Security implementerar dessa bakom kulisserna?
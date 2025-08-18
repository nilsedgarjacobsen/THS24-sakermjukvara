# Vanliga säkerhetshot och försvar

## Introduktion

Nu när du förstår vad som gör webbsäkerhet unikt, låt oss titta på de vanligaste säkerhetshoten som webbutvecklare måste hantera. 

Dessa hot kommer från OWASP Top 10 - en lista över de allvarligaste säkerhetriskerna för webbapplikationer, uppdaterad av säkerhetsexperter världen över.

## 1. Broken Access Control (Trasig åtkomstkontroll)

### Vad är det?
När användare kan komma åt saker de inte ska ha tillgång till.

### Så här ser det ut:
```
Vanlig användare Anna loggar in
↓
Anna ändrar URL:en från /profile/123 till /profile/124  
↓
Anna ser nu Bertils privata profil
```

### Andra exempel:
- Komma åt admin-panelen utan att vara admin
- Se andras beställningar genom att ändra order-ID
- Ladda ner filer genom att gissa filnamn

### Så skyddar vi oss:
- **Kontrollera på servern** - Aldrig bara förlita sig på att användaren "inte kan" komma åt något
- **Principen om minsta behörighet** - Ge bara åtkomst till det som verkligen behövs
- **Auktorisering på varje request** - Kontrollera behörighet för varje åtgärd
- **Loggning** - Logga åtkomstförsök för att upptäcka missbruk

## 2. Cryptographic Failures (Kryptografiska fel)

### Vad är det?
När känslig data inte skyddas ordentligt med kryptering, eller när kryptering används fel.

### Så här ser det ut:
```
Lösenord sparas i klartext i databasen
↓
Någon får åtkomst till databasen
↓
Alla lösenord är synliga direkt
```

### Andra exempel:
- Skicka känslig data över HTTP istället för HTTPS
- Använda svag kryptering som lätt kan knäckas
- Spara kreditkortsnummer okrypterat

### Så skyddar vi oss:
- **HTTPS överallt** - All kommunikation måste vara krypterad
- **Hasha lösenord** - Aldrig spara lösenord i klartext
- **Stark kryptering** - Använd moderna, säkra algoritmer
- **Kryptera känslig data** - I databasen och vid överföring

## 3. Injection (Injektion)

### Vad är det?
När skadlig kod "injiceras" i din applikation genom användarinput.

### SQL Injection exempel:
```
Användaren skriver i sökrutan: "'; DROP TABLE users; --"
↓
Din SQL-fråga blir: SELECT * FROM products WHERE name = ''; DROP TABLE users; --'
↓
Hela användartabellen raderas
```

### Andra injektionstyper:
- **NoSQL injection** - Samma problem för NoSQL-databaser
- **Command injection** - Köra systemkommandon
- **LDAP injection** - Attackera LDAP-queries

### Så skyddar vi oss:
- **Prepared statements** - Separera SQL-kod från data
- **Input-validering** - Kontrollera all användarinput
- **Principle of least privilege** - Databas-användaren ska ha minimala rättigheter
- **Escaping** - Behandla användardata som data, inte kod

## 4. Insecure Design (Osäker design)

### Vad är det?
När säkerhetsbrister är inbyggda i applikationens grundläggande design.

### Så här ser det ut:
```
Ett system för lösenordsåterställning:
1. Användaren anger e-post
2. Systemet skickar det gamla lösenordet i klartext
3. Ingen verifiering att personen äger e-posten
```

### Andra exempel:
- Inte implementera rate limiting (begränsa antal försök)
- Sakna backup- och återhämtningsplaner
- Inte överväga säkerhet vid systemdesign

### Så skyddar vi oss:
- **Threat modeling** - Tänk på vad som kan gå fel redan i designfasen
- **Säkerhetsarkitektur** - Bygga in säkerhet från början
- **Peer review** - Låta andra granska designen
- **Säkerhetskrav** - Definiera säkerhetsmål tidigt

## 5. Security Misconfiguration (Säkerhetsmiskonfiguration)

### Vad är det?
När säkerhetsinställningar är felaktiga, inkompletta eller använder standardinställningar.

### Så här ser det ut:
```
En webbserver med standardinställningar:
- Admin-lösenord: "admin123"
- Felsidor visar stacktraces
- Onödiga tjänster körs
- Ingen brandvägg konfigurerad
```

### Andra exempel:
- Glömma att stänga av utvecklingsläge i produktion
- Använda standard-lösenord
- Lämna testdata i produktionsdatabasen

### Så skyddar vi oss:
- **Säkra standardinställningar** - Ändra alla standard-lösenord
- **Minimala installationer** - Installera bara vad som behövs
- **Regelbundna säkerhetsuppdateringar** - Hålla system uppdaterade
- **Separata miljöer** - Utveckling, test och produktion ska vara åtskilda

## 6. Vulnerable Components (Sårbara komponenter)

### Vad är det?
När du använder bibliotek, ramverk eller komponenter som har kända säkerhetshål.

### Så här ser det ut:
```
Du använder ett JavaScript-bibliotek från 2018
↓
En säkerhetsbrist upptäcks i biblioteket 2020
↓
Du uppdaterar aldrig biblioteket
↓
Din app är sårbar för denna attack
```

### Andra exempel:
- Gamla versioner av Spring Boot
- JavaScript-bibliotek med säkerhetsbrister
- Operativsystem som inte uppdaterats

### Så skyddar vi oss:
- **Inventering** - Håll koll på alla bibliotek och versioner
- **Regelbundna uppdateringar** - Uppdatera komponenter ofta
- **Sårbarhetsscanning** - Automatiska verktyg som hittar kända problem
- **Källkontroll** - Endast använda bibliotek från pålitliga källor

## 7. Identification and Authentication Failures (Identifierings- och autentiseringsfel)

### Vad är det?
När systemet inte kan bevisa att användaren är den de påstår sig vara.

### Så här ser det ut:
```
Ett lösenordsystem som tillåter:
- Lösenord: "123456"
- Ingen begränsning på antal inloggningsförsök
- Svaga frågor för lösenordsåterställning: "Vad heter din hund?"
```

### Andra exempel:
- Sessioner som aldrig går ut
- Lösenord som skickas via e-post
- Brute force-attacker inte förhindras

### Så skyddar vi oss:
- **Starka lösenordskrav** - Längd och komplexitet
- **Multi-factor authentication** - Mer än bara lösenord
- **Rate limiting** - Begränsa inloggningsförsök
- **Säkra sessioner** - Timeout och säker hantering

## 8. Software and Data Integrity Failures (Programvaru- och dataintegritetfel)

### Vad är det?
När din kod eller data kan ändras av obehöriga utan att du märker det.

### Så här ser det ut:
```
Din app laddar JavaScript från en CDN
↓
CDN:en blir hackad
↓
Skadlig kod injiceras i ditt JavaScript
↓
Alla dina användare påverkas
```

### Andra exempel:
- Auto-updates utan signaturverifiering
- Användning av osäkra pip/npm-packages
- Serialisering av opålitlig data

### Så skyddar vi oss:
- **Subresource Integrity** - Verifiera att externa resurser inte ändrats
- **Signerade updates** - Alla uppdateringar ska vara kryptografiskt signerade
- **Input-validering** - Även för serialiserad data
- **Säkra CI/CD-pipelines** - Build-processen ska vara säker

## 9. Security Logging and Monitoring Failures (Brister i säkerhetsloggning och övervakning)

### Vad är det?
När du inte märker att säkerhetsincidenter pågår eller har hänt.

### Så här ser det ut:
```
En angripare testar olika lösenord i 2 timmar
↓
Får tillgång till ett konto
↓
Laddar ner känslig data i 3 dagar
↓
Du märker ingenting förrän en kund klagar
```

### Andra exempel:
- Inga loggar över inloggningsförsök
- Varningar skickas inte till rätt personer
- Loggar sparas inte tillräckligt länge

### Så skyddar vi oss:
- **Omfattande loggning** - Logga säkerhetsrelevanta händelser
- **Realtidsövervakning** - Automatiska varningar vid misstänkt aktivitet
- **Logganalys** - Regelbunden granskning av loggar
- **Incidentrespons** - Plan för vad som ska hända vid säkerhetsbrott

## 10. Server-Side Request Forgery (SSRF)

### Vad är det?
När din server kan luras att göra requests åt en angripare.

### Så här ser det ut:
```
En funktion som hämtar bilder från URL:er:
Användaren anger: "http://internal-admin-panel/delete-all-users"
↓
Servern gör en request till intern admin-panel
↓
Admin-funktioner körs utan behörighet
```

### Andra exempel:
- Läsa lokala filer på servern
- Attackera interna system
- Port scanning av interna nätverk

### Så skyddar vi oss:
- **URL-validering** - Kontrollera vilka URL:er som tillåts
- **Whitelist** - Endast tillåta specifika domäner
- **Nätverkssegmentering** - Begränsa vad servern kan komma åt
- **Input sanitization** - Rensa användarinput innan användning

## Gemensamma försvarsstrategier

### Defense in Depth (Försvar på djupet)
Förlita dig aldrig på bara ett säkerhetslager. Kombinera flera:
- Input-validering OCH output-encoding
- Autentisering OCH auktorisering
- HTTPS OCH säkra cookies

### Principle of Least Privilege (Princip om minsta behörighet)
Ge bara den åtkomst som verkligen behövs:
- Användare får bara komma åt sina egna data
- Databas-användare får bara de rättigheter som behövs
- Applikationen får bara komma åt nödvändiga resurser

### Fail Securely (Misslyckanden ska vara säkra)
När något går fel, var säker snarare än funktionell:
- Om auktorisering är osäker → neka åtkomst
- Om input är tvetydig → avvisa istället för att gissa
- Om systemet är överbelastat → begränsa istället för att krascha

## Nästa steg

Nu när du känner till de vanligaste hoten är du redo att lära dig hur man implementerar konkreta säkerhetslösningar med Spring Security och andra verktyg.

**Kom ihåg:** Säkerhet handlar inte om att vara perfekt från dag ett, utan om att medvetet arbeta med dessa risker och kontinuerligt förbättra skyddet.

---

**Reflektion:** Vilka av dessa hot känner du igen från webbsidor du använt? Har du märkt när webbsidor implementerat skydd mot dessa problem?
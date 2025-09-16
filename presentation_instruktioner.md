# Spring Boot Presentation - Instruktioner (10 minuter)

##  Vi bokar in pass för presentation på tisdagen, onsdagen och torsdagen. Alla pass är på distans. 

## 1. Demo av fungerande applikation (4 minuter)

### Välj ett av:
- **Webbläsare**: Navigera genom din app, visa att endpoints fungerar
- **Postman**: Visa GET/POST requests till dina REST endpoints

### Visa grundfunktionalitet:
- 2-3 huvudfunktioner som fungerar
- Lyckade API-anrop/sidor som laddar

---

## 2. Security Demo (3 minuter)

### Visa att säkerheten fungerar:
- **Webbläsare**: Försök nå skyddad sida → login required
- **Postman**: Visa 401/403 utan token → lyckad request med token/auth
- Visa olika roller/behörigheter om du har det

---

## 3. Kodgenomgång - snabbt (2 minuter)

### Välj 1-2 saker att visa:
- SecurityConfig (authentication/authorization setup)
- En Controller med säkerhetsannotationer (`@PreAuthorize` etc.)
- Eller den koddel som löser ditt specifika krav

---

## 4. Särskilt krav/VG-moment (1 minut)

### Välj ett att lyfta fram:
- Specifikt funktionskrav du löst elegant
- VG-krav (JWT tokens, custom authentication, etc.)
- Något tekniskt utmanande du löst

---

## Tips för presentationen:

- **Förbered båda demo-sätt** (webbläsare + Postman)
- **Välj ditt starkaste krav** att highlighta
- **Håll kodvisningen kort** - bara det viktigaste
- **Ha tydliga exempel** på vad som funkar/inte funkar med säkerhet
- **Ha backup-screenshots** om något krånglar
- **Testa allt innan presentation**

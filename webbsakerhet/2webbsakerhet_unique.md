# Vad gör webbsäkerhet unikt?

## Varför är webbsäkerhet annorlunda?

Du kanske undrar: "Säkerhet är väl säkerhet? Varför behöver jag lära mig specifikt om *webb*säkerhet?"

Svaret är att webben skapar unika utmaningar som inte finns i andra typer av mjukvaruutveckling. Låt oss utforska varför.

## Webben är designad för delning

### Webbens ursprungliga syfte
Webben skapades för att **dela information fritt**. Den byggdes med filosofin:
- All information ska vara tillgänglig
- Länkar ska kunna peka vart som helst
- Innehåll ska kunna ses av alla

### Dagens utmaning
Nu ska vi använda samma platform för att **skydda information**:
- Banksystem
- Privata meddelanden  
- Företagshemligheter
- Personlig data

Detta är som att använda en megafon (webben) för att viska hemligheter.

## Grundläggande skillnader från annan mjukvara

### Desktop-applikationer vs Webbapplikationer

**Desktop-app:**
```
Du som utvecklare:
- Vet vilken dator appen körs på
- Vet vilket operativsystem
- Kontrollerar hela miljön
- Apps installeras lokalt
```

**Webb-app:**
```
Du som utvecklare:
- Vet inte vilken enhet som kommer åt appen
- Vet inte vilken webbläsare
- Kan inte kontrollera användarens miljö  
- Appen körs på internet (tillgänglig för alla)
```

### Förtroendemodellen

**Desktop:**
```
Användaren installerar din app
↓
Implicit förtroende: "Jag litar på denna app"
↓
Appen kan göra nästan vad som helst på datorn
```

**Webb:**
```
Användaren besöker din webbsida
↓
Inget förtroende: "Jag vet inte vem som gjort detta"
↓
Webbläsaren begränsar vad appen får göra
```

## Webbens unika arkitektur

### HTTP är "glömsk"

**Vanlig applikation:**
```
Start programmet
↓
Använd programmet (kommer ihåg vad du gör)
↓
Stäng programmet
```

**Webbapplikation:**
```
Skicka request → Få svar → Glöm allt
Skicka request → Få svar → Glöm allt  
Skicka request → Få svar → Glöm allt
```

Varje klick på en webbsida är som att ringa ett företag där personen som svarar aldrig kommer ihåg att ni pratat tidigare.

### Klienten kan inte litas på

**Desktop:**
```
Du skickar ut en .exe-fil
- Användaren kan inte ändra din kod
- Det som körs är det du skrev
```

**Webb:**
```
Du skickar HTML, CSS och JavaScript
- Användaren kan ändra allt i webbläsaren
- Kan se all kod (högerklicka → "Visa källa")
- Kan stänga av JavaScript
- Kan ändra formulärdata innan det skickas
```

**Viktig princip:** Allt som händer i webbläsaren kan manipuleras av användaren.

## Webbens komplexa miljö

### Flera parter involverda

En webbsida innehåller ofta kod från många källor:
```
Din webbsida kan innehålla:
- Din egen kod (backend)
- JavaScript från CDN (jquery, bootstrap)
- Fonts från Google
- Analytics från Google/Adobe
- Reklam från andra företag
- Sociala medier-widgets (Facebook, Twitter)
```

**Säkerhetsfrågan:** Litar du på alla dessa parter?

### Cross-origin komplexitet

**Desktop:** All kod kommer från samma källa (din app)
**Webb:** Kod kan komma från hundratals olika domäner

Detta skapar frågor som:
- Får JavaScript från domain A läsa cookies från domain B?
- Kan en iframe från domain C komma åt data från domain D?
- Vad händer om en av de externa källorna blir hackade?

## Unika säkerhetsutmaningar för webben

### 1. Input kommer från internet

**Desktop:** Input kommer från tangentbord, mus, filer
**Webb:** Input kommer från vem som helst på internet

```
Webb-input kan vara:
- Formulärdata som användaren skrev
- URL-parametrar som ändrats manuellt
- HTTP-headers som modifierats
- Cookies som manipulerats
- Filer som laddats upp
```

### 2. Output visas i webbläsaren

**Desktop:** Du kontrollerar hur output visas
**Webb:** Webbläsaren tolkar din output som HTML/CSS/JavaScript

Detta betyder att skadlig data kan bli:
- HTML som ändrar sidan
- JavaScript som körs i webbläsaren  
- CSS som ändrar utseendet

### 3. Sessioner över HTTP

**Desktop:** Programmet "vet" att det är samma användare
**Webb:** Måste konstgjort skapa sessioner med cookies eller tokens

```
Problem med web-sessioner:
- Cookies kan stjälas
- Sessions kan kapas
- Svårt att veta när användaren verkligen loggat ut
```

### 4. Samtidiga användare

**Desktop:** Oftast en användare per installation
**Webb:** Tusentals användare samtidigt

Detta skapar race conditions och samtidighetsproblem som inte finns i desktop-appar.

## Webbens säkerhetslager

På webben måste säkerhet implementeras på fler nivåer:

```
🌍 DNS/Domän-nivå (DNS-hijacking, subdomain takeover)
├── 🔐 Transport-nivå (HTTPS, TLS-certifikat)
├── 🌐 HTTP-nivå (säkra headers, cookies)
├── 🛡️ Applikations-nivå (autentisering, auktorisering)
├── 🔍 Input/Output-nivå (validering, encoding)
├── 🗄️ Data-nivå (SQL injection, data access)
├── 📱 Klient-nivå (XSS, CSRF)
└── 👤 Användar-nivå (social engineering, phishing)
```

**Desktop-appar behöver kanske 2-3 av dessa lager.**

## Säkerhet som standard vs säkerhet som tillägg

### Desktop-utveckling
```
Säkerhet = Främst access control till filsystem
Standard: Relativt säkert från början
```

### Webb-utveckling
```
Säkerhet = Måste byggas in i varje del
Standard: Osäkert från början (HTTP, klartext, öppet)
```

## Exempel: Samma funktion, olika säkerhetsproblem

**Funktion:** "Visa användarens profil"

### Desktop-version:
```
1. Programmet vet vem som är inloggad
2. Hämta profilen från lokal databas
3. Visa profilen i ett fönster

Säkerhetsproblem: Få (mest lokal access control)
```

### Webb-version:
```
1. Vem gjorde denna HTTP-request? (autentisering)
2. Får de se denna profil? (auktorisering)  
3. Är profil-ID:t giltigt? (input-validering)
4. Kan svaret cachas säkert? (cache-policy)
5. Vilka headers ska skickas? (säkerhetsheaders)
6. Hur ska känslig data visas? (output-encoding)
7. Ska detta loggas? (audit logging)

Säkerhetsproblem: Många!
```

## Varför detta är viktigt att förstå

När du förstår vad som gör webbsäkerhet unikt kan du:

1. **Tänka rätt från början** - Inte bara applicera desktop-säkerhet på webben
2. **Prioritera rätt** - Fokusera på de problem som faktiskt existerar
3. **Undvika vanliga fällor** - Som att lita på klient-validering
4. **Bygga robust** - Förstå varför "defense in depth" är nödvändigt

## Nästa steg

Nu när du förstår vad som gör webben speciell, är du redo att lära dig om de specifika säkerhetshot som webben skapar och hur vi skyddar oss mot dem.

---

**Reflektion:** Tänk på en mobil-app du använder vs en webbsida som gör samma sak. Märker du skillnader i hur de hanterar säkerhet? Vilket känns säkrare och varför?
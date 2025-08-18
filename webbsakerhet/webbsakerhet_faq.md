# FAQ - Vanliga frågor om webbsäkerhet

## Allmänna frågor

### Fråga: Är min lilla webbapp verkligen intressant för hackare?
**Svar:** Ja! De flesta attacker är automatiserade. Robotar scannar hela internet efter sårbarheter, oavsett hur stor eller liten din app är. Det är som att säga "min bil är för gammal för att stjälas" - tjuvar testar ändå om den är olåst.

### Fråga: Räcker det inte med HTTPS för att vara säker?
**Svar:** Nej, HTTPS skyddar bara data under transport (mellan webbläsare och server). Det skyddar inte mot SQL injection, XSS, eller broken access control. Det är som att ha pansarglas på bilen men lämna dörrarna olåsta.

### Fråga: Kan jag inte bara kopiera säkerhetslösningar från Stack Overflow?
**Svar:** Farligt! Mycket säkerhetskod på Stack Overflow är förenklad för att visa koncept, inte för produktion. Det är som att kopiera en receptbild istället för receptet - det ser rätt ut men fungerar inte.

### Fråga: Vad är viktigast att fokusera på först?
**Svar:** 
1. **Autentisering** - Vem är användaren?
2. **Auktorisering** - Vad får de göra? 
3. **Input-validering** - Kontrollera all användardata
4. **HTTPS** - Kryptera all kommunikation

## Frågor om autentisering och lösenord

### Fråga: Varför kan jag inte bara spara lösenord som vanlig text?
**Svar:** För att när (inte om) databasen läcker ut ska lösenorden vara oanvändbara. Hashade lösenord är som att förvandla "password123" till "kj3h4k5j6h4k3j5h63k4j5h6" - omöjligt att få tillbaka originalet.

### Fråga: Är det okej att skicka lösenord via e-post?
**Svar:** Aldrig! E-post är som vykort - alla som hanterar posten kan läsa den. Skicka istället en länk för lösenordsåterställning som går ut efter kort tid.

### Fråga: Vad är skillnaden mellan autentisering och auktorisering?
**Svar:** 
- **Autentisering** = "Vem är du?" (inloggning)
- **Auktorisering** = "Vad får du göra?" (behörigheter)

Som att visa ID-kort på krogen (autentisering) och sedan få komma in i VIP-rummet (auktorisering).

### Fråga: Behöver jag tvåfaktorsautentisering (2FA)?
**Svar:** För system med känslig data - absolut! Det är som att ha både lås och larm på huset. Även om någon får lösenordet behöver de fortfarande din telefon/app.

## Frågor om sessions och cookies

### Fråga: Vad är skillnaden mellan sessions och JWT?
**Svar:**
- **Sessions** = Servern kommer ihåg att du är inloggad (som garderobbiljett)
- **JWT** = Du bär med dig bevis att du är inloggad (som ID-kort)

### Fråga: Kan jag lita på cookies från användaren?
**Svar:** Nej! Användare kan ändra cookies precis som de kan ändra vilken text som helst. Validera alltid på servern.

### Fråga: Hur länge ska en session vara giltig?
**Svar:** Beror på applikationen:
- **Bankapp:** 15-30 minuter
- **E-handel:** 2-24 timmar  
- **Social media:** Dagar till veckor
- **Admin-panel:** Mycket kort, kanske 1 timme

## Frågor om databaser och SQL injection

### Fråga: Vad är SQL injection egentligen?
**Svar:** När skadlig SQL-kod "smugglas in" via användarinput. Som att be om någons namn och de svarar "Anna'; DROP TABLE users; --" istället för bara "Anna".

### Fråga: Räcker det att validera input på frontend?
**Svar:** Nej! Frontend-validering är för användarupplevelsen. Säkerhetsvalideringen MÅSTE vara på backend eftersom användare kan stänga av/ändra JavaScript.

### Fråga: Vad är prepared statements?
**Svar:** Ett sätt att separera SQL-kod från data. Som att säga "fyll i namnet här: _____" istället för att bygga meningen genom att klistra ihop delar.

## Frågor om HTTPS och kryptering

### Fråga: Vad är skillnaden mellan HTTP och HTTPS?
**Svar:**
- **HTTP** = Vykort (alla kan läsa)
- **HTTPS** = Försegelt brev (endast mottagaren kan läsa)

### Fråga: Kostar HTTPS pengar?
**Svar:** Inte längre! Let's Encrypt ger gratis SSL-certifikat. Det finns ingen anledning att inte använda HTTPS idag.

### Fråga: Kan jag blanda HTTP och HTTPS på samma sajt?
**Svar:** Undvik det! "Mixed content" skapar säkerhetshål och varningar i webbläsaren. Använd HTTPS överallt.

## Frågor om felhantering och loggning

### Fråga: Får jag visa detaljerade felmeddelanden för användaren?
**Svar:** Nej för säkerhetsfel! Visa generiska meddelanden som "Inloggning misslyckades" istället för "Användaren finns inte" eller "Fel lösenord". Logga detaljerna istället.

### Fråga: Vad ska jag logga?
**Svar:** 
**Logga:**
- Inloggningsförsök (lyckade och misslyckade)
- Behörighetsfel
- Misstänkt aktivitet

**Logga ALDRIG:**
- Lösenord
- Kreditkortsnummer
- Andra känsliga personuppgifter

### Fråga: Hur länge ska jag spara loggar?
**Svar:** Beror på krav, men vanligt är:
- **Säkerhetsloggar:** 1-2 år
- **Accessloggar:** 3-6 månader
- **Applikationsloggar:** 1-3 månader

## Frågor om utvecklingsprocess

### Fråga: När ska jag tänka på säkerhet i utvecklingsprocessen?
**Svar:** Från dag ett! Det är mycket dyrare att "lägga till säkerhet" efteråt än att bygga in det från början. Som att bygga ett hus utan fundament och sen försöka gräva under det.

### Fråga: Hur testar jag säkerheten i min app?
**Svar:**
1. **Automatiska verktyg** (OWASP ZAP, Burp Suite)
2. **Manuell testning** (testa med konstiga input)
3. **Code review** (låt kollegor granska koden)
4. **Penetrationstestning** (anlita experter)

### Fråga: Vad gör jag om jag hittar en säkerhetsbrist i produktion?
**Svar:**
1. **Bedöm allvarlighetsgraden** - Akut eller kan vänta?
2. **Fixa problemet** - Prioritera säkerhet över funktionalitet
3. **Analysera påverkan** - Vilka användare påverkades?
4. **Förbättra processer** - Hur förhindrar vi detta framöver?

## Frågor om prestanda och säkerhet

### Fråga: Gör säkerhet min app långsam?
**Svar:** Lite, men det märks sällan. Hashning av lösenord tar millisekunder, HTTPS lägger till minimal overhead. Användarupplevelsen är viktigare än att spara några millisekunder.

### Fråga: Kan jag cacha säkerhetsdata för bättre prestanda?
**Svar:** Försiktigt! Vissa saker som användarroller kan cachas kort, men känslig data eller sessioner ska inte cachas länge. Säkerhet före hastighet.

## Frågor om tredjepart och bibliotek

### Fråga: Kan jag lita på open source-bibliotek?
**Svar:** Populära bibliotek med aktiv community är generellt säkra, men:
- Håll dem uppdaterade
- Använd verktyg som kontrollerar kända sårbarheter
- Läs inte bara dokumentationen, förstå vad biblioteket gör

### Fråga: Vad gör jag om ett bibliotek jag använder har en säkerhetsbrist?
**Svar:**
1. **Uppdatera omedelbart** om möjligt
2. **Hitta workarounds** om uppdatering inte är möjlig
3. **Byt bibliotek** om det inte längre underhålls
4. **Informera användare** om det påverkar dem

## Frågor om lagar och regleringar

### Fråga: Vilka lagar måste jag följa?
**Svar:** Beror på var du och dina användare finns:
- **GDPR** (EU) - Personuppgifter
- **PCI DSS** - Om du hanterar kreditkort
- **Branschspecifika** - Sjukvård, finans, etc.

### Fråga: Vad händer om min app blir hackad?
**Svar:** Beror på skada och lagar:
- **Anmälningsplikt** - Måste ofta rapportera till myndigheter
- **Användarnotifiering** - Informera påverkade användare
- **Juridiska konsekvenser** - Böter och skadestånd
- **Förtroendeskada** - Kanske värst av allt

## Praktiska tips

### Fråga: Vad är första tecknet på att min app attackeras?
**Svar:** Vanliga signaler:
- Många misslyckade inloggningsförsök
- Konstiga 404-fel (någon testar URL:er)
- Ovanlig trafik från samma IP
- Långsamma svarstider (DDoS-attack)

### Fråga: Vad ska jag göra JUST NU för att förbättra säkerheten?
**Svar:**
1. **Aktivera HTTPS** om du inte har det
2. **Uppdatera alla bibliotek** till senaste version
3. **Lägg till input-validering** på server-sidan
4. **Implementera rate limiting** för inloggning
5. **Börja logga säkerhetsrelevanta händelser**

---

**Kom ihåg:** Det är okej att inte veta allt från början. Säkerhet är något du lär dig kontinuerligt. Det viktiga är att börja någonstans och gradvis förbättra!
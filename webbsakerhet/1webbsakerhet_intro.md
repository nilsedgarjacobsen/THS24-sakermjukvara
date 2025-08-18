# Introduktion till webbsäkerhet

## Vad är en webbapplikation?

När du besöker en webbsida händer det mycket mer än vad du ser. Bakom kulisserna finns det en **webbserver** - en dator som väntar på att få frågor från webbläsare och svarar med webbsidor.

### Så här fungerar det
```
Din webbläsare              Webbservern
     |                           |
     |  "Visa startsidan"        |
     |-------------------------->|
     |                           |
     |    HTML + CSS + JS        |
     |<--------------------------|
     |                           |
```

**Webbservern** är som en restaurang:
- Den tar emot beställningar (HTTP-requests)
- Lagar mat (bearbetar data)
- Serverar tallrikar (skickar HTML-sidor)

**Din webbläsare** är som en kund:
- Beställer mat (skickar requests)
- Äter maten (visar webbsidan)
- Kan beställa mer (klicka på länkar)

## Vad händer när vi sätter en webbapp på nätet?

När du utvecklar hemma är du säker - bara du kan komma åt din applikation. Men när du publicerar den på internet förändras allt.

### Hemma vs Internet

**På din dator:**
```
- Bara du kan komma åt appen
- Du vet vad du kommer att skriva i formulär
- Du använder appen "som den är tänkt"
```

**På internet:**
```
- Vem som helst i världen kan komma åt
- Folk kan skriva vad som helst i formulär  
- Människor (och robotar) testar gränser
```

Det är som skillnaden mellan att lämna din cykel i ditt garage vs att lämna den olåst på en stor torg.

## Varför är webbsäkerhet viktigt?

### För användarna
- Deras personuppgifter ska vara säkra
- Deras konton ska inte kunna kapas
- De ska kunna lita på att systemet fungerar

### För företaget
- Förtroende från kunder
- Undvika juridiska problem
- Skydda affärshemligheter

### För dig som utvecklare
- Bygga system som håller över tid
- Professionell stolthet
- Karriärmöjligheter

## Grundläggande risker med webbapplikationer

När din app är på internet finns det flera typer av risker:

### 1. Obehörig åtkomst
**Vad det betyder:** Någon kommer in där de inte ska vara

**Exempel:**
- En vanlig användare kommer åt admin-panelen
- Någon läser andras privata meddelanden
- En person ändrar priser i en webbutik

### 2. Dataskada eller förlust
**Vad det betyder:** Viktig information förstörs eller förändras

**Exempel:**
- Någon raderar alla användarkonton
- Produktpriser ändras till 0 kronor
- Kundregister försvinner

### 3. Tjänsteavbrott
**Vad det betyder:** Webbappen slutar fungera

**Exempel:**
- För många besökare gör att servern kraschar
- Någon överbelastas systemet med förfrågningar
- Servern blir så långsam att ingen kan använda den

### 4. Dataintrång
**Vad det betyder:** Känslig information läcker ut

**Exempel:**
- Lösenord blir synliga för utomstående
- Kreditkortsnummer hamnar i fel händer
- Privata meddelanden publiceras

## Vem kan vara ett hot?

### Nyfikna användare
- Klickar på saker de inte ska
- Testar "vad händer om jag gör så här?"
- Oftast ofarliga men kan hitta problem

### Automatiserade robotar
- Scannar internet efter sårbara webbsidor
- Testar vanliga säkerhetshål
- Körs 24/7 utan paus

### Riktade angripare
- Har ett specifikt mål (ditt företag, din data)
- Lägger tid på att studera din webbapp
- Mest farliga men minst vanliga

### Misstag och slump
- Användare som råkar skriva fel saker
- Buggar som orsakar säkerhetsproblem
- Faktiskt den vanligaste orsaken till problem!

## De tre grundpelarna för säkerhet

All webbsäkerhet handlar om att skydda tre saker:

### 1. Konfidentialitet
**"Rätt personer ser rätt information"**

Exempel: Bara du ska kunna se dina egna meddelanden

### 2. Integritet  
**"Information ändras bara av rätt personer"**

Exempel: Bara bankens system ska kunna ändra ditt saldo

### 3. Tillgänglighet
**"Systemet fungerar när det behövs"**

Exempel: Du ska kunna logga in när du vill använda tjänsten

## Säkerhet är en process, inte en produkt

Säkerhet är inte något du "installerar" eller "fixar en gång". Det är som att hålla sitt hem rent - det kräver kontinuerligt arbete.

### Säkerhet påverkar allt du bygger
- Varje ny funktion = nya säkerhetsrisker
- Varje användare = ny potentiell risk
- Varje uppgradering = möjlighet att förbättra eller förvärra säkerheten

### Säkerhet vs Användarvänlighet
Det finns alltid en balansgång:
- Mer säkerhet kan göra saker krångligare
- Mer användarvänlighet kan göra saker mindre säkra
- Din uppgift: Hitta rätt balans för ditt system

## Vad kommer härnäst?

I nästa avsnitt kommer vi att utforska vad som gör just webbsäkerhet unikt jämfört med andra typer av säkerhet. Efter det tittar vi på de vanligaste säkerhetshoten och hur vi skyddar oss mot dem.

**Kom ihåg:** Målet är inte att bli rädd för att sätta din app på nätet, utan att förstå riskerna så att du kan hantera dem på ett smart sätt.

---

**Reflektion:** Tänk på webbsidor du använder dagligen. Vilken typ av information litar du dem med? Vad skulle hända om den informationen hamnade i fel händer?
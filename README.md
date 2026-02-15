# KalenteriProjektin toimintalogiikka

Tämä dokumentti kuvaa **Calendar Assistant** -sovelluksen toimintaa: miten se etsii vapaat ajat ja luo ehdotukset kalenterimerkintöihin. Dokumentti on kirjoitettu niin, että sekä tavallinen käyttäjä että kehittäjä ymmärtävät järjestelmän.

---

## Idea

**Calendar Assistant** automaattistaa valmennusprojektien kalenterointia. Sen tehtävä on yksinkertainen:

1. Sinulla on projekti, joka vaatii useita kalenterimerkintöjä (esim. suunnitteluaikoja, Teams-palaveri, valmistelutöitä).
2. Sovellus hakee sinun (ja mahdollisesti valmentajan) kalenterin varaukset.
3. Se etsii vapaat ajat ja ehdottaa sopivia aikavälit – yhdellä klikkauksella.

Algoritmi toimii kuin palapeli: se pilkkoo projektin tarvitsemiin aikoihin, etsii kalenterista "reiät" ja sovittaa palaset niihin, huomioiden tauot ja suosituimmat ajat.

---

## Käyttäjän näkökulmasta: mitä tapahtuu?

| Vaihe | Mitä käyttäjä tekee | Mitä järjestelmä tekee taustalla |
|-------|---------------------|----------------------------------|
| 1 | Luo projektin (nimi, **hakuväli**, tyyppi) | Laskee tarvittavan kaavan (esim. Suunnittelu + Teams + Valmistelu) ja toistaa sitä koko valitulle hakuvälille (esim. 1 kk). |
| 2 | Klikkaa "Etsi vapaat ajat" | Hakee sinun ja valmentajan kalenterit, etsii vapaat ajat, luo ehdotukset |
| 2b | Klikkaa "Etsi varatut ajat" | Hakee sovelluksen kautta luodut merkinnät aikavälillä (tänään → valmennuspäivä). Näyttää vain kalenterissa olevat (poistetut ei näy). |
| 3 | Suodattaa ja valitsee ehdotukset | Voi rajata näkymää **Asiakas**- ja **Päivämäärä**-valikoilla (tekstihaku). Lista rullautuu jos kortteja on yli 10. "Luo merkinnät" luo vain näytetyt kortit. |
| 4 | Tarkastelee "Varatut ajat" -osiota | Näkee jo luodut merkinnät (kuka varasi, luontipäivämäärä). Sama lista tulee "Etsi vapaat ajat" -haun jälkeen automaattisesti. |

**Tärkeää:** Jos valmentajan sähköposti on asetettu, järjestelmä tarkistaa molemmat kalenterit. Ehdotetut ajat ovat vapaat sekä sinulle että valmentajalle (valmentajan tili tulee olla Microsoft 365 -tilillä).

**Varatut ajat:** Merkinnät, jotka on poistettu Microsoft-kalenterista, eivät näy "Varatut ajat" -osiossa – järjestelmä tarkistaa Graph API:sta, mitkä tapahtumat ovat yhä olemassa.

---

## Projektityypit ja tarvittavat ajat

Kun valitset projektityypin, järjestelmä laskee automaattisesti tarvittavat aikavälit:

| Projekti | Suunnittelu | Teams-palaveri | Valmistelu | Yhteensä |
|----------|-------------|----------------|------------|----------|
| **Standard** | 2 × 3 h | 1 × 1 h | 2 × 3 h | 13 h |
| **Express** | 1 × 3 h | 1 × 1 h | 1 × 3 h | 7 h |
| **Extended** | 3 × 3 h | 1 × 2 h | 3 × 3 h | 20 h |

Teams-palaveri voi sisältää valmentajan sähköpostin, jos se on asetettu preferensseissä.

---

## 🔍 Suodatus ja Selaus (UUSI v5.2)

Sovelluksessa on edistyneet työkalut suurten ehdotusmäärien (esim. 50+ korttia) hallintaan:

1.  **MultiSelect-suodattimet:**
    -   **Asiakas:** Valitse listasta yksi tai useampi asiakas.
    -   **Päivämäärä:** Valitse tarkat päivät.
    -   **Tekstihaku:** Voit kirjoittaa valikon sisään (esim. "Fir") löytääksesi nopeasti oikean vaihtoehdon.
2.  **Scrollaus:**
    -   Jos lista on pitkä (>10 korttia), se rullautuu pystysuunnassa (max korkeus n. 600px).
    -   Painikkeet ja otsikot pysyvät aina näkyvissä.
3.  **Luo merkinnät -logiikka:**
    -   Painike luo merkinnät **vain suodatetusta ja näkyvästä listasta**. Tämä mahdollistaa esim. vain tietyn asiakkaan aikojen massaluonnin.

---

## Miten vapaa aika löydetään? (Algoritmin ytimessä)

Aikataulutusmoottori toimii kolmessa vaiheessa. Tämä on se osa, joka tekee "magian" – tässä selitetään tarkasti miten.

### Vaihe 1: Tarpeen laskenta (`CalculateRequiredSlots`)

Ensin järjestelmä määrittää, mitä projekti vaatii:
- Montako slotia kustakin tyypistä (suunnittelu, Teams, valmistelu)
- Kuinka pitkiä ne ovat (esim. 3 h)
- Missä järjestyksessä ne tulevat (prioriteetti: suunnittelu ensin, sitten Teams, sitten valmistelu)

### Vaihe 2: Vapaiden aikojen etsintä (`FindAvailableSlots`)

Tämä on logiikan sydän. Järjestelmä käy läpi päivät ja etsii sopivia aikoja.

1. **Työpäivät**  
   Vain sellaiset päivät huomioidaan, jotka on merkitty työpäiviksi (ma–su, asetuksista).

2. **Työaika**  
   Työajan ulkopuolelle ei ehdoteta aikoja (esim. 8:00–16:00).

3. **Varaukset**  
   Haetaan kaikki varaukset kyseiselle päivälle – sekä sinun että valmentajan kalenterista (jos valmentaja on määritelty).

4. **Aukkojen etsintä**  
   Algoritmi etenee aikajanalla aina seuraavaan varaukseen asti:
   - **Tauko ennen varausta:** Jos seuraava varaus alkaa klo 11:00, vapaa aika päättyy 10:30 (30 min tauko oletuksena).
   - **Tauko varauksen jälkeen:** Varauksen (esim. 10:00–11:00) jälkeen seuraava vapaa aika voi alkaa vasta 11:30.
   - **Vähimmäispituus:** Jos aukko on alle 1 tunnin, sitä ei käytetä.

5. **Suositeltu aika**  
   Jos vapaa aika osuu "suositeltuun aikaan" (esim. 8:00–12:00), se merkitään prioriteettiseksi (`IsPreferred`). Nämä valitaan ensin.

### Vaihe 3: Ehdotusten luominen (`GenerateProposals`)

Lopuksi tarpeet ja vapaat ajat yhdistetään. **Tämä vaihe on muuttunut versiossa 5.1:**

1.  **Toistuva kaava (Loop):**
    Järjestelmä ei lopeta yhden kierroksen jälkeen, vaan **toistaa projektin kaavaa** (Suunnittelu → Teams → Valmistelu) niin kauan kuin vapaata tilaa riittää hakuvälillä.

2.  **Settien luonti:**
    -   **Setti 1:** Ensimmäinen täysi kierros on oletuksena valittu (`IsAccepted = true`).
    -   **Setti 2, 3, jne.:** Seuraavat kierrokset luodaan vaihtoehdoiksi, joista käyttäjä voi valita lisää aikoja.

3.  **Prioriteettijärjestys:**
    Kaavan sisällä välilyönnit täytetään tärkeysjärjestyksessä: ensin suunnittelu, sitten Teams, sitten valmistelu.

4.  **Sopivan ajan valinta:**
    Jokaiselle tarvittavalle välille etsitään vapaa aika, joka:
    -   On riittävän pitkä (esim. 3 h)
    -   Ei ole jo käytetty toiseen ehdotukseen
    -   Ei riko minimiväliä edellisen ehdotuksen jälkeen
    -   Suositaan ensin "Preferred"-aikoja

5.  **Varatilanne (fallback) - Vain 1. setti:**
    Jos ensimmäisessä setissä ei löydy täydellistä 3 h aikaa, järjestelmä yrittää lyhentää sitä (varoitus: *"Ei täydellistä aikaa, lyhennetty"*). Seuraavissa seteissä (2+) ei käytetä fallbackia, vaan jos täydellistä aikaa ei löydy, settiä ei jatketa.

---

## Lisäominaisuudet (tarkennukset)

| Ominaisuus | Selitys |
|------------|---------|
| **Minimiväli ehdotusten välillä** | Ehdotusten välillä vaaditaan tauko (esim. 30 min). Esim. 08–11 ja 11–14 eivät kelpaa peräkkäin; seuraava voi alkaa vasta 11:30 |
| **Jäljellä olevan ajan käyttö** | Kun slotista käytetään vain osa (esim. 08–11 osana 08–16), jäljellä oleva aika (11:30–16) lisätään käytettävissä oleviin. Näin samalle päivälle voi tulla useampi ehdotus ilman päällekkäisyyksiä |

---

## Tarkistus ennen merkinnän luontia

Ennen kuin merkintä luodaan kalenteriin, järjestelmä tarkistaa vielä kerran (`IsSlotStillAvailable`), että aika on vapaa – myös valmentajan kalenterilla, jos sellainen on käytössä. Jos aika on jo varattu (toinen käyttäjä varaa samaan aikaan), merkintää ei luoda ja käyttäjälle näytetään virheviesti.

---

## Merkinnän muokkaus

Ennen merkinnän luontia käyttäjä voi muokata yksittäisiä ehdotuksia (kynä-ikoni). Muokkausmodaalissa voi muuttaa:
- Otsikko
- Asiakas
- **Lisätieto** (valinnainen tekstikenttä)
- Päivämäärä
- Alkaa / Päättyy

---

## Älykäs Uudelleenjärjestely (Smart Rescheduling) ( KESKEN )

Kun kalenteri on täysi, "Älykäs haku" voi auttaa löytämään tilaa siirtämällä vähemmän tärkeitä omia varauksia.

### Miten se toimii?

Kun valitset **"Etsi myös siirrettävät ajat"**, algoritmi tekee seuraavat asiat:
1.  **Analyysi**: Se käy läpi varatut ajat ja tunnistaa, mitkä niistä ovat "siirrettäviä".
2.  **Pisteytys**: Varaus katsotaan siirrettäväksi (Moveable), jos:
    *   **Olet järjestäjä (Organizer):** Sinulla on valta siirtää se.
    *   **Ei muita osallistujia:** Kyseessä on oma työaika (esim. "Focus", "Suunnittelu"), ei palaveri.
    *   **Status on normaali:** Ei ole merkitty "Out of Office" -tilaan.
    *   **Ei toistuva:** Toistuvien sarjojen siirtäminen on riskialtista, joten niihin ei kosketa.
3.  **Haku**: Järjestelmä etsii vapaata aikaa *kuvittelemalla*, että nämä siirrettävät varaukset eivät ole tiellä.
4.  **Ehdotus**: Jos aika löytyy tällaisen varauksen päältä, se ehdotetaan sinulle varoituksella: _"Vapaa JOS siirrät: 'Oma suunnittelu'"_.


### Mitä se ehdottaa siirrettäväksi?

Sovelluksen itse luomista merkinnöistä:

*   **✅ Suunnittelu (Planning) ja Valmistelu (Preparation):**
    *   Nämä luodaan ilman muita osallistujia.
    *   Järjestelmä tulkitsee ne omaksi työajaksi, jota voi tarvittaessa siirtää tärkeämmän tieltä.
    *   *Esimerkki: "Suunnittelu 1/3: Asiakas Oy"*

*   **❌ Teams-palaveri:**
    *   Nämä luodaan online-kokouksina.
    *   Järjestelmä pitää näitä kiinteinä aikatauluvarauksina, eikä ehdota niiden siirtämistä (vaikka olisit niissä yksin).

Tämä on **deterministinen heuristiikka**. Se ei käytä tekoälyä arvailuun, vaan noudattaa tiukkoja sääntöjä (Organizer + No Attendees), jotta se ei koskaan ehdota asiakaspalaverin siirtoa.

---

## Tietosuoja ja luetut tiedot (Privacy)

Jotta sovellus voi toimia älykkäästi, se lukee kalenteristasi seuraavat tiedot Microsoft Graph -rajapinnan kautta (`Calendar.ReadWrite`):

1.  **Ajankohta:** `start`, `end`, `timeZone` (Milloin varaus on)
2.  **Tila:** `showAs` (Onko aika varattu, vapaa, alustava vai "Out of Office")
3.  **Otsikko:** `subject` (Tarvitaan "Älykkäässä haussa" tunnistamaan omat merkinnät, kuten "Lounas")
4.  **Osallistujat:** `attendees` (Onko muita mukana? Jos on, emme ehdota siirtoa)
5.  **Järjestäjä:** `isOrganizer` (Oletko sinä luonut varauksen?)
6.  **Toistuvuus:** `recurrence`, `type` (Onko kyseessä toistuva sarja?)

**Huomio:** Sovellus **EI** lue sähköpostien sisältöä, liitetiedostoja tai body-tekstiä. Se käsittelee vain kalenterin metatietoja aikataulutusta varten. Kaikki tiedot käsitellään väliaikaisesti muistissa.

## Yhteenveto

Lyhyesti: järjestelmä pilkkoo projektin tarvitsemiin aikoihin, etsii kalenterista vapaat "reiät" (huomioiden tauot ja työajat), sovittaa palaset paikoilleen ja suosii aamupäiväaikoja sekä valmennuspäivän läheisiä slotteja. Kaikki tämä tapahtuu yhdellä "Etsi vapaat ajat" -napin painalluksella. **"Etsi varatut ajat"** -nappi näyttää jo luodut merkinnät ilman vapaiden aikojen hakua.

---

## Tietoturvallisuus

Sovellus käyttää useita turvatoimia API- ja käyttäjädatan suojaukseen:

| Suoja | Mitä vaikuttaa |
|-------|----------------|
| **Rate Limiting** | 100 req / 10 s per käyttäjä tai IP. OAuth callback: 10 req / min per IP (brute-force -suojaus). Estää DoS-hyökkäyksiä. |
| **JWT-autentikaatio** | Varmistaa, että vain kirjautuneet käyttäjät pääsevät API-endpointeihin. ValidateIssuer/Audience/Lifetime/SigningKey = true, ClockSkew 2 min, ValidAlgorithms = HS256 (estää alg:none). JWT-eventit loggaavat vain devissä. |
| **CORS** | Sallii pyynnöt vain määritellyistä alkuperistä (dev: localhost, prod: whitelist). Estää muut sivustot käyttämästä API:asi. |
| **Security Headers** | Backend: CSP, HSTS jne. Frontend: Middleware lisää CSP (connect-src Supabase + API), X-Content-Type-Options, X-Frame-Options. Vähentävät XSS-, clickjacking- ja MIME-sniffing -riskejä. |
| **HTTPS (tuotanto)** | Pakottaa salatun yhteyden tuotannossa. Suojaa datan man-in-the-middle -hyökkäyksiltä. |
| **OAuth state + CSRF** | Microsoft-kirjautumisen `state`-parametri on HMAC-allekirjoitettu. Estää väärennetyt OAuth-callbackit. |
| **Token-salaus** | Microsoft Graph -tokenit salataan (AES-256) ennen tietokantaan tallennusta. Suojaa, vaikka tietokanta vuotaisi. |
| **Input Validation** | FluentValidation tarkistaa syötteet (esim. preferenssit) ennen käsittelyä. Estää virheellisen tai haitallisen datan. |
| **PII-maskaus** | Sähköposteja ei kirjoiteta logeihin selkokielisinä (maskataan tyyliin `m***@firma.fi`). Vähentää henkilötietojen vuotamista lokista. |
| **API-avainten suojaus** | Arkaluonteiset avaimet (JWT Secret, Microsoft Graph ClientSecret, ClientId, TenantId, Supabase Url) säilytetään backendin `.env`-tiedostossa, eivät koskaan repossa tai frontendissa. Käytä `.env.example`-pohjaa. Frontend saa vain Supabasen anon key (suunniteltu julkiseksi; RLS rajaa oikeudet). `service_role` -avainta ei käytetä client-puolella. Graph-tokenit salataan tietokantaan (Data Protection API). |
| **Microsoft OAuth TenantId** | Käytä `TenantId=common` henkilökohtaisten Microsoft-tilien (outlook.com, hotmail.com) tukemiseksi. Org-spesifiselle käytölle käytä Azure AD:n tenant-ID:tä. |
| **Lokituksen turvallisuus** | OAuth- ja Graph API -virhevastauksia ei logiteta (estää token- ja PII-vuodon). Logitetaan vain StatusCode ja ReasonPhrase. |
| **Frontend build-tarkistus** | Next.js build epäonnistuu, jos `NEXT_PUBLIC_`-avaimissa on `SERVICE_ROLE` tai `SECRET_KEY`. Estää vahingollisen paljastuksen build-vaiheessa. |
| **postMessage origin-validoinnit** | Microsoft OAuth -popup lähettää `msauth-callback`-viestin frontendille. Frontend tarkistaa `event.origin` ja hyväksyy vain backendin OAuth-callback-origin (`NEXT_PUBLIC_API_URL`). Estää, että vihamielinen sivusto lähettäisi vääriä viestejä. Backend käyttää `Cors:AllowedOrigins` -konfiguraatiota postMessage `targetOrigin`-parametrina (ei wildcardia `*`). |
| **Tuotantologitus (frontend)** | `console.error` rajoitettu kehitysympäristöön (`NODE_ENV === 'development'`). Tuotannossa virheitä ei logata selaimen konsoliin – vähentää token- tai PII-vuodon riskiä, jos virheolio sisältäisi arkaluontoisia tietoja. |
| **Dependency Security (Supply Chain)** | Backend: `dotnet list package --vulnerable`. Frontend: `npm audit`. Suositus: Dependabot/Renovate automaattisille PR:ille, säännöllinen patchaus (viikoittain), kriittisten CVE:tien korjaus 48 h. Supply chain -riskit (vahingoittuneet paketit, typosquatting) ovat merkittävämpiä kuin suorat injektiot. |

---


## Korjattavaa


1. **Projektin laajuus customoituna. Suunnittelu / valmistelu = Molemmat suunnitelu yhdistä. Omat tekstiboxit / Save / load asetukset**
2. **Sijainnin tunnistus ominaisuus. Kalenteriin varattujen merkintöjen sijainnin huomiominen, Jos helsinki niin ei ehdota siihen mitään. (Varoituksen kanssa voi ehdottaa suunnitelua)**
3. **Kategorioiden mukaan priorisointi ( Puhelimessa kuva. Monalta järjestys )**
4. **Suunnittelu labelien värit vastaa, Valitse ajatboxien reunoja + Otsikot vastaa väriä.**
5. **Muokkaa otsikko vaihtaa labelit oikein. Esim Jos vaihtaa otsikon suunnittelusta > Teams vaihtuu myös labelit ja värit oikein**

---

## Suoritetut Korjaukset | 15/02/26

1. **Kalenteri merkinnät näkyviin 6kk ajalta**
2. **Suunnitelu aikataulut ei hakenut 08 eteenpäin**
3. **Microsoft kirjautuminen ei toiminut, Osoite localhost:8080 API Calendar**

---

**Dokumentin versio:** 5.2.0 | **Päivitetty:** 2026-02-15 (MultiSelect Filtering with Search & Scrollable UI)


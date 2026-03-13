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
| 2 | Klikkaa "Etsi vapaat ajat" | Hakee sinun kaikista omista kalentereistasi ja valmentajan kalenterit, etsii vapaat ajat, luo ehdotukset |
| 2b | Klikkaa "Etsi varatut ajat" | Hakee varatut merkinnät aikavälillä (tänään → valmennuspäivä) kaikista omista kalentereista (v5.8.3). Näyttää vain kalenterissa olevat (poistetut ei näy). |
| 3 | Suodattaa ja valitsee ehdotukset | Voit vaihtaa näkymää: **Kortti**, **Taulukko** tai **Kalenteri** (ViewSwitcher). Voi rajata listaa **Asiakas**- ja **Päivämäärä**-valikoilla (tekstihaku). "Luo merkinnät" luo vain näytetyt kortit. |
| 4 | Tarkastelee "Varatut ajat" -osiota | Näkee jo luodut merkinnät (kuka varasi, luontipäivämäärä). Sama lista tulee "Etsi vapaat ajat" -haun jälkeen automaattisesti. |

### 4. Valmentajan kalenterin huomioiminen
- **ICS-integraatio:** Käyttäjä voi syöttää asetuksiin valmentajan julkisen ICS-linkin (esim. Outlookista).
- **Aktivointi (v5.8.0):** Uusi "Ota valmentajan seuranta käyttöön" -valinta (checkbox). Kun pois päältä, valmentajan kalenteria (Email/ICS) ei haeta eikä huomioida.
- **Vapaiden aikojen haku:** Järjestelmä hakee valmentajan kalenteritapahtumat annetusta linkistä.
- **Visualisointi:** Valmentajan menot näkyvät kalenterissa harmaina "Varattu"-palkkeina (ClientName = "Valmentaja").
- **Esto:** Järjestelmä ei ehdota aikoja, jotka menevät päällekkäin valmentajan menojen kanssa.
- **Välimuisti:** ICS-kalenterin tiedot tallennetaan välimuistiin (15 min) suorituskyvyn optimoimiseksi.

**Tärkeää:** Jos valmentajan sähköposti on asetettu, järjestelmä tarkistaa molemmat kalenterit. Ehdotetut ajat ovat vapaat sekä sinulle että valmentajalle (valmentajan tili tulee olla Microsoft 365 -tilillä).

**Varatut ajat:** Merkinnät, jotka on poistettu Microsoft-kalenterista, eivät näy "Varatut ajat" -osiossa – järjestelmä tarkistaa Graph API:sta, mitkä tapahtumat ovat yhä olemassa. Järjestelmä hakee merkinnät kaikista käyttäjän omista kalentereista (v5.8.3).

### 5. Päällekkäisten varauksien näyttö (v5.8.1)
Kun kalenterissa on sekä omia että toisen (jaetun/ICS) merkintöjä, sama aika voi olla varattuna useampaan kertaan. Kalenterinäkymässä:
- **Varatut kortit:** Jos oma varaus ja toisen merkintä menevät päällekkäin, varatun kortin **alaosassa** (scrollin ulkopuolella) näkyy painike **"Näytä päällekkäiset (N)"**.
- **Modaali:** Painikkeen avulla avautuu **OverlapModal**: vasemmalla oma merkintä, oikealla toisen/toisten merkinnät (otsikko, aika, asiakas, lisätiedot, sijainti, varannut).
- **Ehdotetut slotit:** Ehdotetuissa (Suunnittelu/Teams) korteissa tätä painiketta ei ole – järjestelmä ei ehdota päällekkäisiä aikoja.

### 6. Oma vs toisen merkintä (v5.8.2)
- **Tunnistus:** Graph API:n `isOrganizer` määrittää, onko merkintä oma vai toisen. Sähköpostivertailu varavaihtoehtona.
- **Poisto ja muokkaus:** Toisen käyttäjän merkintää ei voi poistaa eikä muokata. Poisto-nappi (X) näkyy vain omille merkinnöille.
- **Modaali:** Toisen merkinnän klikkaus avaa muokkausmodaalin **vain tarkastelutilassa** – otsikko "Vain tarkastelu", kentät read-only, "Kenellä varaus" -laatikko näyttää varauksen tekijän.
- **Kortin järjestys:** Asiakas → Lisätiedot → Sijainti → Varannut (alimpana).
- **Overlap-painike:** Säilyttää oranssit värit kaikissa päällekkäisissä korteissa (myös toisen merkinnän kortilla).

### 7. Ulkoisten merkintöjen tietojen säilytys (v5.8.2)
Järjestelmä ei muokkaa toisen käyttäjän merkitsemiä tietoja:
- **Graph body:** API hakee tapahtuman `body`-kentän. Parsitaan Asiakas ja Lisätiedot (tuki: "Asiakas: X", "Suunnittelu: X", "Teams-palaveri: X").
- **Ei ylikirjoitusta:** Kun body sisältää asiakkaan tai kuvauksen, ne näytetään sellaisenaan. Vain jos body on tyhjä, käytetään oletusta "Muu varaus (Outlook)".

### 8. Asetusten tallennus ja kalenterin päivitys (v5.8.2)
Kun tallennat asetukset (esim. kytket "Ota kalenterin jakaminen käyttöön" pois ja painat "Tallenna asetukset"), kalenterinäkymä päivittyy automaattisesti. Valmentajan ICS-merkinnät katoavat ilman sivun manuaalista päivitystä. Jos olet aiemmin hakenut vapaat ajat, järjestelmä hakee ne uudestaan uusilla asetuksilla (v5.8.3).

### 9. Kaikki omat kalenterit (v5.8.3)
Sovellus hakee merkinnät **kaikista käyttäjän omista kalentereista** (oletus + lisäkalenterit kuten Henkilökohtainen, Työ). Jos käyttäjällä on useita kalentereita Outlookissa, kaikki omat merkinnät näkyvät. Jaetut kalenterit (kollegan kalenteri) jätetään pois – ne hoidetaan Valmentajan ICS-asetuksella. Haetaan vain tämän päivän ja tulevaisuuden merkinnät.

### 10. Kalenterinäkymän korttien järjestys (v5.8.3)
Kalenterinäkymässä varatut ja ehdotetut kortit näytetään nyt **aikajärjestyksessä** – yhdistetty lista, ei enää varatut ensin ja ehdotetut perään. Esim. 08:00-ehdotus näkyy oikeassa paikassa ennen 10:00-varausta.

---

## Projektityypit ja tarvittavat ajat

Kun valitset projektityypin (presetin), järjestelmä laskee automaattisesti tarvittavat aikavälit:

| Projekti | Suunnittelu | Teams-palaveri | Yhteensä |
|----------|-------------|----------------|----------|
| **Standard** | 4 × 3 h | 1 × 1 h | 13 h |
| **Express** | 2 × 3 h | 1 × 1 h | 7 h |
| **Laaja (Extended)** | 6 × 3 h | 1 × 2 h | 20 h |

> **Huom:** Vanhempi dokumentaatio mainitsee "Valmistelu"-slottityypin. Tämä on yhdistetty Suunnittelu-slotteihin yksinkertaisuuden vuoksi. Järjestelmä käyttää nyt vain kahta tyyppiä: **Suunnittelu** (yksin työskentely) ja **Teams-palaveri** (valmentajan kanssa).

Teams-palaveri voi sisältää valmentajan sähköpostin kutsuttuna, jos se on asetettu preferensseissä.

---

## 🛠️ Projektin laajuus & Presetit (v5.4)

Käyttäjä voi nyt vapaasti määrittää projektin laajuuden ns. **Custom Scope** -näkymässä, ilman lukittuja "Standard/Express" -paketteja.

### 1. Käyttöliittymä
*   **Korttien asettelu:** Suunnittelu ja Teams ovat omia, selkeitä korttejaan, pinottuina päällekkäin.
*   **Syötteet:** Molemmille voi antaa **määrän** (kpl) ja **keston** (h).
    *   *Esim.* Suunnittelu: `4 kpl × 3 h`
    *   *Esim.* Teams: `1 kpl × 1 h`

### 2. Presettien hallinta
Usein toistuvat asetukset voi tallentaa **presetiksi** (esim. "Iso Asiakas"), jolloin ne voi ladata yhdellä klikkauksella.

#### Tallennus (Save Preset)
1.  Säädä arvot haluamallasi tavalla.
2.  Klikkaa **"Tallenna asetuksena"**.
3.  Avautuu modaali, joka näyttää **yhteenvedon** tallennettavista arvoista (varmistus).
4.  Kirjoita nimi ja tallenna.

#### Poisto (Delete Preset)
1.  Valitse poistettava preset listasta.
2.  Paina punaista roskakorikuvaketta.
3.  Avautuu **varmistusmodaali** (punainen teema), joka kysyy oletko varma.
4.  Poisto palauttaa näkymän oletusarvoihin (Standard).

---

## 🔍 Suodatus, Haku ja Massapoisto (v5.3)

Sovelluksessa on edistyneet työkalut suurten ehdotusmäärien hallintaan:

1.  **Vapaatekstihaku (Älykäs haku):**
    -   Kirjoita hakukenttään mitä vain: Asiakkaan nimi, viikonpäivä ("Maanantai"), päivämäärä ("16.2.") tai lisätieto.
    -   Lista päivittyy reaaliajassa.

2.  **Suodattimet:**
    -   **Päivämäärä-valikko:** Valitse tarkat päivät listasta.
    -   **Tyyppirajaus (Checkboxit):** Voit piilottaa "Suunnittelu" tai "Teams" -kortit näkyvistä.

3.  **Massapoisto (Delete Visible):**
    -   Kun olet rajannut listan haulla (esim. kaikki "Maanantait"), hakupalkin yhteyteen ilmestyy punainen **"Poista näkyvät"** -painike.
    -   Tällä voit poistaa kerralla kaikki ruudulla näkyvät ehdotukset.

4.  **Luo merkinnät -logiikka:**
    -   "Luo merkinnät" -painike luo kalenteriin **vain näkyvissä olevat** kortit.
    -   Voit siis ensin hakea "Firma Oy", tarkistaa että listassa on vain heidän aikansa, ja painaa "Luo merkinnät".

---

## Miten vapaa aika löydetään? (Algoritmin ytimessä)

Aikataulutusmoottori toimii kolmessa vaiheessa. Tämä on se osa, joka tekee "magian" – tässä selitetään tarkasti miten.

### Vaihe 1: Tarpeen laskenta (`CalculateRequiredSlots`)

Ensin järjestelmä määrittää, mitä projekti vaatii:
- Montako slotia kustakin tyypistä (**Suunnittelu** ja **Teams-palaveri**)
- Kuinka pitkiä ne ovat (esim. 3 h suunnittelulle, 1 h Teamsille)
- Missä järjestyksessä ne tulevat (prioriteetti: suunnittelu ensin, sitten Teams)

### Vaihe 2: Vapaiden aikojen etsintä (`FindAvailableSlots`)

Tämä on logiikan sydän. Järjestelmä käy läpi päivät ja etsii sopivia aikoja.

1. **Työpäivät**  
   Vain sellaiset päivät huomioidaan, jotka on merkitty työpäiviksi (ma–su, asetuksista).

2. **Työaika**  
   Työajan ulkopuolelle ei ehdoteta aikoja (esim. 8:00–16:00).

3. **Varaukset**  
   Haetaan kaikki varaukset kyseiselle päivälle – sinun kaikista omista kalentereistasi (v5.8.3) sekä valmentajan kalenterista (jos valmentaja on määritelty). Varauksen **sijainti** (location) luetaan; jos päivällä on merkintä, jonka sijainti on "Helsinki", kyseinen päivä käsitellään erityisesti (Location Awareness, v5.6.2).

4. **Aukkojen etsintä**  
   Algoritmi etenee aikajanalla aina seuraavaan varaukseen asti:
   - **Tauko ennen varausta:** Jos seuraava varaus alkaa klo 11:00, vapaa aika päättyy 10:30 (30 min tauko oletuksena).
   - **Tauko varauksen jälkeen:** Varauksen (esim. 10:00–11:00) jälkeen seuraava vapaa aika voi alkaa vasta 11:30.
   - **Vähimmäispituus:** Jos aukko on alle 1 tunnin, sitä ei käytetä.

5. **Suositeltu aika**  
   Jos vapaa aika osuu "suositeltuun aikaan" (esim. 8:00–12:00), se merkitään prioriteettiseksi (`IsPreferred`). Nämä valitaan ensin.

### Vaihe 3: Ehdotusten luominen (`GenerateProposals`)

Lopuksi tarpeet ja vapaat ajat yhdistetään. **Tämä vaihe on päivitetty versiossa 5.6:**

Järjestelmässä on kaksi tapaa täyttää kalenteri, riippuen **"Maksimoi täyttöaste"** -valinnasta (ent. "Toista kaavaa"):

#### A. "Maksimoi täyttöaste" PÄÄLLÄ (Maximize Mode)
*Oletus: täyttää kalenterin tiiviisti (Tehopäivät).*
1.  **Logiikka:** Järjestelmä toistaa projektin kaavaa (Suunnittelu + Teams) niin monta kertaa kuin hakuvälille mahtuu.
2.  **Käyttötapaus:** "Haluan saada työn tehdyksi mahdollisimman nopeasti. Anna kaikki ajat, jotka sopivat."
3.  **Valinta (Acceptance):**
    -   **Kaikki setit:** Järjestelmä ehdottaa **kaikkia** löydettyjä aikoja listassa ("kirkkaina").
    -   **Käyttäjän hallinta:** Koska ehdotuksia voi tulla paljon (esim. 10 projektia), voit helposti karsia listaa:
        -   Poista yksittäisiä kortteja roskakorista.
        -   Käytä "Poista näkyvät" -toimintoa, jos haluat poistaa esim. kaikki tietyn päivän ajat kerralla.

#### B. "Maksimoi täyttöaste" POIS (Weekly Quota Mode)
*Tasainen/Stressitön: viikkokiintiö (Ma–Su).*
1.  **Logiikka:** Järjestelmä jakaa hakuvälin **kalenteriviikkoihin (Ma-Su)**.
2.  **Tavoite:** Se yrittää löytää *tasan* määritellyn määrän (esim. 4 suunnittelua) **joka viikolle**.
3.  **Käyttötapaus:** "Haluan tehdä tätä projektia tasaisesti 4 tuntia joka viikko seuraavan kuukauden ajan."
4.  **Valinta (Acceptance):**
    -   **Kaikki setit:** Koska pyysit nimenomaan viikottaista toistoa, järjestelmä valitsee oletuksena *kaikki* löydetyt ajat (yksi setti per viikko).

---

#### Prioriteettijärjestys (molemmat moodit):

3.  **Prioriteettijärjestys:**
    Kaavan sisällä välilyönnit täytetään tärkeysjärjestyksessä: ensin **Suunnittelu**, sitten **Teams-palaveri**.

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
| **Sijainnin tunnistus (v5.6.2)** | Jos kalenterissa on merkintä, jonka sijainti on "Helsinki", järjestelmä tulkitsee päivän mahdollisesti lähityöpäiväksi/matkustukseksi: Teams-palaverit estetään kyseiseltä päivältä, suunnitteluajoille näytetään varoitus |

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

## Älykäs Uudelleenjärjestely (Smart Rescheduling) (v5.9.0)

Kun kalenteri on täysi, "Älykäs haku" (checkbox: "Etsi myös siirrettävät ajat") auttaa löytämään tilaa siirtämällä vähemmän tärkeitä omia varauksia.

### 1. OR-Tools Optimointi (Backend)
Sovellus käyttää nyt **Google OR-Tools** -kirjastoa (CP-SAT solver), joka etsii matemaattisesti optimaalisen ratkaisun. Se yrittää:
1.  Löytää tilaa uudelle projektille.
2.  Minimoida muutosten määrän (mieluummin yksi siirto kuin viisi).
3.  Minimoida siirtomatkan (mieluummin siirto samalle päivälle kuin viikon päähän).

### 2. Mitä se ehdottaa siirrettäväksi?
Algoritmi koskee vain merkintöihin, jotka ovat turvallisia siirtää:
*   **Olet järjestäjä (IsOrganizer):** Vain omat varaukset.
*   **Ei muita osallistujia:** Vain yksilötyöaika (esim. "Suunnittelu", "Focus").
*   **Ei Teams-palaverit:** Online-kokouksia pidetään kiinteinä.
*   **Ei toistuvat sarjat:** Toistuvien sarjojen siirto on liian riskialtista.

### 3. Käyttäjän valinta (Swap)
Kun algoritmi löytää ratkaisun, se ei tee muutoksia automaattisesti, vaan **ehdottaa** niitä:
1.  **Ehdotus:** Listaan ilmestyy kortti, jossa lukee esim. *"Vapaa JOS siirrät: 'Suunnittelu (3/9)'"*.
2.  **OverlapModal:** Klikkaamalla "EHDOTUS", aukeaa modaali.
    *   **Vasen puoli:** Nykyinen varaus, joka on tiellä.
    *   **Oikea puoli (UUSI):** "Siirtoehdotukset". Näyttää mihin aikaan vanha varaus siirtyisi.
3.  **Suorita vaihto:** Painike "Suorita vaihto" tekee kaksi asiaa atomisesti:
    1.  **Poistaa** vanhan varauksen (vapauttaen ajan).
    2.  **Luo** uuden varauksen tilalle.

---

## Tietosuoja ja luetut tiedot (Privacy)

Jotta sovellus voi toimia älykkäästi, se lukee kalenteristasi seuraavat tiedot Microsoft Graph -rajapinnan kautta (`Calendar.ReadWrite`):

1.  **Ajankohta:** `start`, `end`, `timeZone` (Milloin varaus on)
2.  **Tila:** `showAs` (Onko aika varattu, vapaa, alustava vai "Out of Office")
3.  **Otsikko:** `subject` (Tarvitaan "Älykkäässä haussa" tunnistamaan omat merkinnät, kuten "Lounas")
4.  **Osallistujat:** `attendees` (Onko muita mukana? Jos on, emme ehdota siirtoa)
5.  **Järjestäjä:** `isOrganizer` (Oletko sinä luonut varauksen? Käytetään oma vs toisen -tunnistukseen)
6.  **Toistuvuus:** `recurrence`, `type` (Onko kyseessä toistuva sarja?)
7.  **Body (v5.8.2):** `body` (Kalenterimerkinnän kuvaus. Parsitaan Asiakas ja Lisätiedot, jotta toisen käyttäjän merkitsemät tiedot säilyvät.)

**Huomio:** Sovellus **EI** lue sähköpostien sisältöä tai liitetiedostoja. Kalenterimerkinnän `body`-teksti haetaan vain ulkoisten merkintöjen (esim. toisen käyttäjän) asiakas- ja lisätietojen näyttämiseksi – ei muuhun käyttöön. Kaikki tiedot käsitellään väliaikaisesti muistissa.

## Tietoturvallisuus

Sovellus käyttää useita turvatoimia API- ja käyttäjädatan suojaukseen:

| Suoja | Mitä vaikuttaa |
|-------|----------------|
| **Rate Limiting** | 100 req / 10 s per käyttäjä tai IP. OAuth callback: 10 req / min per IP (brute-force -suojaus). Estää DoS-hyökkäyksiä. |
| **JWT-autentikaatio** | Varmistaa, että vain kirjautuneet käyttäjät pääsevät API-endpointeihin. ValidateIssuer/Audience/Lifetime/SigningKey = true, ClockSkew 2 min, ValidAlgorithms = HS256 (estää alg:none). JWT-eventit loggaavat vain devissä. |
| **Middleware-autentikaatio (v6.0.0)** | Next.js middleware tarkistaa käyttäjäsession palvelintasolla (`supabase.auth.getUser()`). Suojatut reitit (`/calendar`) uudelleenohjaavat kirjautumissivulle ennen sivun renderöintiä. Ei luota pelkkään client-side -tarkistukseen. |
| **CORS** | Sallii pyynnöt vain määritellyistä alkuperistä (dev: localhost, prod: whitelist). Estää muut sivustot käyttämästä API:asi. |
| **Security Headers** | Backend: CSP, HSTS jne. Frontend: Middleware lisää CSP (connect-src Supabase + API), X-Content-Type-Options (`nosniff`), X-Frame-Options (`DENY`), Referrer-Policy, COOP/CORP. Vähentävät XSS-, clickjacking- ja MIME-sniffing -riskejä. |
| **HTTPS (tuotanto)** | Pakottaa salatun yhteyden tuotannossa. Suojaa datan man-in-the-middle -hyökkäyksiltä. |
| **OAuth state + CSRF** | Microsoft-kirjautumisen `state`-parametri on HMAC-SHA256-allekirjoitettu. Backend validoi state-arvon. Estää väärennetyt OAuth-callbackit. |
| **Token-salaus** | Microsoft Graph -tokenit salataan (ASP.NET Core Data Protection API, AES-256) ennen tietokantaan tallennusta. Suojaa, vaikka tietokanta vuotaisi. |
| **Input Validation (Backend)** | FluentValidation tarkistaa kaikki muokattavat DTO:t (v6.0.0): `CoachingProject`, `ProposedSlot`, `FindSlotsRequest`, `CreateEventsRequest`, `UpdateBookedEventRequest`, `UserPreferences`. Numeriset rajat, merkkijonojen pituudet, URL-formaatit. Estää virheellisen tai haitallisen datan. |
| **Input Validation (Frontend)** | Zod-skeemoja käytetään lomakkeissa ja API-pyynnöissä. DataAnnotations-attribuutit täydentävät backendin DTO-malleja. |
| **PII-maskaus** | Sähköposteja ei kirjoiteta logeihin selkokielisinä (maskataan tyyliin `m***@firma.fi`). Vähentää henkilötietojen vuotamista lokista. |
| **API-avainten suojaus** | Arkaluonteiset avaimet (JWT Secret, Microsoft Graph ClientSecret/ClientId/TenantId, Supabase Url) säilytetään backendin `.env`-tiedostossa, eivät koskaan repossa tai frontendissa. Frontend saa vain Supabasen anon key (suunniteltu julkiseksi; RLS rajaa oikeudet). `service_role` -avainta ei käytetä client-puolella. |
| **Snapshot-kansiosuojaus (v6.0.0)** | `.gitignore` estää IDE:n snapshot-kansioiden (`.lh/`, `.history/`) commitoinnin. Estää `.env`-kopioiden vahingollisen paljastumisen. |
| **Microsoft OAuth TenantId** | Käytä `TenantId=common` henkilökohtaisten Microsoft-tilien (outlook.com, hotmail.com) tukemiseksi. Org-spesifiselle käytölle käytä Azure AD:n tenant-ID:tä. |
| **Lokituksen turvallisuus** | OAuth- ja Graph API -virhevastauksia ei logiteta kokonaan (estää token- ja PII-vuodon). Logitetaan vain StatusCode ja ReasonPhrase. |
| **Frontend build-tarkistus** | `next.config.ts` tarkistaa build-vaiheessa, ettei `NEXT_PUBLIC_`-avaimissa ole `SERVICE_ROLE` tai `SECRET_KEY`. Epäonnistuu, jos yritetään paljastaa arkaluontoisia avaimia. |
| **postMessage origin-validoinnit** | Microsoft OAuth -popup lähettää `msauth-callback`-viestin frontendille. Frontend tarkistaa `event.origin` ja hyväksyy vain backendin OAuth-callback-origin (`NEXT_PUBLIC_API_URL`). Estää vihamielisten sivustojen viestit. Backend käyttää `Cors:AllowedOrigins` `targetOrigin`-parametrina (ei wildcardia `*`). |
| **Tuotantologitus (frontend)** | `console.error` rajoitettu kehitysympäristöön (`NODE_ENV === 'development'`). Tuotannossa virheitä ei logata selaimen konsoliin – vähentää token- tai PII-vuodon riskiä. |
| **Dependency Security** | Backend: `dotnet list package --vulnerable`. Frontend: `npm audit`. Suositus: Dependabot/Renovate automaattisille PR:ille, säännöllinen patchaus (viikoittain), kriittisten CVE:tien korjaus 48 h. |
| **SQL Injection -suojaus** | Entity Framework Core / LINQ käytössä kaikessa tietokantakäytössä. Ei raakaa SQL:ää käyttäjäsyötteillä. |

---

---


## Korjattavaa

1. **Kategorioiden mukaan priorisointi ( Puhelimessa kuva. Monalta jää oikea järjestys )**
| **Lokituksen turvallisuus** | OAuth- ja Graph API -virhevastauksia ei logiteta kokonaan (estää token- ja PII-vuodon). Logitetaan vain StatusCode ja ReasonPhrase. |
| **Frontend build-tarkistus** | `next.config.ts` tarkistaa build-vaiheessa, ettei `NEXT_PUBLIC_`-avaimissa ole `SERVICE_ROLE` tai `SECRET_KEY`. Epäonnistuu, jos yritetään paljastaa arkaluontoisia avaimia. |
| **postMessage origin-validoinnit** | Microsoft OAuth -popup lähettää `msauth-callback`-viestin frontendille. Frontend tarkistaa `event.origin` ja hyväksyy vain backendin OAuth-callback-origin (`NEXT_PUBLIC_API_URL`). Estää vihamielisten sivustojen viestit. Backend käyttää `Cors:AllowedOrigins` `targetOrigin`-parametrina (ei wildcardia `*`). |
| **Tuotantologitus (frontend)** | `console.error` rajoitettu kehitysympäristöön (`NODE_ENV === 'development'`). Tuotannossa virheitä ei logata selaimen konsoliin – vähentää token- tai PII-vuodon riskiä. |
| **Dependency Security** | Backend: `dotnet list package --vulnerable`. Frontend: `npm audit`. Suositus: Dependabot/Renovate automaattisille PR:ille, säännöllinen patchaus (viikoittain), kriittisten CVE:tien korjaus 48 h. |
| **SQL Injection -suojaus** | Entity Framework Core / LINQ käytössä kaikessa tietokantakäytössä. Ei raakaa SQL:ää käyttäjäsyötteillä. |

---

## Suoritetut Korjaukset

1. **Kalenteri merkinnät näkyviin 6kk ajalta**
2. **Suunnitelu aikataulut hakee klo 10 eteenpäin vain**
3. **Microsoft kirjautuminen ei toiminut, Osoite localhost:8080 API Calendar**
4. **Suunnittelu labelien värit vastaa, Valitse ajatboxien reunoja + Otsikot vastaa väriä.**
5. **Projektin laajuus customoituna. Suunnittelu / valmistelu = Molemmat suunnittelua. Omat tekstiboxit / Save / Load asetukset.**
6. **Slot Remainder Fix:** Jäännösajan palautus käyttöön.
7. **Varatut ajat -osio (v5.5.0):** Täysi hallinta varatuille ajoille:
    - **Haku:** Vapaatekstihaku (nimi, asiakas, pvm)
    - **Suodatus:** Checkboxit Suunnittelu/Teams ja Pvm-valitsin (MultiSelect)
    - **Poisto:** Yksittäinen poisto roskakorista ja massapoisto ("Poista näkyvät")
    - **Graph-integraatio:** Poisto poistaa merkinnän myös Microsoft-kalenterista

8. **Päällekkäisyyskorjaus (v5.5.1):**
    - **Aikavyöhyke:** Graph API palauttaa joskus UTC-aikoja (`Z`-pääte) huolimatta pyynnöstä. Järjestelmä tunnistaa nämä nyt ja muuntaa oikeaan Helsinki-aikaan.
    - **Vapaat ajat:** Kalenterimerkinnät, joiden tila on "Free", eivät enää estä ajanvarausta.

9. **Päällekkäisten aikojen ehdotus – sivutus ja hakuväli (v5.5.2):**
    - **Sivutus:** Microsoft Graph `calendarView` palauttaa tapahtumat sivuittain. Järjestelmä seuraa nyt `@odata.nextLink`-linkkiä ja hakee kaikki sivut (max 999 tapahtumaa/sivu). Aiemmin vain ensimmäinen sivu haettiin, jolloin osa varauksista jäi näkymättä ja syntyi päällekkäisiä ehdotuksia.
    - **Hakutapa:** Hakuvälin viimeinen päivä jäi aiemmin pois. Graph API:lle välitetään nyt `searchEnd.Date.AddDays(1)`, jotta viimeinen päivä tulee mukaan.

10. **Layoutin responsiivisuus (v5.5.3):**
    - **Komponentit:** `SlotList` ja `BookedTimesPanel` on päivitetty reagoimaan paremmin kapeaan tilaan. Korttien sisältö ja hakupalkit rivittyvät pystysuuntaan (stack) tarvittaessa, estäen sisällön leikkautumisen tai ylivuodon.

11. **Ulkoiset kalenterimerkinnät (v5.6.1):**
    - **Näkyvyys:** Sovellus hakee nyt myös Microsoft Kalenterin omat merkinnät (esim. Outlookissa tehdyt varaukset) ja näyttää ne harmaina palkkeina ("Muu varaus (Outlook)"). Tämä selkeyttää, miksi tietyt ajat eivät ole valittavissa.

12. **Sijainnin tunnistus ja Tehopäivät (v5.6.2):**
    - **Sijainnin tunnistus (Location Awareness):** Jos kalenterissa on merkintä, jonka sijaintina on "Helsinki", sovellus estää Teams-palaverit kyseiseltä päivältä (oletetaan lähityöpäiväksi/matkustukseksi). Suunnitteluajoille annetaan varoitus.
    - **Täyttöasteen valinta:** Käyttöliittymässä checkbox **"Maksimoi täyttöaste"** (ent. "Toista kaavaa"). Valinta ohjaa, täytetäänkö päivät tiiviisti (Tehopäivät) vai jaetaanko työkuorma tasaisesti viikoille (Tasainen/Stressitön).

13. **Muokkausmodaalin kentät – täysi synkronointi (v5.6.3):**
    - **Lisätiedot:** Kentän sisältö päivittyy nyt Microsoft-kalenterin body-kenttään, tietokantaan (CalendarEvent.Description) ja frontend-laatikkoon (kalenteri-/korttinäkymä). Aiemmin tallennus ei välittynyt API:lle eikä Graphiin.
    - **Asiakas:** Asiakas-kentän muokkaus päivittyy tietokantaan (CalendarEvent.ClientName), Graphin bodyyn ("Asiakas: ...") ja frontend-laatikkoon. Optimistic update + uudelleenlataus.
    - **Otsikko:** Asiakasnimi poistettu otsikosta – näkyy nyt vain "Suunnittelu (1/4)" tms.; asiakas omalla rivillään "Asiakas: ...".

14. **Sijainti-kenttä muokkausmodaaliin (v5.6.3):**
    - **Modaali:** Uusi kenttä "Sijainti" (MapPin-ikoni). Synkronoituu Microsoft Graphin `location.displayName` -kenttään ja tietokantaan (CalendarEvent.Location).
    - **Laatikot:** Sijainti näytetään ehdotus- ja varattujen slottien laatikossa vain jos kenttä ei ole tyhjä. Sijainnin tunnistus (Location Awareness, Helsinki-päivä) toimii edelleen samalla tavalla.

15. **Matkustuspäivävaroitus laatikon sisällä (v5.6.3):**
    - **Ehdotusslotti:** Kun ehdotus on matkustuspäivällä (sijainti Helsinki), laatikon sisällä näkyy varoituskolmio ja teksti "Mahdollinen matkustuspäivä" (amber-tyyli).
    - **Varattu slotti:** Kun merkinnän sijainti on "Helsinki", sama varoitus laatikon sisällä.
    - **Korttinäkymä:** Kortilla lyhyt teksti "Mahdollinen matkustuspäivä" varoituskolmion kanssa (täysi viesti tooltipissa).
      
---

## Yhteenveto

Lyhyesti: järjestelmä pilkkoo projektin tarvitsemiin aikoihin, etsii kalenterista vapaat "reiät" (huomioiden tauot ja työajat), sovittaa palaset paikoilleen ja suosii aamupäiväaikoja sekä valmennuspäivän läheisiä slotteja. OR-Tools optimointi auttaa löytämään tilaa täydestä kalenterista siirtämällä joustavia tehtäviä.

---

**Dokumentin versio:** 6.0.1 | **Päivitetty:** 2026-03-07 (Tietoturvaosio, middleware-auth, laajennettu backend-validointi, preset/projektityyppitaulukko korjattu)


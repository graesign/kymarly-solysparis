### kymarly-solysparis


# Projectopdracht: bouw je eigen advertentie-attributietool

**Werkvorm:** individueel, met wekelijkse code review

---

## 1. De situatie

Stel je runt een webshop. Je zet advertenties op Facebook en Instagram en je betaalt daar geld voor. De grote vraag die je elke week moet beantwoorden is simpel:

> "Ik heb vorige week €500 aan advertenties uitgegeven. Hoeveel omzet heeft dat opgeleverd, en welke advertentie werkte het best?"

Dat klinkt makkelijk, maar het isx het niet. Als iemand op een advertentie klikt en drie dagen later pas iets koopt, hoe weet je dan dat die bestelling bij die ene advertentie hoorde? En hoe weet je het verschil tussen advertentie A en advertentie B?

Het antwoord: je moet bij elke klik een **spoor** achterlaten, dat spoor **opslaan**, en het later **koppelen** aan de bestelling. Dat heet attributie, en dat ga jij bouwen.

Dit is geen verzonnen schoolopdracht. Dit is een versimpelde versie van een systeem dat echt bestaat en waar bedrijven maandelijks voor betalen.

---

## 2. Wat je gaat bouwen

Een webapplicatie met drie onderdelen:

**A. De tracker**
Een landingspagina die parameters uit de URL leest en opslaat in de database. Als iemand binnenkomt via
`jouwsite.nl/?click_id=abc123&campaign_id=550&ad_id=901`
dan sla jij die waardes op, samen met tijdstip en wat browser-info.

**B. De koppeling**
Een endpoint dat een bestelling ontvangt (mock-data, geen echte webshop) en die bestelling koppelt aan de eerder opgeslagen klik via het `click_id`.

**C. Het dashboard**
Een simpele pagina die per campagne laat zien: uitgaven, omzet, aantal bestellingen en de ROAS.

**D. Login en registratie**
Het dashboard toont bedrijfscijfers. Die mag niet iedereen zien. Je bouwt een registratiepagina, een loginpagina en een uitlogfunctie, en je zorgt dat het dashboard onbereikbaar is zonder geldige sessie. Hoofdstuk 7 gaat hier volledig over.

> **ROAS** = Return On Ad Spend = omzet ÷ advertentiekosten.
> €500 uitgegeven, €1500 omzet → ROAS 3.0. Alles onder 1.0 betekent verlies.

---

## 3. Leerdoelen

Aan het eind van dit project kun je:

1. Een relationele database ontwerpen met meerdere tabellen en relaties
2. Een Postgres-database opzetten, vullen en bevragen vanuit code
3. Geheimen (wachtwoorden, connectiestrings, API-keys) veilig beheren buiten je broncode
4. Een REST API bouwen die data ontvangt, valideert en opslaat
5. Een veilige login bouwen en uitleggen tegen welke aanvallen elke maatregel beschermt
6. Uitleggen waarom je bepaalde technische keuzes hebt gemaakt

Punt 3, 5 en 6 zijn waar de meeste mensen op vastlopen. Neem die serieus.

---

## 4. Technische eisen

Deze zijn niet onderhandelbaar. De rest mag je zelf invullen.

### Verplicht

- **Postgres** als database. Geen SQLite, geen JSON-bestand, geen Excel.
- **Environment variables** voor alle configuratie die per omgeving verschilt.
- **Geen enkel geheim in de broncode.** Niet in comments, niet "tijdelijk even", niet in een bestand dat je later wel opruimt.
- **Git** met een zinnige commit-historie. Kleine commits, duidelijke berichten.
- **README.md** waarin een vreemde jouw project kan opstarten zonder jou iets te vragen.
- **Werkende authenticatie.** Registratie, login, uitloggen, en een dashboard dat zonder geldige sessie niets prijsgeeft.
- **Alleen mock-data.** Je gaat geen echte webshop of echt advertentieaccount koppelen.

### Vrije keuze

Taal en framework mag je zelf kiezen, zolang je de eisen hierboven haalt. Suggesties:

| Stack | Waarom |
|---|---|
| Node.js + Express + `pg` | Weinig magie, je ziet precies wat er gebeurt |
| C# / .NET + Entity Framework | Als je dat op school al doet |
| Python + FastAPI + SQLAlchemy | Prettige documentatie |
| Next.js + `pg` | Alleen als je React al beheerst, zie hieronder |

Kies er één en blijf erbij. Halverwege switchen kost je een week.

### Mag ik Next.js gebruiken?

Ja, maar het is geen eis en het is niet automatisch de beste keuze.

**Kies het wel** als je React al redelijk beheerst. Dan bouw je iets wat lijkt op wat er in de praktijk draait, en dat is meegenomen.

**Kies het niet** als je React nog aan het leren bent. Je leert dan twee dingen tegelijk, en als je vastloopt weet je niet of het aan je database ligt of aan een hydration error. Dit project gaat over data en beveiliging, niet over frontend-frameworks.

### Verboden shortcuts

Ongeacht welke stack je kiest, deze drie dingen bouw je zelf:

1. **Geen kant-en-klare auth-bibliotheek.** Geen Auth.js/NextAuth, geen Clerk, geen Supabase Auth, geen ASP.NET Identity. Die doen precies het werk waar je van moet leren. Losse bouwstenen zoals `bcrypt` of `otplib` mag je natuurlijk wel gebruiken, zie hoofdstuk 7.
2. **Geen ORM in de eerste versie.** Geen Prisma, geen Entity Framework, geen SQLAlchemy-magie. Schrijf je queries als SQL. Als alles werkt mag je 'm alsnog omzetten naar een ORM, en dan zie je meteen wat zo'n tool voor je oplost.
3. **Geen gekopieerd boilerplate-project** waar login en database al in zitten.

Dit voelt onhandig, en dat klopt. Je bouwt hier expres de langzame route, omdat je later moet kunnen beoordelen of een bibliotheek zijn werk goed doet.

### Let op bij Next.js

Next.js draait code op de server én in de browser. Alles wat je in een client component zet, kan je bezoeker lezen. Een variabele met `NEXT_PUBLIC_` ervoor wordt letterlijk in de JavaScript-bundel gebakken die naar elke bezoeker gaat.

Zet daar dus nooit je `DATABASE_URL` of `SESSION_SECRET` in, ook niet als het "even niet werkt zonder". Als je database-connectie in een client component belandt, staat je wachtwoord in de broncode van je website. Open na afloop de devtools van je eigen site en zoek in de bundel naar je geheimen. Als je ze vindt, heb je een probleem.

---

## 5. Het datamodel

Dit is een startpunt, geen wet. Denk er zelf over na en wijk af als je een betere reden hebt.

```
visits
  id              serial primary key
  click_id        text not null unique
  campaign_id     text
  adset_id        text
  ad_id           text
  landing_page    text
  user_agent      text
  created_at      timestamptz not null default now()

orders
  id              serial primary key
  order_number    text not null unique
  total_amount    numeric(10,2) not null
  currency        text not null default 'EUR'
  click_id        text            -- verwijst naar visits.click_id, mag NULL zijn
  created_at      timestamptz not null default now()

ad_spend
  id              serial primary key
  date            date not null
  campaign_id     text not null
  campaign_name   text
  spend           numeric(10,2) not null
  impressions     integer
  clicks          integer

users
  id              serial primary key
  email           text not null unique
  password_hash   text not null
  totp_secret     text            -- alleen bij de 2FA-bonus, anders NULL
  totp_enabled    boolean not null default false
  created_at      timestamptz not null default now()
  last_login_at   timestamptz

sessions
  id              text primary key      -- lange willekeurige waarde, geen oplopend nummer
  user_id         integer not null references users(id) on delete cascade
  expires_at      timestamptz not null
  created_at      timestamptz not null default now()
```

**Denkvragen bij dit model:**

- Waarom mag `orders.click_id` leeg zijn? Wat betekent het als er geen klik bij een bestelling hoort?
- Wat gebeurt er als hetzelfde `click_id` twee keer binnenkomt? Wat *moet* er gebeuren?
- Waarom `numeric` en niet `float` voor bedragen? (Zoek dit op. Het antwoord is belangrijker dan je denkt.)
- Waarom `timestamptz` en niet `timestamp`?
- Waarom heet de kolom `password_hash` en niet `password`? Waarom is dat verschil belangrijk genoeg om in de naam te zetten?
- Waarom mag een sessie-id geen oplopend nummer zijn? Wat kan een aanvaller dan doen?

---

## 6. Beveiliging: geheimen buiten je code

Dit is het onderdeel waar je het meest van leert, dus hier de details.

### Het probleem

Je hebt een databasewachtwoord nodig om verbinding te maken. De makkelijkste oplossing is dat wachtwoord gewoon in je code zetten:

```js
const db = new Client({ password: "MijnWachtwoord123" });  // FOUT
```

Zodra je dit naar GitHub pusht, staat je wachtwoord publiek op internet. En het blijft daar staan, ook als je het later weghaalt — Git onthoudt alle oude versies. Er draaien bots die GitHub continu afzoeken op precies dit soort fouten.

### De oplossing

**1. Zet configuratie in een `.env`-bestand**

```
DATABASE_URL=postgresql://gebruiker:wachtwoord@localhost:5432/attributie
PORT=3000
SESSION_SECRET=lange-willekeurige-string-uit-een-generator
```

Die `SESSION_SECRET` verzin je niet zelf. Genereer 'm, bijvoorbeeld met `openssl rand -hex 32`. Mensen zijn slecht in willekeur.

**2. Zet `.env` in je `.gitignore`** — vóór je eerste commit, niet erna.

```
.env
node_modules/
```

**3. Maak een `.env.example` die je wél commit**

```
DATABASE_URL=postgresql://user:password@localhost:5432/dbnaam
PORT=3000
SESSION_SECRET=
```

Dit is documentatie: het vertelt de volgende ontwikkelaar welke variabelen hij moet invullen, zonder de waardes prijs te geven.

**4. Lees ze in bij het opstarten van je applicatie**

En laat je app *crashen* met een duidelijke foutmelding als een verplichte variabele ontbreekt. Beter meteen stuk dan halverwege een rare fout.

### Security-checklist

Loop deze af voordat je oplevert:

- [ ] `.env` staat in `.gitignore` en is nooit gecommit
- [ ] `.env.example` bestaat en is compleet
- [ ] Geen wachtwoorden, connectiestrings of tokens in de broncode
- [ ] Geen geheimen in `console.log` of logbestanden
- [ ] Alle database-queries gebruiken **parameters**, geen string-plakwerk (SQL-injectie)
- [ ] Het dashboard is afgeschermd met een login (zie hoofdstuk 7 voor de eisen daar)
- [ ] Input van buiten wordt gevalideerd voordat je 'm opslaat
- [ ] Je database-gebruiker heeft alleen de rechten die hij nodig heeft

**Extra opdracht:** commit één keer expres een nepwachtwoord, en zoek daarna uit hoe je het écht uit je Git-historie krijgt. Je zult merken dat dat vervelend is. Dat is precies de les.

---

## 7. Authenticatie: registratie en login

Je dashboard toont omzetcijfers van een bedrijf. Een login die "werkt" is niet genoeg, want bijna elke zelfgebouwde login werkt. De vraag is of hij ook standhoudt.

### Wat je minimaal bouwt

- Een registratiepagina (e-mailadres + wachtwoord)
- Een loginpagina
- Uitloggen, waarbij de sessie ook echt aan de serverkant ongeldig wordt
- Een dashboard dat zonder geldige sessie niets teruggeeft

Dat laatste punt is belangrijker dan het lijkt. Het is niet genoeg om de knop naar het dashboard te verbergen. Als iemand de URL van je data-endpoint intypt zonder in te loggen, moet hij een 401 krijgen en geen cijfers. Controleer dit op elk endpoint, niet alleen op de pagina.

### Wachtwoorden opslaan

Eén regel: **schrijf zelf geen crypto.** Gebruik `bcrypt` of `argon2`, in welke taal je ook werkt. Deze bibliotheken zijn jarenlang door specialisten getest en jouw versie is dat niet.

Wat je wel moet begrijpen:

**Hashen is niet versleutelen.** Versleutelen kun je terugdraaien, hashen niet. Bij het inloggen ontsleutel je het opgeslagen wachtwoord dus niet om te vergelijken. Je hasht wat de gebruiker net intypte en vergelijkt de twee hashes.

**Waarom is dat beter?** Denk aan het scenario waarin je database uitlekt. Bij versleuteling ligt de sleutel meestal in dezelfde applicatie, dus de aanvaller heeft alles. Bij hashing heeft hij niets direct bruikbaars.

**Een salt** is willekeurige data die per gebruiker bij het wachtwoord wordt gehasht. Zoek op waarom dat nodig is en wat *rainbow tables* zijn. `bcrypt` regelt de salt automatisch en stopt 'm in de hash-string, dus je hebt er geen aparte kolom voor nodig. Kijk een keer naar zo'n opgeslagen hash en zoek uit welk deel wat is.

**Bewust traag.** Bcrypt heeft een instelbare *cost factor*. Hoger betekent langzamer, en langzamer betekent dat een aanvaller minder wachtwoorden per seconde kan proberen. Zoek uit wat op dit moment een redelijke waarde is en waarom die door de jaren heen omhoog is gegaan.

### Wachtwoordbeleid

Het advies is de afgelopen jaren veranderd. De regel "minstens één hoofdletter, één cijfer en één leesteken" leidt in de praktijk tot `Welkom01!`, en dat is slecht. Lengte doet meer dan complexiteit.

Bepaal zelf je beleid, en kun je onderbouwen waarom. Denk na over: een minimumlengte, een maximum (waarom zou je die überhaupt hebben?), en of je bekende gelekte wachtwoorden wilt blokkeren.

### Sessies

Na een geslaagde login moet de gebruiker ingelogd blijven. Twee routes:

| Aanpak | Voordeel | Nadeel |
|---|---|---|
| Sessie in de database, id in een cookie | Je kunt een sessie direct intrekken | Elke request kost een database-query |
| JWT in een cookie | Geen opslag nodig | Je kunt 'm niet intrekken tot hij verloopt |

Voor dit project raad ik de eerste aan, want de `sessions`-tabel staat al in je model en uitloggen wordt dan echt uitloggen. Kies je JWT, leg dan uit hoe je omgaat met een gestolen token.

Je sessiecookie heeft in beide gevallen deze eigenschappen nodig. Zoek per stuk op wat hij doet:

- `HttpOnly`
- `Secure`
- `SameSite`
- een verloopdatum

### Denk zelf na: wat kan er misgaan?

Dit is de kern van de opdracht. Hieronder staan zes aanvalscenario's. Schrijf voor elk op **wat de aanvaller doet** en **hoe jouw code het tegenhoudt**, en zorg dat die maatregel er ook daadwerkelijk in zit.

1. Iemand probeert 10.000 wachtwoorden op het account `ishan@webshop.nl`. Wat merkt jouw applicatie daarvan, en wat doet ze?

2. Iemand voert bij registratie 500 willekeurige e-mailadressen in. Bij de bestaande adressen krijgt hij "dit account bestaat al", bij de rest niet. Wat heeft hij nu, en waarom is dat een probleem? (Zoek op: *user enumeration*)

3. Bij het inloggen krijgt hij bij een onbekend e-mailadres "gebruiker niet gevonden" en bij een fout wachtwoord "wachtwoord onjuist". Wat is daar mis mee?

4. Een aanvaller heeft je hele `users`-tabel gestolen. Hoeveel accounts kan hij openen, en waar hangt dat vanaf?

5. Iemand blijft ingelogd op een openbare computer. Wat gebeurt er als hij op uitloggen klikt? En als hij dat vergeet?

6. Iemand plakt `' OR 1=1 --` in het e-mailveld. Wat gebeurt er in jouw code, en waarom?

Werk deze zes uit in een apart bestand in je repo, bijvoorbeeld `SECURITY.md`. Dat is meteen je voorbereiding op de eindpresentatie.

### Bonus: tweestapsverificatie

Haal dit pas aan als alles hierboven af is. Het is een echte bonus, geen eis.

**Het idee.** Een wachtwoord is iets wat je *weet*. Als dat lekt, is de aanvaller binnen. Een tweede factor is iets wat je *hebt*, meestal je telefoon. Dan heeft een aanvaller aan een gelekt wachtwoord niets meer.

**Bouw TOTP**, de variant met een authenticator-app die elke 30 seconden een code van zes cijfers toont. Gebruik een bibliotheek zoals `otplib` of `speakeasy`. In grote lijnen:

1. Genereer bij het aanzetten een geheim per gebruiker en sla het op in `users.totp_secret`
2. Toon dat geheim als QR-code, zodat de gebruiker het in Google Authenticator of Bitwarden kan scannen
3. Laat de gebruiker één code invoeren om te bewijzen dat het scannen gelukt is, en zet dan pas `totp_enabled` op true
4. Vraag bij het inloggen na het wachtwoord om de code

**Denkvragen bij de bonus:**

- De code verandert elke 30 seconden. Wat als de klok van de telefoon een paar seconden afwijkt? Zoek op hoe een *time window* daarmee omgaat, en waarom je dat venster niet te groot maakt.
- Wat gebeurt er als iemand zijn telefoon verliest? Zoek op wat *recovery codes* zijn en hoe je die opslaat. Hint: net als wachtwoorden.
- Waarom wordt 2FA via sms als zwakker gezien dan een authenticator-app? (Zoek op: *SIM-swapping*)
- `totp_secret` staat als leesbare tekst in je database. Is dat erg? Wat zou je eraan kunnen doen, en wat kost dat je?

### Auth-checklist

- [ ] Wachtwoorden opgeslagen met bcrypt of argon2, nooit als platte tekst en nooit versleuteld
- [ ] Bij inloggen wordt vergeleken met de functie van de bibliotheek zelf, niet met `==` op twee hashes
- [ ] Foutmeldingen bij login verraden niet of een account bestaat
- [ ] Mislukte pogingen worden afgeremd of geteld
- [ ] Sessiecookie heeft `HttpOnly`, `Secure`, `SameSite` en een verloopdatum
- [ ] Uitloggen verwijdert de sessie aan de serverkant, niet alleen de cookie
- [ ] Elk data-endpoint controleert zelf de sessie
- [ ] Wachtwoorden verschijnen nergens in logs, ook niet bij een foutmelding
- [ ] `SECURITY.md` bestaat en beantwoordt de zes scenario's

---

## 8. Fasering

Werk van boven naar beneden. Ga niet naar de volgende fase voordat de vorige werkt.

**Fase 0 — Opzet (2–3 dagen)**
Git-repo, Postgres draaiend (lokaal via Docker of gratis bij Neon/Supabase), `.env` opgezet, applicatie start en maakt verbinding met de database. Meer niet. Als dit werkt, heb je de saaiste helft gehad.

**Fase 1 — Database (3–4 dagen)**
Tabellen aanmaken via een migratiescript (geen handmatig klikken in een tool). Bijlage B inladen in `ad_spend`. Seed-script dat `visits` en `orders` genereert volgens de regels in de bijlage.

**Fase 2 — Tracker (3–4 dagen)**
Endpoint of pagina die URL-parameters uitleest en als `visit` opslaat. Testen met de hand in je browser.

**Fase 3 — Orders (2–3 dagen)**
`POST /api/orders` die een bestelling ontvangt, valideert en opslaat inclusief `click_id`. Testen met Postman of `curl`.

**Fase 4 — Berekening (4–5 dagen)**
De kern. Een query die per campagne omzet en uitgaven optelt en de ROAS berekent. Dit is een `JOIN` plus `GROUP BY`. Schrijf 'm eerst met de hand in psql voordat je 'm in code zet.

**Fase 5 — Authenticatie (5–6 dagen)**
Registratie, login, uitloggen, sessies. Werk hoofdstuk 7 af inclusief `SECURITY.md`. Plan hier ruim tijd voor. Dit is het onderdeel waar je het langst op zult zitten en waar je het meeste van leert.

**Fase 6 — Dashboard (4–5 dagen)**
Een tabel met per campagne: naam, uitgaven, omzet, aantal bestellingen, ROAS. Kleur ROAS onder 1.0 rood. Alleen bereikbaar met een geldige sessie.

**Fase 7 — Afronden (2–3 dagen)**
README, beide checklists aflopen, code opruimen, voorbereiden op de presentatie. Eventueel de 2FA-bonus als je tijd overhoudt.

---

## 9. Definition of done

Je bent klaar als iemand anders jouw repo kan clonen, `.env.example` kan kopiëren naar `.env`, de instructies in je README kan volgen, en binnen tien minuten een werkend dashboard met data ziet.

Test dit echt. Zet je project op een andere computer neer, of laat iemand anders het proberen. Je zult verrast zijn wat er stukgaat.

---

## 10. Vragen die je moet kunnen beantwoorden

Bij de oplevering:

1. Waarom mag een wachtwoord niet in de broncode? Wat kan er misgaan?
2. Wat is het verschil tussen `.env` en `.env.example`, en waarom commit je er maar één?
3. Wat is SQL-injectie? Laat in je eigen code zien hoe je het voorkomt.
4. Waarom sla je een wachtwoord gehasht op in plaats van versleuteld?
5. Wat is een salt en welk probleem lost het op?
6. Laat je loginpagina zien. Hoe weet ik dat een account bestaat of niet? Als dat niet te zien is, hoe heb je dat opgelost?
7. Ik heb je sessiecookie gestolen. Hoe lang kan ik ermee doen wat ik wil, en wat kun jij daaraan doen?
8. Wat gebeurt er in jouw systeem met een bestelling zonder `click_id`? Waarom is die keuze verdedigbaar?
9. Je ROAS voor campagne X is 0.4. Wat zou je de webshop-eigenaar adviseren?

Vraag 9 is geen technische vraag. Dat is het punt. Je bouwt software voor iemand met een probleem, niet voor de code zelf.

---

## 11. Als je klaar bent en meer wilt

- Meerdere klikken vóór één bestelling: welke krijgt de eer? (Zoek op: *first-touch* vs *last-touch attributie*)
- Attributievenster: een klik van 90 dagen geleden telt niet meer mee. Bouw dat in.
- Grafiek van ROAS over tijd
- Automatische dagelijkse import van mock-uitgaven via een geplande taak
- Deploy naar een echte server met environment variables via de hosting-provider in plaats van een `.env`-bestand

---

## Bijlage: de mock-data

De webshop waar dit project op gebaseerd is, verkoopt Franse mode aan de Franse markt en adverteert uitsluitend op Meta. De bijgeleverde bestanden gebruiken die context: Parijse productnamen, Franse campagnestructuur, bedragen in euro's.

### Wat je krijgt

Deze vier bestanden staan in de map `bijlagen/`. Behandel ze als data die je van buiten binnenkrijgt, precies zoals dat in het echt zou gaan.

| Bestand | Wat het is | In het echt zou dit komen van |
|---|---|---|
| `bijlage-a-campagnestructuur.csv` | 4 campagnes, 7 adsets, 10 advertenties met hun ID's | Meta Marketing API |
| `bijlage-b-adspend.csv` | 120 regels: 30 dagen uitgaven per campagne | Meta Marketing API |
| `bijlage-c-producten.csv` | 27 producten met prijs en categorie | Je Shopify-catalogus |
| `bijlage-d-*` | Kleine testset met randgevallen plus het juiste antwoord | Jouw eigen testsuite |

`bijlage-b` gaat rechtstreeks je `ad_spend`-tabel in. Bij elkaar is dat € 6.244,80 aan advertentie-uitgaven over juli 2026.

### Wat je zelf bouwt

De `visits` en `orders` genereer je zelf, met een script. Niet met de hand, en niet gekopieerd. Dat script is onderdeel van de opdracht.

Regels waar je generator zich aan moet houden:

- **Trek campagne- en advertentie-ID's uit bijlage A.** Verzin ze niet, anders koppelt er straks niets.
- **Trek bedragen uit bijlage C.** Een bestelling bevat 1 tot 3 artikelen, dus tel prijzen op.
- **Aantal visits per campagne = het aantal clicks uit bijlage B.** Die twee moeten op elkaar aansluiten.
- **Conversie verschilt per campagne.** Gebruik ongeveer deze percentages, want die bepalen of je dashboard iets interessants laat zien:

  | Campagne | Conversie | Verwachte ROAS |
  |---|---|---|
  | PROSPECTING \| Broad | 0,75% | rond 0,7 (verlies) |
  | PROSPECTING \| Interesses Mode | 1,65% | rond 1,9 |
  | LOOKALIKE 1% | 1,87% | rond 2,6 |
  | RETARGETING \| ATC 14d | 3,01% | rond 5,5 |

- **Ongeveer 20% van de bestellingen heeft géén `click_id`.** Dat is direct verkeer, e-mail, of een bezoeker met een adblocker. Dit is geen fout in je data, dit is de kern van het probleem.
- **Bestellingen komen niet direct na de klik.** Verdeel de vertraging: ongeveer de helft binnen een dag, de rest verspreid over 1 tot 14 dagen.
- **Spreid alles over 1 tot 30 juli 2026**, gelijk aan de periode in bijlage B.

Als je het goed doet, kom je rond de 105 bestellingen uit en een blended MER van ongeveer 1,8.

### Waarom die verliesgevende campagne erin zit

`PROSPECTING | Broad` krijgt het grootste budget en levert het minst op. Dat is met opzet, en het gebeurt in het echt voortdurend: de campagne met de meeste uitgaven is niet de beste campagne.

Een dashboard waarop alles er goed uitziet, heb je niet getest. En je kunt er niets mee, want de hele reden dat iemand zo'n tool wil, is om te zien wáár het geld weglekt.

### Voordat je begint met genereren

Laad eerst bijlage D in een lege database en zorg dat je berekening daar het juiste antwoord op geeft. Die set is met de hand na te rekenen. Als je query daarop klopt, mag je hem loslaten op je grote gegenereerde dataset.

Andersom werkt niet. Op 6.000 willekeurige records zie je een fout in je `JOIN` gewoon niet.

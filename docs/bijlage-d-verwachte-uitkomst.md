# Bijlage D: gouden testset

## Waarom deze bijlage bestaat

De data die je zelf genereert is willekeurig. Daardoor kun je niet controleren of je ROAS-query klopt, want je weet het goede antwoord niet. Een query met een fout in de `JOIN` geeft gewoon een getal terug, en dat getal ziet er net zo geloofwaardig uit als het juiste.

Deze testset is zo klein dat je 'm met de hand kunt narekenen. Laad hem in een lege database, draai je query, en vergelijk. Wijkt het af, dan zit de fout in je query en niet in je data.

Bewaar dit als een echte test in je project. Elke keer dat je aan de berekening sleutelt, draai je hem opnieuw.

---

## De zes randgevallen

De testset zit vol met dingen die in het echt misgaan. Zoek ze eerst zelf op in de CSV's voordat je verder leest.

| # | Wat er aan de hand is | Waar |
|---|---|---|
| 1 | Normale bestelling: klik en aankoop binnen het uur | `clk_A1` → `SOL-1001` |
| 2 | Twee bestellingen uit dezelfde klik, twee dagen uit elkaar | `clk_A2` → `SOL-1002` en `SOL-1003` |
| 3 | Een klik die nooit tot een bestelling leidt | `clk_A3` en `clk_B2` |
| 4 | Dezelfde `click_id` komt twee keer binnen als visit | `clk_A2` staat er twee keer in |
| 5 | Bestelling zonder `click_id` (direct verkeer, of tracking geblokkeerd) | `SOL-1004` |
| 6 | Bestelling met een `click_id` die niet in `visits` bestaat | `SOL-1006` verwijst naar `clk_ZZ9` |

### Wat je met elk geval moet doen

**Geval 2** is een keuze, geen fout. Tel je beide bestellingen mee voor campagne A? De meeste attributietools doen dat wel. Kies iets en schrijf op waarom.

**Geval 4** raakt je database-ontwerp. Als je een `unique` staat op `visits.click_id`, dan mislukt de tweede insert. Is dat wat je wilt? Of wil je de eerste behouden en de tweede negeren? Of juist de laatste? Zoek op wat `ON CONFLICT DO NOTHING` doet in Postgres, en beslis. Je seed-script moet twee keer achter elkaar kunnen draaien zonder stuk te gaan.

**Geval 5** is geen randgeval maar dagelijkse kost. In een echte webshop heeft 20 tot 40 procent van de bestellingen geen bruikbare klik. Die omzet is echt, maar je kunt hem aan geen enkele campagne toewijzen. Hij mag dus nooit stilletjes bij een campagne worden opgeteld.

**Geval 6** is verraderlijk. Als je een `INNER JOIN` gebruikt, verdwijnt deze bestelling geruisloos uit je totalen. Je omzet klopt dan niet meer en je ziet nergens een foutmelding. Dit is precies het soort bug waar je pas weken later achter komt.

---

## De verwachte uitkomst

Reken dit eerst met de hand na voordat je je query draait.

### Per campagne

| Campagne | Uitgaven | Omzet | Bestellingen | ROAS |
|---|---|---|---|---|
| TEST Campagne A | € 100,00 | € 265,00 | 3 | 2,65 |
| TEST Campagne B | € 50,00 | € 200,00 | 1 | 4,00 |

Campagne A: 120 + 80 + 65 = 265
Campagne B: 200

### Niet toegewezen

| | Omzet | Bestellingen |
|---|---|---|
| Zonder `click_id` | € 45,00 | 1 |
| Onbekende `click_id` | € 55,00 | 1 |
| **Totaal niet toegewezen** | **€ 100,00** | **2** |

### Totalen

| | Waarde |
|---|---|
| Totale uitgaven | € 150,00 |
| Toegewezen omzet | € 365,00 |
| Totale omzet (alles) | € 465,00 |
| Toegewezen ROAS | 2,43 |
| Blended MER (alle omzet ÷ alle uitgaven) | 3,10 |

---

## De belangrijkste les uit deze cijfers

Je hebt twee verschillende getallen die allebei kloppen: **2,43** en **3,10**.

De eerste zegt: van elke euro advertentiegeld kan ik € 2,43 aan omzet aanwijzen. De tweede zegt: er komt € 3,10 aan omzet binnen per euro advertentiegeld, maar een deel daarvan kan ik niet aan een campagne koppelen.

Het verschil van € 100 zit in bestellingen zonder bruikbare klik. Die zijn niet verdwenen. Ze zijn alleen onherleidbaar.

Dit is het hele probleem waar dit project over gaat. Bouw je dashboard zo dat allebei de getallen zichtbaar zijn, plus het bedrag dat je niet kon toewijzen. Een dashboard dat alleen 2,43 laat zien, laat de eigenaar denken dat zijn advertenties slechter presteren dan ze doen.

**Vraag om over na te denken:** als het aandeel niet-toegewezen omzet van 20 procent naar 45 procent stijgt, wat is er dan waarschijnlijk gebeurd? Er zijn minstens drie verklaringen, en maar één daarvan gaat over je advertenties.

---

## Controlequery

Nadat je query werkt, controleer dan of deze drie altijd waar zijn:

```sql
-- 1. Elke bestelling wordt precies één keer geteld
SELECT
  (SELECT COUNT(*) FROM orders) AS totaal,
  (SELECT COUNT(*) FROM orders o JOIN visits v ON o.click_id = v.click_id) AS toegewezen,
  (SELECT COUNT(*) FROM orders WHERE click_id IS NULL) AS zonder_click;
-- toegewezen + zonder_click + wees-bestellingen moet gelijk zijn aan totaal

-- 2. Toegewezen omzet is nooit hoger dan totale omzet
-- 3. Er is geen campagne met omzet maar zonder uitgaven
```

Punt 1 gaat mis zodra je `visits`-tabel dubbele `click_id`-waardes bevat. Eén bestelling koppelt dan aan twee visits en telt dus dubbel in je `JOIN`. Dat is randgeval 4, en dit is waarom het ertoe doet.

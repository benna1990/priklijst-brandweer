# 🔥 Priklijst — Kookrooster Brandweer

Een lichtgewicht, installeerbare webapp (PWA) voor het bijhouden van het kookrooster op de brandweerkazerne. Wie kookt er volgende dienst? De app berekent het eerlijk op basis van aanwezigheid.

---

## Wat doet het?

- **Dienst-tab** — selecteer wie er in dienst is, de app bepaalt wie aan de beurt is om te koken
- **Ranglijst** — overzicht van alle collega's gesorteerd op beurtteller; toont gemiddelde maaltijdscore per kok als die feature aan staat
- **Historie** — overzicht van alle verwerkte diensten, bewerkbaar; beoordeel maaltijden met een cijfer 0–10
- **Wijzigingslog** — automatisch bijgehouden log van alle aanpassingen, zichtbaar via de LOG-knop in de History-footer
- **Beveiliging** — PIN-beveiliging voor bewerken en verwijderen
- **Sync** — alle data staat in Supabase, altijd actueel op alle apparaten
- **Offline-capable** — werkt ook zonder internet via serviceworker

---

## Technologie

| Onderdeel | Keuze |
|---|---|
| App | Één HTML-bestand — geen buildstap, geen framework |
| Database & sync | [Supabase](https://supabase.com) (gratis tier) |
| Hosting | [GitHub Pages](https://pages.github.com) (gratis) |
| PWA | Installeerbaar op iPhone en Android via "Voeg toe aan beginscherm" |

---

## Opzetten voor een nieuwe kazerne

### 1. Fork of kopieer deze repository

**Optie A (makkelijkst):** Klik op **Fork** rechtsboven op deze GitHub-pagina. Je krijgt een eigen kopie onder je eigen account.

**Optie B:** Download `index.html` uit deze repo, maak een nieuwe repo aan via [github.com/new](https://github.com/new) en upload het bestand daar.

### 2. Maak een Supabase-project aan

1. Ga naar [supabase.com](https://supabase.com) en maak een gratis account
2. Maak een nieuw project aan (kies een EU-regio, bijv. Frankfurt)
3. Ga naar **SQL Editor** en voer dit uit:

```sql
CREATE TABLE priklijst_state (
  id TEXT PRIMARY KEY,
  people JSONB DEFAULT '[]',
  history JSONB DEFAULT '[]',
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Voeg de beginrij in
INSERT INTO priklijst_state (id, people, history)
VALUES ('main', '[]', '[]');

-- Sta anonieme lees- en schrijftoegang toe (de app heeft geen login)
ALTER TABLE priklijst_state ENABLE ROW LEVEL SECURITY;

CREATE POLICY "public read" ON priklijst_state FOR SELECT USING (true);
CREATE POLICY "public write" ON priklijst_state FOR ALL USING (true);

-- Dataverlies-beveiliging: voorkom dat een oud apparaat met gecachte data
-- de history in Supabase verkort (bijv. door een verouderde lokale opslag)
CREATE OR REPLACE FUNCTION prevent_history_shrink()
RETURNS TRIGGER LANGUAGE plpgsql AS $$
BEGIN
  IF NEW.id = 'main'
     AND jsonb_typeof(OLD.history) = 'array'
     AND jsonb_typeof(NEW.history) = 'array'
     AND jsonb_array_length(NEW.history) < jsonb_array_length(OLD.history) THEN
    RAISE EXCEPTION 'history_shrink_blocked: % → % entries',
      jsonb_array_length(OLD.history), jsonb_array_length(NEW.history);
  END IF;
  RETURN NEW;
END;
$$;

CREATE TRIGGER trg_prevent_history_shrink
  BEFORE UPDATE ON priklijst_state
  FOR EACH ROW EXECUTE FUNCTION prevent_history_shrink();
```

4. Ga naar **Settings → API** en noteer:
   - **Project URL** (bijv. `https://xyzxyz.supabase.co`)
   - **anon public key** (lange JWT-string)

### 3. Configureer de app

Open `index.html` en pas deze twee regels aan (bovenin het `<script>`-blok):

```javascript
const SUPABASE_URL = 'https://JOUW-PROJECT-ID.supabase.co';
const SUPABASE_KEY = 'JOUW-ANON-KEY';
```

### 4. Maak de config-rij aan in Supabase

Voer dit eenmalig uit in de **SQL Editor** van je Supabase project:

```sql
-- Config: kazerne naam en PIN
INSERT INTO priklijst_state (id, people, history)
VALUES ('config', '{"kazerneNaam":"Jouw Kazerne","pin":"1234"}', '[]')
ON CONFLICT (id) DO UPDATE SET people = EXCLUDED.people;

-- Audit log rij
INSERT INTO priklijst_state (id, people, history)
VALUES ('audit', '[]', '[]')
ON CONFLICT (id) DO NOTHING;
```

> Vervang `"Jouw Kazerne"` door de naam van je kazerne. Deze verschijnt in de header van de app.

### 5. Zet GitHub Pages aan

1. Push `index.html` naar je repository als `index.html` in de `main`-branch
2. Ga naar **Settings → Pages**
3. Kies als source: `Deploy from a branch → main → / (root)`
4. Na ~1 minuut is de app live op `https://jouw-org.github.io/priklijst-jouw-kazerne`

### 6. Eerste keer opstarten

1. Open de app op je telefoon
2. Tik op ⚙️ rechtsonder
3. Voer de standaard-PIN in: **`1234`**
4. De app waarschuwt automatisch dat je de PIN moet wijzigen
5. Scroll naar **PIN wijzigen** → voer een nieuwe PIN in → **Opslaan**
6. Scroll naar **Kazerne naam** → vul je kazernenaam in → **Opslaan**
7. Tik **+ Persoon toevoegen** en voeg alle collega's toe

> **Tip:** Stel de begintellers in op basis van historische data, zodat niemand met een achterstand begint.
> De PIN en kazerne naam worden opgeslagen in Supabase en zijn direct geldig op alle apparaten.

---

## Instellingen (⚙️)

Alle instellingen zijn bereikbaar via het tandwiel-icoontje rechtsonder. Ze worden opgeslagen in Supabase en zijn direct actief op alle apparaten.

| Instelling | Uitleg |
|---|---|
| **Kazerne naam** | Verschijnt in de header van de app |
| **Kleur- en tekstwaarschuwingen** | Aan/uit — toont of de dienst-bezetting afwijkt van het bezettingsaantal |
| **Bezettingsaantal** | Het verwachte aantal personen per dienst (standaard 8). Bepaalt wanneer de kleurwaarschuwing rood/groen/geel kleurt |
| **Maaltijdbeoordeling (0–10)** | Aan/uit — voeg een cijfer toe aan elke gekookte maaltijd. Zichtbaar in Historie en als gemiddelde in de Ranglijst |
| **PIN wijzigen** | Minimaal 4 cijfers. Vereist voor bewerken, verwijderen en beheer |

---

## Maaltijdbeoordeling

Als de maaltijdbeoordeling aanstaat, verschijnt er een **☆-knop** rechts in elke history-entry.

1. Tik op **☆** bij een dienst
2. Kies een cijfer van **0 tot 10**
3. De score verschijnt als gekleurde badge (rood, amber, groen of goud)
4. In de **Ranglijst** zie je het gemiddelde per kok in de Score-kolom

Score wissen doe je via de "Score wissen"-knop onderaan de picker.

---

## PIN wijzigen

1. Open ⚙️ → scroll naar **PIN wijzigen**
2. Vul de nieuwe PIN in (minimaal 4 cijfers)
3. Tik **Opslaan**

> De PIN wordt opgeslagen in Supabase en is direct geldig op alle apparaten.

---

## Hoe werkt de beurtteller?

| Term | Betekenis |
|---|---|
| **Beurtteller** | Aantal diensten aanwezig geweest zonder te koken. Hoog = eerder aan de beurt |
| **Gekookt** | Totaal aantal keer gekookt (lifetime) |
| **Diensten** | Totaal aantal diensten aanwezig geweest |

De volgorde wordt bepaald door: beurtteller → minst gekookt → meeste diensten → langst geleden gekookt → loting.

---

## Bevelvoerders

Bevelvoerders (BV) staan in de lijst maar doen standaard **niet** mee in de kookroulatie. Ze kunnen per dienst handmatig op "kookt" gezet worden.

---

## AVG / Privacy

- Persoonsgegevens (namen, statistieken) staan **uitsluitend in Supabase** — niet in de broncode
- De GitHub-repository bevat geen namen
- Supabase staat in de EU (Frankfurt)
- Geen analytics, geen tracking, geen cookies van derden

---

## Beveiliging

- Diensten verwerken, bewerkingen en verwijderen: **vereist PIN**
- Basisgebruik (aanwezigheid bijhouden, ranglijst bekijken): **geen PIN**
- Data reist via HTTPS naar Supabase
- De anon-key is alleen-schrijven op de `priklijst_state`-tabel via Row Level Security

> **Let op:** de anon-key is zichtbaar in de HTML-broncode. Dit is bewust: de app heeft geen server-side laag. Bescherm de data via Supabase RLS-policies en niet via de key.

---

## Lokaal testen

Gewoon `index.html` openen in een browser werkt. Voor de Supabase-sync heb je een internetverbinding nodig.

---

## Licentie

MIT — vrij te gebruiken, aanpassen en verspreiden. Vermeld de oorsprong als je wil.

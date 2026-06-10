# 🔥 Priklijst — Kookrooster Brandweer

Een lichtgewicht, installeerbare webapp (PWA) voor het bijhouden van het kookrooster op de brandweerkazerne. Wie kookt er volgende dienst? De app berekent het eerlijk op basis van aanwezigheid.

---

## Wat doet het?

- **Dienst-tab** — selecteer wie er in dienst is, de app bepaalt wie aan de beurt is om te koken
- **Ranglijst** — overzicht van alle collega's gesorteerd op beurtteller
- **Historie** — overzicht van alle verwerkte diensten, bewerkbaar
- **Wijzigingslog** — automatisch bijgehouden log van alle aanpassingen (transparantie)
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

```bash
# Maak een nieuwe GitHub-repo aan en push de index.html
git clone https://github.com/jouw-org/priklijst-jouw-kazerne.git
cp index.html /pad/naar/jouw-repo/
```

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
```

4. Ga naar **Settings → API** en noteer:
   - **Project URL** (bijv. `https://xyzxyz.supabase.co`)
   - **anon public key** (lange JWT-string)

### 3. Configureer de app

Open `index.html` en pas deze twee regels aan:

```javascript
const SUPABASE_URL = 'https://JOUW-PROJECT-ID.supabase.co';
const SUPABASE_KEY = 'JOUW-ANON-KEY';
```

### 4. Zet GitHub Pages aan

1. Push `index.html` naar je repository als `index.html` in de `main`-branch
2. Ga naar **Settings → Pages**
3. Kies als source: `Deploy from a branch → main → / (root)`
4. Na ~1 minuut is de app live op `https://jouw-org.github.io/priklijst-jouw-kazerne`

### 5. Voeg collega's toe

1. Open de app
2. Tik op ⚙️ rechtsonder
3. Voer de PIN in (standaard: `1234` — **wijzig dit meteen**)
4. Tik **+ Persoon toevoegen**
5. Herhaal voor alle collega's

> **Tip:** Stel de begintellers in op basis van historische data, zodat niemand met een achterstand begint.

---

## PIN wijzigen

1. Open ⚙️ → scroll naar **PIN wijzigen**
2. Vul de nieuwe PIN in (minimaal 4 cijfers)
3. Tik **Opslaan**

> De PIN wordt lokaal opgeslagen per apparaat. Stel hem in op elk apparaat dat de app gebruikt.

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

---

*Gebouwd voor Brandweer Victor Noord-Holland. Vragen of bijdragen? Open een issue.*

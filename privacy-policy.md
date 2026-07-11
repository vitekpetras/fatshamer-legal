---
layout: page
title: Zásady ochrany osobních údajů
permalink: /privacy-policy/
---

# Zásady ochrany osobních údajů — Fatshamer

**Účinné od:** 2026-05-22
**Verze:** v1
**Aplikace:** Fatshamer (Android, package `cz.fatshamer.app`)
**Provozovatel:** Fyzická osoba — <!-- TODO: doplnit celé jméno a doručovací adresu před prvním Google Play release -->
**Kontakt pro ochranu údajů (GDPR):** <!-- TODO: doplnit kontaktní e‑mail (založen na Google před prvním Play release) -->

---

## 0. Pro koho jsou tyto zásady

Fatshamer je fitness aplikace s „shame‑loop" UX (notifikace před plánovaným tréninkem + zúčtování po něm). Aplikace je veřejně dostupná v EU a je určena uživatelům **od 13 let včetně** (Google Play hodnocení Teen). Pokud je ti méně než 13, aplikaci nepoužívej.

Tyto zásady popisují **přesně** jaká data Fatshamer sbírá, kam je posílá a co s nimi děláš ty. Není to obecný šablonový text — jsou napsány tak, abys mohl/a porozumět konkrétnímu fungování aplikace.

## 1. Stručný přehled (TL;DR)

- Přihlašuješ se přes **Google účet**. Od Google získáme jen tvé jméno, e‑mail a Google ID.
- Tvá data (rozvrhy, dokončené tréninky, hodnocení hlášek, vlastní hlášky) ukládáme na **Supabase** v **EU regionu**.
- Pády aplikace posíláme do **Sentry** (DE region).
- Anonymní událostní telemetrii (kdy přišla notifikace, jaký byl drift) posíláme do **PostHog** (EU host).
- **Neprodáváme tvá data.** Nepoužíváme reklamní SDK, neděláme remarketing.
- Můžeš si svá data kdykoli **stáhnout** (Nastavení → Profil → Exportovat moje data) <!-- TODO GPLAY_5: link export flow path -->.
- Můžeš svůj **účet kdykoli smazat** (Nastavení → Účet → Smazat účet).

## 2. Kdo je správcem údajů

Správcem osobních údajů ve smyslu nařízení **GDPR (EU) 2016/679** je provozovatel uvedený v hlavičce. Můžeš ho kontaktovat na e‑mailu uvedeném v hlavičce ve všech věcech ohledně tvých osobních údajů.

## 3. Jaká data sbíráme a proč

Záměrně rozlišujeme **3 vrstvy** zpracování: účet, provoz aplikace, technická telemetrie.

### 3.1 Účet (Supabase Auth + Google Sign‑in)

Aby ses mohl/a přihlásit a aby tvá data byla dostupná napříč zařízeními:

| Údaj | Zdroj | Kde se ukládá | Účel |
|---|---|---|---|
| Google ID (sub) | Google Identity Services | Supabase `auth.users` | Identifikace účtu |
| E‑mail | Google profil | Supabase `auth.users` + `public.users.email` | Identifikace, případná komunikace ohledně účtu |
| Jméno / přezdívka | Google profil (`name` / `full_name`) | Supabase `public.users.nickname` | Zobrazení v žebříčku, ve Stěně hanby, ve vlastních hláškách |
| Datum vytvoření účtu, datum posledního přihlášení | Server | Supabase `public.users.created_at`, `last_seen_at` | Hygiena účtu, leaderboard aktivních uživatelů |

Přihlášení probíhá přes **Google Identity Services / Credential Manager**. Fatshamer **nezíská tvé heslo** k Google účtu — dostane pouze tzv. *ID token* od Google, který Supabase ověří a vymění za session JWT.

**Právní základ:** plnění smlouvy o poskytnutí služby (čl. 6 odst. 1 písm. b) GDPR).

### 3.2 Provoz aplikace (Supabase Postgres, EU region)

Data, která vznikají tvým používáním aplikace a jsou ti pak k dispozici napříč zařízeními:

| Údaj | Tabulka | Co obsahuje |
|---|---|---|
| Tréninkové rozvrhy | `schedules` | Tvůj popis tréninku (label), dny v týdnu, čas (hh:mm), zapnuto/vypnuto |
| Plánované/dokončené tréninky | `workout_occurrences` | Plánovaný čas, status (dokončeno/zmeškáno), čas reakce |
| Hodnocení hlášek (palec nahoru / dolů) | `quote_ratings` | Tvůj user_id, ID hlášky, palec, čas |
| Vlastní vytvořené hlášky | `quotes` | Text, jazyk (cs/en), intenzita (1‑4), fáze (PRE/POST), autor, příznak anonymity, tagy, status moderace |
| Historie zobrazených hlášek | `quote_show_log` | Tvůj user_id, ID hlášky, čas, fáze, intenzita — slouží anti‑opakování v aplikaci a analytickým přehledům |
| Denní agregáty | `events_daily` | Počet notifikací, dokončených tréninků, hodnocených hlášek za den |
| Uživatelské předvolby | `public.users.preferences` (JSONB) | Tvůj zvolený avatar (emoji), preference jazyka hlášek, preference tagů |

Data jsou hostována na **Supabase** v **EU regionu** (Frankfurt nebo Dublin, viz odd. 5). Přístup k řádkům, které ti patří, je vynucován pomocí **Row Level Security** politik v Postgresu (`user_id = auth.uid()`).

**Právní základ:** plnění smlouvy (čl. 6 odst. 1 písm. b) GDPR).

### 3.3 Lokální data na zařízení (Room DB)

Aplikace si stejná data zrcadlí lokálně v **Room databázi** uvnitř soukromého úložiště aplikace (`/data/data/cz.fatshamer.app/databases/`):

- rozvrhy (`schedules`)
- výskyty tréninků (`workout_occurrences`)
- log doručených hlášek (`quote_delivery_log`) — slouží jen anti‑opakování notifikací
- cache hlášek + hodnocení + tagů

Tato data jsou přístupná **pouze této aplikaci**. Pokud aplikaci odinstaluješ, lokální data zmizí. Synchronizace na server probíhá automaticky a po `Nastavení → Synchronizovat`.

### 3.4 Notifikace

Aplikace plánuje **lokální notifikace** ve tvém zařízení podle rozvrhů (`schedules`). Plánování využívá Android API `AlarmManager` se systémovým oprávněním `SCHEDULE_EXACT_ALARM` a `USE_EXACT_ALARM` (Android 13+), aby notifikace dorazila s přesností v rámci minuty od zvoleného času. Žádná notifikace neopouští zařízení — vše se odehrává lokálně.

Aplikace **nepoužívá push notifikace** (FCM) v současné verzi.

### 3.5 Technická telemetrie

#### 3.5.1 Sentry — hlášení pádů (region DE)

Když aplikace spadne (uncaught exception, ANR), pošleme do **Sentry** (`@sentry/android`, datacentrum Frankfurt, DE):

- typ a stack trace chyby,
- verzi aplikace, model zařízení, verzi Androidu,
- *breadcrumbs* — posloupnost interních akcí aplikace v posledních minutách (např. „zobrazena obrazovka X", „spuštěn worker Y"),
- pseudonymizovaný identifikátor uživatele (Supabase user UUID), pokud jsi přihlášen/a,
- **NEsbíráme**: obsah obrazovky (screenshoty), klávesy, obsah tvých hlášek, e‑mail.

Aplikace **NEPOUŽÍVÁ** Sentry Session Replay (vypnuto v konfiguraci) ani performance tracing (`tracesSampleRate=0.0`).

**Právní základ:** oprávněný zájem na stabilitě aplikace (čl. 6 odst. 1 písm. f) GDPR).

#### 3.5.2 PostHog — produktová analytika (region EU)

Pro pochopení, jak spolehlivě fungují notifikace (kvůli driftu na různých výrobcích zařízení), posíláme do **PostHog Cloud EU** (`https://eu.i.posthog.com`, hostováno v Frankfurtu) jmenované doménové události:

| Událost | Co obsahuje |
|---|---|
| `notification_scheduled` | fáze (PRE/POST), ID rozvrhu, plánovaný čas, plánovaný odklad |
| `notification_fired` | fáze, ID rozvrhu, plánovaný čas, skutečný čas vystřelení, drift v sekundách |
| `notification_dropped` | fáze, ID rozvrhu, důvod zahození |
| `$identify` | mapuje anonymní instalaci na tvůj Supabase user UUID po přihlášení |

Aplikace **NEPOUŽÍVÁ**:
- PostHog Session Replay (vypnuto)
- PostHog Autocapture obrazovek (`captureScreenViews = false`)
- žádné události typu „uživatel klikl na X" mimo výše uvedené

Automaticky sbírané metadata (typ zařízení, verze OS, jazyk, anonymní install ID) jsou součástí PostHog SDK.

**Právní základ:** oprávněný zájem na zlepšování spolehlivosti aplikace (čl. 6 odst. 1 písm. f) GDPR). Možnost námitky viz odd. 8.

#### 3.5.3 Co NEDĚLÁME

Pro úplnost: **nepoužíváme** Google Analytics, Firebase Analytics, Crashlytics, Meta SDK, AppsFlyer, Adjust, Branch, AdMob, ani jiné reklamní/atribuční/marketingové SDK. **Neprodáváme data třetím stranám.** **Neprovozujeme cross‑app tracking.** **Nepoužíváme advertising ID.**

## 4. Příjemci dat (zpracovatelé)

| Příjemce | Účel | Region | Doklad |
|---|---|---|---|
| **Supabase, Inc.** (Delaware, USA — EU instance) | Hosting databáze a autentizace | EU (Frankfurt/Dublin) | DPA: https://supabase.com/legal/dpa |
| **Google LLC / Google Ireland Ltd.** | Sign‑In identity provider | EU (Irsko) | https://policies.google.com/privacy |
| **Functional Software, Inc. (Sentry)** | Crash reporting | EU (Frankfurt) | DPA: https://sentry.io/legal/dpa/ |
| **PostHog, Inc. (PostHog Cloud EU)** | Produktová analytika | EU (Frankfurt) | DPA: https://posthog.com/dpa |
| **Google LLC (Google Play)** | Distribuce aplikace, in‑app updates | EU + USA (SCC) | https://play.google.com/about/play-terms/ |

**Přenos do třetích zemí:** Supabase, Sentry i PostHog provozují EU instance, do kterých jsou tvá data ukládána. Mateřské společnosti sídlí v USA — případné incidentní administrativní přístupy (např. support) jsou pokryty **Standardními smluvními doložkami (SCC)** podle prováděcího rozhodnutí Komise (EU) 2021/914.

## 5. Doba uchovávání

| Data | Doba uchovávání |
|---|---|
| Účet a profil | Dokud existuje účet. Při smazání účtu (odd. 8) okamžitě. |
| Rozvrhy, výskyty tréninků, hodnocení, vlastní hlášky, historie zobrazení | Dokud existuje účet, nebo dokud je ručně nesmažeš v aplikaci. |
| Pády (Sentry) | 90 dní (retence Sentry projektu) |
| Telemetrie notifikací (PostHog) | 12 měsíců (retence PostHog projektu) |
| Lokální data v zařízení (Room) | Dokud aplikaci neodinstaluješ nebo se neodhlásíš |

Po smazání účtu zůstávají **agregované anonymizované záznamy** (např. `events_daily` s `user_id = NULL`, schválené komunitní hlášky s `is_anonymous = true`) — viz odd. 8.3.

## 6. Bezpečnost

- Veškerá komunikace mezi aplikací a serverem probíhá přes **HTTPS s TLS 1.2+**.
- Přístup k databázi je vynucován **Row Level Security** politikami v Postgresu — tvá data si může číst jen tvůj přihlášený session JWT.
- Service‑role klíče (admin přístup k databázi) **nikdy neopustí** server / Edge Function — Android klient je nemá.
- Lokální data v zařízení jsou v privátním sandboxu aplikace; další aplikace na ně bez root přístupu nedosáhnou.
- Aplikace **podporuje Auto Backup** (Android), ale citlivá data (Supabase session, OAuth tokeny) jsou z backupu **vyloučena** (`backup_rules.xml`).

## 7. Děti mladší 13 let

Fatshamer není určen dětem mladším 13 let. Pokud se dozvíme, že jsme získali účet od dítěte mladšího 13 let, **smažeme jej bez prodlení**. Pokud jsi rodič nebo zákonný zástupce a domníváš se, že tvé dítě má účet, kontaktuj nás na e‑mailu v hlavičce.

## 8. Tvá práva (GDPR čl. 15–22)

Máš následující práva, která můžeš **kdykoli a bezplatně** uplatnit:

### 8.1 Právo na přístup a přenositelnost (čl. 15, 20)

V aplikaci v **Nastavení → Profil → Exportovat moje data** <!-- TODO GPLAY_5: link path --> si můžeš stáhnout strojově čitelný JSON se všemi tvými daty (profil, rozvrhy, tréninky, hodnocení, vlastní hlášky, historie zobrazení).

### 8.2 Právo na opravu (čl. 16)

Přezdívku a předvolby si můžeš změnit přímo v Nastavení. Pro opravu e‑mailu (zděděný z Google) změň primární e‑mail ve svém Google účtu a znovu se přihlas.

### 8.3 Právo na výmaz (čl. 17)

V aplikaci v **Nastavení → Účet → Smazat účet** si můžeš okamžitě smazat účet (dvoustupňové potvrzení — napíšeš slovo `SMAZAT`). Co se stane:

- **Mazáno okamžitě**: tvůj Supabase `auth.users` záznam, profil v `public.users`, rozvrhy, tréninkové výskyty, neschválené koncepty vlastních hlášek.
- **Anonymizováno** (zachová se pro statistiky, ale bez vazby na tebe): hodnocení hlášek, denní agregáty, historie zobrazení hlášek, **schválené** komunitní hlášky (zobrazeny jako anonymní bez vazby na autora). To proto, abychom mohli i nadále poskytovat ostatním uživatelům komunitní obsah, který jsi schválil/a poskytnout pod přezdívkou nebo anonymně.
- **Lokální data** v zařízení jsou smazána okamžitě po potvrzení smazání serverem a vrátíš se na přihlašovací obrazovku. (Pokud smazání selže — např. bez připojení — zůstaneš přihlášen/a a nic se nesmaže, takže to můžeš zkusit znovu.)

Pokud chceš úplně smazat i schválené komunitní hlášky, napiš nám na e‑mail v hlavičce a smažeme je manuálně.

### 8.4 Právo na omezení a námitku (čl. 18, 21)

Můžeš vznést námitku proti zpracování na základě oprávněného zájmu (odd. 3.5 — Sentry, PostHog) zasláním e‑mailu na adresu v hlavičce. Vyřídíme do 30 dní; do té doby telemetrii pro tvůj účet zastavíme.

### 8.5 Stížnost u dozorového úřadu (čl. 77)

Máš právo podat stížnost u **Úřadu pro ochranu osobních údajů** (https://www.uoou.cz), Pplk. Sochora 27, 170 00 Praha 7.

## 9. Oprávnění Androidu

Aplikace si vyžaduje tato oprávnění:

| Oprávnění | K čemu |
|---|---|
| `INTERNET` | Komunikace se Supabase, Sentry, PostHog |
| `POST_NOTIFICATIONS` | Zobrazení notifikací o tréninku (Android 13+) |
| `RECEIVE_BOOT_COMPLETED` | Obnovení plánovaných notifikací po restartu telefonu |
| `SCHEDULE_EXACT_ALARM` | Plánování notifikací v přesný čas (Android 12+) |
| `USE_EXACT_ALARM` | Plánování notifikací v přesný čas bez uživatelského potvrzení (Android 13+) — důvod: rozvrh tréninků s konkrétním HH:MM zadaný uživatelem |
| `WAKE_LOCK` | Probuzení zařízení pro doručení notifikace |

Žádná z těchto permissions **nepřistupuje k osobním datům** (kontakty, kalendář, poloha, mikrofon, kamera, úložiště).

## 10. Změny zásad

Materiální změny ti oznámíme přímo v aplikaci (banner v Nastavení) minimálně **30 dní** před účinností. Historii verzí najdeš na <https://github.com/vitekpetras/fatshamer-legal/commits/main>.

## 11. Kontakt

E‑mail pro veškeré dotazy ohledně osobních údajů: **<!-- TODO: doplnit email -->**

Odpovídáme do 30 dní (typicky kratší).

---

*Tento dokument je psán v češtině jako primárně závazné verzi. V případě nesrovnalostí s případnou anglickou verzí má přednost česká.*

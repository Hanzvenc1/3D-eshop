# 🖨️ Specifikace: 3D Tiskový E-shop pro Školu (ScioEdu Intranet)

> **Repozitář předlohy:** https://github.com/PeMajer/scio-edu-intranet  
> **Stack:** Remix 2.x · TypeScript · Supabase (PostgreSQL + Auth) · Tailwind CSS v3 · shadcn/ui · Cloudflare Workers

---

## 1. Přehled projektu

Rozšíření stávajícího ScioEdu intranetu o modul **„3D Tisk"** — interní objednávkový systém, kde si zaměstnanci školy mohou objednat 3D tisk předmětů. Přihlášení zůstává přes Google OAuth (`@scioskola.cz`), nový modul se přidá jako sekce do existující navigace.

---

## 2. Hostingová omezení — Cloudflare Free + Supabase Free

Celý projekt musí běžet zdarma. Níže jsou klíčová omezení obou free tierů a jak se s nimi vypořádat.

### Cloudflare (Pages + Workers)

| Limit | Free tier | Dopad |
|-------|-----------|-------|
| Requests | 100 000 / den | Pro školní intranet dostatečné |
| CPU čas | **10 ms / request** | I/O (fetch na Supabase) se nezapočítává — dotazy jsou v pohodě; pozor na těžký SSR rendering |
| Paměť | 128 MB | Postačuje |
| Build minuty | 500 / měsíc | Postačuje |

**Povinný adapter:** Remix musí používat `@remix-run/cloudflare`, **ne** `@remix-run/node`. Node.js API (fs, crypto, …) na Workers nefunguje.

```typescript
// app/entry.server.tsx — musí importovat z cloudflare adapteru
import { createPagesFunctionHandler } from "@remix-run/cloudflare-pages";
```

Supabase JS klient v2 funguje na Workers nativně (používá `fetch`, ne Node HTTP).

### Supabase Free tier

| Limit | Hodnota | Dopad na projekt |
|-------|---------|-----------------|
| Databáze | 500 MB | Postačuje |
| Storage | **1 GB celkem** | ⚠️ Kritické — viz úprava níže |
| Bandwidth | 5 GB / měsíc | Postačuje |
| Aktivní projekty | 2 | OK |
| **Pauza projektu** | **po 7 dnech bez aktivity** | ⚠️ Kritické — viz workaround níže |
| Edge Functions | 500 000 invokací / měsíc | Postačuje |

### ⚠️ Úprava: limit uploadu STL/3MF souborů

Původní spec povoloval soubory do **50 MB**. Při 1 GB celkového storage by stačilo ~20 uploadů a storage je plný.

**Nový limit: 10 MB na soubor.** Většina STL pro malé předměty (stojánky, klíčenky, jmenovky) je do 10 MB. Upravit v:
- UI validaci (`_app.order.new.tsx`)
- Supabase Storage policy
- Popis v sekci 7 (formulář)

```typescript
// Validace v action funkci
if (file.size > 10 * 1024 * 1024) {
  return json({ error: "Soubor je příliš velký. Maximum je 10 MB." }, { status: 400 });
}
```

### ⚠️ Workaround: Supabase pauza projektu

Free tier pauzuje projekt po 7 dnech bez DB aktivity. První request po pauze trvá ~30 sekund (cold start).

**Řešení — Cloudflare Cron Trigger (zdarma):**

```toml
# wrangler.toml
[triggers]
crons = ["0 8 * * 1-5"]  # Každý pracovní den v 8:00 UTC
```

```typescript
// functions/cron.ts — jednoduchý ping aby projekt zůstal aktivní
export async function scheduled(event: ScheduledEvent, env: Env) {
  await fetch(`${env.SUPABASE_URL}/rest/v1/filament_colors?limit=1`, {
    headers: { apikey: env.SUPABASE_ANON_KEY }
  });
}
```

Cron se spouští jen v pracovní dny — přes víkend projekt může uspnout, ale v pondělí ráno ho probudí před příchodem zaměstnanců.

---

## 3. Role uživatelů

| Role    | Popis                                                                 |
|---------|-----------------------------------------------------------------------|
| `user`  | Přihlášený zaměstnanec — může zadávat a sledovat své objednávky       |
| `admin` | Správce tisku — vidí všechny objednávky, mění jejich stavy            |

Role se ukládá v Supabase ve vlastní tabulce `user_roles` (viz schéma níže).

---

## 4. Autorizace — omezení domény `@scioskola.cz`

Přístup do modulu (ani do celého intranetu) nesmí dostat nikdo mimo `@scioskola.cz`. Omezení musí být vynuceno na **třech vrstvách** — obejití jedné vrstvy nesmí stačit.

### Vrstva 1 — Supabase Auth (OAuth callback)

V Supabase Dashboard → Authentication → URL Configuration přidat do **Allowed email domains**: `scioskola.cz`.  
Supabase pak odmítne OAuth session pro jakoukoliv jinou doménu ještě před vznikem uživatele v `auth.users`.

> ⚠️ Toto nastavení existuje jen v Supabase Pro/Team. Pokud je projekt na Free tieru, tato vrstva chybí — tím důležitější jsou vrstvy 2 a 3.

### Vrstva 2 — Remix server-side (session helper)

V existujícím `app/lib/auth.server.ts` (nebo ekvivalentu) přidat helper `requireScioUser`, který se volá v každém `loader` a `action` chráněné route:

```typescript
// app/lib/auth.server.ts

const ALLOWED_DOMAIN = '@scioskola.cz';

export async function requireScioUser(request: Request) {
  const { supabase } = createServerClient(request);
  const { data: { session } } = await supabase.auth.getSession();

  if (!session) {
    throw redirect('/login');
  }

  if (!session.user.email?.endsWith(ALLOWED_DOMAIN)) {
    // Okamžitě odhlásit — zabrání opakovaným pokusům se stejnou session
    await supabase.auth.signOut();
    throw redirect('/login?error=unauthorized_domain');
  }

  return session.user;
}
```

Každý loader v `_app.tsx` (layout route) volá `requireScioUser` — tím je celý modul automaticky chráněn pro všechny vnořené routes.

### Vrstva 3 — Supabase RLS (defense-in-depth)

Doménová podmínka přidána do existujících politik jako dodatečná pojistka pro případ přímého volání Supabase API:

```sql
-- Přidat do VŠECH existujících RLS politik podmínku domény:
-- auth.jwt() ->> 'email' LIKE '%@scioskola.cz'

-- Příklad — upravená politika pro vkládání objednávek:
DROP POLICY IF EXISTS "user_insert_orders" ON orders;
CREATE POLICY "user_insert_orders" ON orders
  FOR INSERT WITH CHECK (
    auth.uid() = user_id
    AND (auth.jwt() ->> 'email') LIKE '%@scioskola.cz'
  );

-- Stejnou podmínku přidat do: user_own_orders, admin_select_orders, admin_update_orders
```

### Chybová stránka (`/login?error=unauthorized_domain`)

Na login stránce zobrazit srozumitelnou hlášku pokud URL obsahuje `error=unauthorized_domain`:

```
Přístup povolen pouze pro účty @scioskola.cz.
Přihlásili jste se jako: jan@example.com
```

---

## 5. Navigace (rozšíření stávajícího menu)

Do hlavní navigace přidej dvě nové položky (viditelné jen pro přihlášené):

```
Domů | ... stávající položky ... | 🖨️ Nová objednávka | 📋 Moje objednávky
```

Pro adminy navíc:

```
... | ⚙️ Admin — objednávky
```

---

## 6. Databázové schéma (Supabase / PostgreSQL)

### Tabulka `user_roles`
```sql
-- ⚠️ PŘIDÁNO: chyběla definice — RLS politiky na tuto tabulku přímo odkazují
CREATE TABLE user_roles (
  user_id uuid PRIMARY KEY REFERENCES auth.users(id) ON DELETE CASCADE,
  role    text NOT NULL CHECK (role IN ('user', 'admin'))
);

-- Pouze service_role může role přidělovat
ALTER TABLE user_roles ENABLE ROW LEVEL SECURITY;

CREATE POLICY "users_read_own_role" ON user_roles
  FOR SELECT USING (auth.uid() = user_id);
```

### Tabulka `filament_colors`
```sql
CREATE TABLE filament_colors (
  id            uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  name          text NOT NULL,
  hex_code      text NOT NULL,
  price_per_gram numeric(6,2) NOT NULL,
  is_active     boolean NOT NULL DEFAULT true  -- ✏️ PŘIDÁNO: umožňuje skrýt barvu bez mazání (archivace)
);
```

### Tabulka `preset_models`
```sql
CREATE TABLE preset_models (
  id           uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  name         text NOT NULL,
  description  text,
  image_url    text,
  weight_grams numeric(7,2) NOT NULL,
  print_hours  numeric(4,1) NOT NULL,
  base_price   numeric(7,2) NOT NULL,
  is_active    boolean NOT NULL DEFAULT true   -- ✏️ PŘIDÁNO: stejný důvod jako výše; smazat preset s historickými objednávkami nejde (FK)
);
```

### Tabulka `orders`
```sql
CREATE TABLE orders (
  id              uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  created_at      timestamptz DEFAULT now(),
  updated_at      timestamptz DEFAULT now(),   -- ✏️ PŘIDÁNO: bez toho nelze v admin panelu zobrazit "poslední změna stavu"
  user_id         uuid REFERENCES auth.users(id) ON DELETE CASCADE,
  customer_name   text NOT NULL,
  model_type      text NOT NULL CHECK (model_type IN ('preset', 'custom')),
  preset_model_id uuid REFERENCES preset_models(id),
  custom_model_description text,
  custom_file_url         text,                                            -- ⚠️ PŘIDÁNO: URL do Supabase Storage pro .stl/.3mf; bez sloupce nebylo kam odkaz uložit
  filament_color_id uuid REFERENCES filament_colors(id) NOT NULL,
  quantity        int NOT NULL DEFAULT 1 CHECK (quantity BETWEEN 1 AND 10),  -- ✏️ PŘIDÁNO: CHECK; UI má max 10, DB musí taky — jinak to lze obejít přímým voláním API
  total_price     numeric(8,2) NOT NULL,
  status          text NOT NULL DEFAULT 'Objednano'
                  CHECK (status IN ('Objednano','Zaplaceno','V tisku','Hotovo','Doruceno')),
  admin_note      text
);

-- ✏️ PŘIDÁNO: trigger pro automatickou aktualizaci updated_at
CREATE OR REPLACE FUNCTION update_updated_at()
RETURNS TRIGGER AS $$
BEGIN
  NEW.updated_at = now();
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER orders_updated_at
  BEFORE UPDATE ON orders
  FOR EACH ROW EXECUTE FUNCTION update_updated_at();
```

### ✏️ PŘIDÁNO: Indexy pro výkon
```sql
-- Loader "Moje objednávky" filtruje podle user_id při každém načtení stránky
CREATE INDEX idx_orders_user_id ON orders(user_id);

-- Admin panel filtruje a třídí podle stavu
CREATE INDEX idx_orders_status ON orders(status);

-- Řazení od nejnovější (výchozí v obou pohledech)
CREATE INDEX idx_orders_created_at ON orders(created_at DESC);

-- Loader v /order/new potřebuje jen aktivní barvy/presety
CREATE INDEX idx_filament_colors_active ON filament_colors(is_active) WHERE is_active = true;
CREATE INDEX idx_preset_models_active ON preset_models(is_active) WHERE is_active = true;
```

### Row Level Security (RLS)
```sql
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;

-- Uživatel vidí jen své objednávky
CREATE POLICY "user_own_orders" ON orders
  FOR SELECT USING (
    auth.uid() = user_id
    AND (auth.jwt() ->> 'email') LIKE '%@scioskola.cz'
  );

CREATE POLICY "user_insert_orders" ON orders
  FOR INSERT WITH CHECK (
    auth.uid() = user_id
    AND (auth.jwt() ->> 'email') LIKE '%@scioskola.cz'
  );

-- Admin vidí a upravuje vše
-- ✏️ OPRAVENO: původní spec měl FOR ALL pro admina i FOR SELECT/INSERT pro uživatele zároveň.
-- Postgres RLS kombinuje policies pro stejnou operaci pomocí OR — funkčně to sice projde,
-- ale pro přehlednost a bezpečnost auditů je lepší admin SELECT/UPDATE oddělit explicitně.
CREATE POLICY "admin_select_orders" ON orders
  FOR SELECT USING (
    (auth.jwt() ->> 'email') LIKE '%@scioskola.cz'
    AND EXISTS (SELECT 1 FROM user_roles WHERE user_id = auth.uid() AND role = 'admin')
  );

CREATE POLICY "admin_update_orders" ON orders
  FOR UPDATE USING (
    (auth.jwt() ->> 'email') LIKE '%@scioskola.cz'
    AND EXISTS (SELECT 1 FROM user_roles WHERE user_id = auth.uid() AND role = 'admin')
  );
```

---

## 7. Stránky a routes (Remix file-based routing)

```
app/
  routes/
    _auth.login.tsx              ← stávající přihlášení (nezměněno)
    _app.tsx                     ← layout s navigací (přidat nové menu položky)
    _app.order.new.tsx           ← Nová objednávka (formulář)
    _app.order.my.tsx            ← Moje objednávky (seznam + stavy)
    _app.admin.orders.tsx        ← Admin panel (všechny objednávky)
    _app.admin.orders.$id.tsx    ← Detail objednávky (admin — změna stavu)
```

---

## 8. Stránka: Nová objednávka (`/order/new`)

### 8.1 Formulář — sekce v pořadí

**① Jméno objednatele**
- Textové pole `customer_name` (povinné)
- Defaultně předvyplněno jménem z Google profilu uživatele

**② Výběr modelu**  
Přepínač (tabs / radio group):

- **„Populární předměty"** — grid karet s náhledem, názvem, popisem a odhadovanou cenou. Karta se kliknutím označí. Loader načítá jen záznamy kde `is_active = true`.
- **„Vlastní model"** — textarea pro popis vlastního předmětu + volitelně upload `.stl` / `.3mf` souboru (max **10 MB**, viz omezení Supabase Storage v sekci 2) do Supabase Storage

**③ Barva filamentu**
- Horizontální seznam barevných koleček (color swatches) generovaný z `filament_colors WHERE is_active = true`
- Po výběru se zobrazí název barvy

**④ Počet kusů**
- Number input (min 1, max 10)

**⑤ Kalkulace ceny (live preview)**
```
Základ tisku:          XX,- Kč
Filament (XX g × Kč/g): XX,- Kč  
Počet kusů × N:        XX,- Kč
─────────────────────────────
Celkem:               XXX,- Kč
```
Cena se přepočítává v reálném čase při změně jakéhokoliv vstupního pole.

**⑥ Tlačítko „Odeslat objednávku"**
- Uloží objednávku do Supabase s výchozím stavem `Objednano`
- **✏️ PŘIDÁNO:** `total_price` se **přepočítá server-side v Remix `action`** a teprve pak uloží — nikdy se nespoléhej na hodnotu poslanou klientem. Uživatel by jinak mohl odeslat libovolnou cenu přímým POST requestem.
- Přesměruje na `/order/my` s flash notifikací „Objednávka byla úspěšně odeslána ✓"

---

## 9. Stránka: Moje objednávky (`/order/my`)

Tabulka / seznam karet s objednávkami aktuálního uživatele, seřazeno od nejnovější.

| Sloupec | Popis |
|---------|-------|
| Datum | `created_at` formátovaný jako `DD. MM. YYYY` |
| Jméno | `customer_name` |
| Předmět | název presetu nebo „Vlastní model" |
| Barva | barevné kolečko + název |
| Ks | počet kusů |
| Cena | `total_price` Kč |
| Stav | badge s barvou podle stavu (viz níže) |

### Barevné odznaky stavů

| Stav | Barva odznaku |
|------|---------------|
| Objednano | šedá |
| Zaplaceno | modrá |
| V tisku | žlutá/oranžová |
| Hotovo | zelená |
| Doruceno | tmavě zelená |

---

## 10. Admin panel (`/admin/orders`)

Viditelný pouze pro uživatele s rolí `admin`.

### Hlavní tabulka (všechny objednávky)
Stejné sloupce jako „Moje objednávky" + navíc:
- **E-mail uživatele** — ⚠️ `auth.users` není přístupná z aplikačních queries přímo. Je potřeba vytvořit DB view nebo RPC funkci se `SECURITY DEFINER`, která e-mail bezpečně vystaví:
```sql
-- supabase/migrations/..._user_email_view.sql
CREATE VIEW public.user_emails AS
  SELECT id, email FROM auth.users;

-- Přístup jen pro přihlášené (admin policy zajistí RLS na orders)
GRANT SELECT ON public.user_emails TO authenticated;
```
  Loader pak joinuje `user_emails` místo přímého přístupu na `auth.users`.
- **Poslední změna** — `updated_at` formátovaný jako `DD. MM. YYYY HH:mm`
- **Akce** — tlačítko „Změnit stav" → otevře modal nebo inline select

### Změna stavu
- Select/dropdown se všemi 5 stavy
- Volitelná admin poznámka
- Uložení přes Remix action → Supabase UPDATE (trigger automaticky nastaví `updated_at`)

### Filtrování a vyhledávání
- Filtr podle stavu (checkbox skupina)
- Vyhledávání podle jména nebo e-mailu

---

## 11. Výpočet ceny — logika

```typescript
// utils/price.ts

export function calculatePrice(
  model: PresetModel | null,   // ⚠️ OPRAVENO: odstraněn redundantní parametr `customModel: boolean`
  filamentColor: FilamentColor, //   — null model jednoznačně znamená vlastní model
  quantity: number
): number {
  const BASE_CUSTOM_PRICE = 50;
  const BASE_PRINT_HOURS_RATE = 20;

  let basePrice: number;
  let weightGrams: number;

  if (model) {
    basePrice = model.base_price + model.print_hours * BASE_PRINT_HOURS_RATE;
    weightGrams = model.weight_grams;
  } else {
    basePrice = BASE_CUSTOM_PRICE;
    weightGrams = 50;
  }

  const filamentCost = weightGrams * filamentColor.price_per_gram;
  return (basePrice + filamentCost) * quantity;
}
```

> **✏️ POZNÁMKA:** Tato funkce musí být volána v Remix `action` server-side při ukládání objednávky. Výsledek se uloží jako `total_price` — hodnota z formuláře se ignoruje.

---

## 12. Ukázková seed data

### Barvy filamentů
```sql
INSERT INTO filament_colors (name, hex_code, price_per_gram) VALUES
  ('Bílá',    '#FFFFFF', 0.80),
  ('Černá',   '#222222', 0.80),
  ('Červená', '#E53E3E', 0.90),
  ('Modrá',   '#3182CE', 0.90),
  ('Zelená',  '#38A169', 0.90),
  ('Žlutá',   '#D69E2E', 0.90),
  ('Oranžová','#DD6B20', 0.95),
  ('Průhledná','#E8F4FD',1.10);
```

### Populární předměty
```sql
INSERT INTO preset_models (name, description, weight_grams, print_hours, base_price) VALUES
  ('Stojánek na tužky',    'Jednoduchý válcový stojánek.',        80,  3.0, 60),
  ('Jmenovka na dveře',    'Plochá jmenovka 10×5 cm.',            25,  1.5, 30),
  ('Organizér na kabely',  'Klip pro správu kabelů.',             15,  1.0, 25),
  ('Mini flowerpot',       'Malý květináček pro sukulenty.',      120,  4.5, 80),
  ('Držák na tablet',      'Nastavitelný stojan na tablet.',      200,  7.0,140),
  ('Klíčenka se jménem',   'Vlastní text, tl. 3 mm.',             10,  0.5, 20);
```

---

## 13. Technické požadavky a konvence

- Dodržuj stávající konvence z `doc/conventions.md` (shadcn/ui, Tailwind, Remix patterns)
- Používej existující layout a navigační komponenty — přidej jen nové menu položky
- Formuláře přes Remix `action` funkce (server-side validation pomocí Zod)
- Používej `useLoaderData` / `useFetcher` stejně jako zbytek projektu
- Stavy objednávek jsou `const` enum v `app/types/order.ts`
- Veškerá komunikace se Supabase přes existující `app/lib/supabase.server.ts`
- **`total_price` se vždy počítá server-side** pomocí `calculatePrice()` z `utils/price.ts` — hodnota z klienta se zahazuje
- Nové shadcn komponenty: `Badge`, `Select`, `Tabs`, `Dialog`
- Migrace přidat do `supabase/migrations/` se správným timestamp prefixem

---

## 14. Checklist implementace (doporučené pořadí)

- [ ] Ověřit `@remix-run/cloudflare` adapter v `entry.server.tsx` (ne Node adapter)
- [ ] Cloudflare Cron Trigger v `wrangler.toml` — ping Supabase každý pracovní den v 8:00
- [ ] Supabase migrace: `user_roles` + `filament_colors`, `preset_models`, `orders` + RLS (včetně domény) + indexy + trigger
- [ ] Seed data: barvy a populární předměty
- [ ] Typ soubor `app/types/order.ts` (OrderStatus enum, typy)
- [ ] `app/lib/auth.server.ts` — helper `requireScioUser` s kontrolou domény `@scioskola.cz`
- [ ] `app/lib/orders.server.ts` — CRUD funkce pro objednávky
- [ ] `utils/price.ts` — cenová kalkulace (sdílená mezi klientem i serverem)
- [ ] Route `_app.order.new.tsx` — formulář + loader + action (server-side přepočet ceny)
- [ ] Route `_app.order.my.tsx` — seznam vlastních objednávek
- [ ] Route `_app.admin.orders.tsx` — admin tabulka + filtrování + `updated_at`
- [ ] Route `_app.admin.orders.$id.tsx` — změna stavu + admin poznámka
- [ ] Rozšíření navigace v `_app.tsx` o nové položky
- [ ] Role guard middleware/helper pro admin routes

---

## 15. Připravený PROMPT pro Claude Code / AI asistenta

```
Kontext: Pracuji na projektu https://github.com/PeMajer/scio-edu-intranet — 
interní portál pro ScioŠkoly. Stack: Remix 2.x + TypeScript + Supabase + 
Tailwind CSS v3 + shadcn/ui + Cloudflare Workers.

Úkol: Přidej do projektu modul "3D Tisk" — objednávkový systém pro 3D tisk.
Kompletní specifikace je v přiloženém souboru. Začni migrací databáze, pak typy,
pak utility, pak routes v pořadí: new → my → admin.

Klíčové bezpečnostní pravidlo: total_price se vždy přepočítává server-side
v Remix action pomocí calculatePrice() — hodnota z formuláře se ignoruje.

Autorizace: přístup mají pouze uživatelé s emailem @scioskola.cz — vynuceno
na třech vrstvách: Supabase Auth (Allowed domains), Remix requireScioUser()
helper v _app.tsx loaderu, a RLS podmínka v každé policy.
```

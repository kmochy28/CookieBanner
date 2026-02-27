# CookieBanner

Cookie consent banner s integrací **Google Consent Mode v2**, navržený pro nasazení přes **Google Tag Manager (GTM)**.

## Funkce

- **Google Consent Mode v2** — automaticky nastavuje `consent default` a `consent update` pro všechny typy souhlasu (analytics_storage, ad_storage, ad_user_data, ad_personalization, personalization_storage, functionality_storage, security_storage)
- **4 kategorie cookies** — Nezbytné (vždy zapnuté), Analytické, Personalizační, Marketingové
- **Vícejazyčná podpora** — CZ, EN a SK out of the box, snadno rozšiřitelné o další jazyky
- **Automatická detekce jazyka** — podle nastavení prohlížeče
- **Accordion UI** — vždy max jedna kategorie rozbalená
- **Responsivní design** — na mobilu se zobrazí jako bottom sheet
- **Přístupnost (a11y)** — ARIA atributy, focus trap, klávesová navigace, screen reader podpora
- **XSS ochrana** — DOMParser-based HTML sanitizer s whitelistem tagů
- **Bezpečné cookies** — `SameSite=Lax; Secure` flags
- **dataLayer eventy** — `cookiebarConsentInit` a `cookiebarConsentUpdate` pro GTM triggery
- **GTM preview kompatibilita** — automatický cleanup při opakované inicializaci

## Nasazení do GTM

### 1. Vytvoření Custom HTML tagu

1. V GTM vytvořte nový tag typu **Custom HTML**
2. Zkopírujte celý obsah souboru `modal-cookiebar-v2.html` do tagu
3. Trigger: **Consent Initialization - All Pages** (důležité pro Consent Mode v2 — musí se spustit PŘED všemi ostatními tagy)

### 2. Konfigurace

V dolní části skriptu najdete objekt `options`. Upravte podle svých potřeb:

```javascript
var options = {
    host: "",                    // viz sekce "Nastavení host" níže
    version: "2.0.0",           // při změně se vynutí nový souhlas
    dismissible: true,          // false = uživatel MUSÍ zvolit (nelze zavřít křížkem/Escape)
    multilingual: true,         // true = zobrazí dropdown pro výběr jazyka
    langAutoSelect: true,       // true = automaticky vybere jazyk podle prohlížeče
    defaultLang: "cs",          // výchozí jazyk, pokud je autodetekce vypnutá nebo selže
    sendToServer: false,        // true = odesílá souhlas na server (vyžaduje apiKey a consentEndpoint)
    logLevel: "INFO",           // NONE, ERROR, WARN, INFO, DEBUG
    // ...
};
```

### 3. Nastavení `host`

Parametr `host` nastavuje atribut `domain=` na consent cookie.

| Scénář | Hodnota `host` | Příklad |
|--------|----------------|---------|
| Web na jedné doméně | `""` (prázdný) | `www.example.cz` — cookie platí jen pro tuto doménu |
| Web se subdoménami | `".example.cz"` | Cookie platí pro `www.example.cz`, `shop.example.cz`, `blog.example.cz` atd. |

**Kdy nastavit:**
- Pokud máte e-shop na `shop.example.cz` a blog na `blog.example.cz` a chcete, aby souhlas platil pro obě — nastavte `host: ".example.cz"`
- Pokud máte web jen na jedné doméně — nechte prázdný

**Tečka na začátku je důležitá** — `.example.cz` pokryje všechny subdomény, `example.cz` bez tečky nemusí fungovat ve všech prohlížečích.

### 4. Nastavení odkazu na zásady ochrany osobních údajů

V sekci `texts` nahraďte placeholder URL skutečnými adresami:

```javascript
// CZ verze
intro: '... <a href="/zasady-ochrany-osobnich-udaju">Zásadách ochrany osobních údajů</a>.'

// EN verze
intro: '... <a href="/privacy-policy">Privacy Policy</a>.'
```

### 5. Nastavení verzování

Pokud potřebujete vynutit nový souhlas od všech uživatelů (např. po změně kategorií nebo právních podmínek), zvyšte číslo `version`:

```javascript
version: "2.1.0",  // všechny existující souhlasy budou zneplatněny
```

## Konfigurace kategorií

```javascript
categoryConfig: {
    "strictly-necessary": { required: true, defaultEnabled: true },
    "analytical":         { required: false, defaultEnabled: false },
    "preferences":        { required: false, defaultEnabled: false },
    "marketing":          { required: false, defaultEnabled: false }
}
```

| Parametr | Popis |
|----------|-------|
| `required: true` | Kategorii nelze vypnout, přepínač je skrytý |
| `defaultEnabled: true` | Předvolena při prvním zobrazení (pozor na GDPR — volitelné kategorie by měly být `false`) |

## Google Consent Mode v2

### Mapování kategorií na consent typy

| Kategorie banneru | Google Consent typy |
|-------------------|---------------------|
| `strictly-necessary` | `functionality_storage`, `security_storage` |
| `analytical` | `analytics_storage` |
| `preferences` | `personalization_storage` |
| `marketing` | `ad_storage`, `ad_user_data`, `ad_personalization` |

### Jak to funguje

1. **Při načtení stránky** — banner okamžitě pošle `gtag("consent", "default", ...)` s výchozími hodnotami (`denied` pro vše kromě nezbytných)
2. **Po volbě uživatele** — pošle `gtag("consent", "update", ...)` s aktualizovanými hodnotami
3. Google tagy (GA4, Google Ads, Floodlight) automaticky reagují na tyto signály

### Omezení na regiony (volitelné)

```javascript
consentModeRegions: ["CZ", "SK", "DE"],  // consent default platí jen pro tyto země
// prázdné pole [] = platí globálně pro všechny uživatele
```

### wait_for_update

```javascript
consentModeWaitForUpdate: 500,  // ms — jak dlouho Google tagy čekají na consent update
```

Google tagy po načtení stránky počkají tuto dobu, než se spustí s výchozími (denied) hodnotami. Dává uživateli čas na interakci s bannerem.

## dataLayer eventy

Banner posílá do dataLayer dva eventy, které můžete použít jako GTM triggery:

| Event | Kdy se spustí | Data |
|-------|---------------|------|
| `cookiebarConsentInit` | Při načtení stránky (pokud existuje uložený souhlas) | `consentedCategories: "strictly-necessary,analytical,..."` |
| `cookiebarConsentUpdate` | Po každé změně souhlasu uživatelem | `consentedCategories: "strictly-necessary,analytical,..."` |

### Příklad GTM triggeru

1. Vytvořte trigger typu **Custom Event**
2. Event name: `cookiebarConsentUpdate`
3. Použijte jako trigger pro tagy, které mají reagovat na změnu souhlasu

## Vícejazyčná podpora

### Přidání nového jazyka

Do pole `texts` přidejte nový objekt:

```javascript
texts: [
    { lang: "cs", header: "...", intro: "...", ... },
    { lang: "en", header: "...", intro: "...", ... },
    // Nový jazyk:
    {
        lang: "de",
        header: "Datenschutzeinstellungen",
        intro: "Diese Website verwendet Cookies...",
        buttons: {
            accept: "Alle akzeptieren",
            deny: "Nur notwendige",
            save: "Einstellungen speichern"
        },
        statusMessages: {
            accepted: "Alle Cookies akzeptiert",
            denied: "Optionale Cookies abgelehnt",
            saved: "Cookie-Einstellungen gespeichert"
        },
        categories: {
            required: { header: "Notwendige Cookies", body: "..." },
            analytical: { header: "Analytische Cookies", body: "..." },
            preferences: { header: "Präferenz-Cookies", body: "..." },
            marketing: { header: "Marketing-Cookies", body: "..." }
        }
    }
]
```

### Vypnutí vícejazyčnosti

```javascript
multilingual: false,     // skryje dropdown pro výběr jazyka
langAutoSelect: false,   // nedetekuje jazyk prohlížeče
defaultLang: "sk",       // vynutí slovenštinu jako primární zobrazený jazyk
```

## Všechny konfigurační parametry

| Parametr | Typ | Default | Popis |
|----------|-----|---------|-------|
| `host` | string | `""` | Doména pro cookie (viz sekce výše) |
| `version` | string | — | Verze CMP; zvýšení vynutí nový souhlas |
| `modal` | HTMLElement | — | Root element modalu (`#oscbr`) |
| `apiKey` | string | `""` | API klíč pro server endpoint |
| `dismissible` | boolean | `true` | Zda lze modal zavřít bez volby |
| `multilingual` | boolean | `false` | Zobrazit dropdown pro výběr jazyka |
| `langAutoSelect` | boolean | `false` | Automatická detekce jazyka prohlížeče |
| `defaultLang` | string | `""` | Výchozí jazyk (např. `"sk"`), použitý pokud selže autodetekce |
| `sendToServer` | boolean | `true` | Odesílat souhlas na server |
| `consentEndpoint` | string | `""` | URL endpointu pro odesílání souhlasu |
| `autoFocus` | boolean | `true` | Automatický focus na modal při zobrazení |
| `logLevel` | string | `"INFO"` | Úroveň logování: NONE, ERROR, WARN, INFO, DEBUG |
| `consentModeRegions` | array | `[]` | Regiony pro consent default (prázdné = globální) |
| `consentModeWaitForUpdate` | number | `500` | Timeout (ms) pro Google tagy |
| `cookieName` | string | `"customConsent"` | Název consent cookie |
| `cookieLifetimeDays` | number | `365` | Životnost cookie ve dnech |
| `refreshDays` | number | `365` | Prodloužení cookie při validaci |
| `categoryConfig` | object | viz výše | Konfigurace kategorií (required, defaultEnabled) |
| `texts` | array | — | Pole jazykových objektů s překlady |
| `delayBeforeSave` | number | `500` | Prodleva (ms) před uložením souhlasu |
| `delayModalClose` | number | `1000` | Prodleva (ms) před zavřením modalu |
| `delayInitialFocus` | number | `100` | Prodleva (ms) před nastavením focusu |

## Přizpůsobení designu

Design je řízen CSS proměnnými (design tokeny) na začátku souboru:

```css
.oscbr {
    --cb-theme: #2563EB;           /* Hlavní barva (tlačítka, linky, toggle) */
    --cb-theme-hover: #1D4ED8;     /* Hover barva */
    --cb-bg: #FFFFFF;              /* Pozadí modalu */
    --cb-text: #1F2937;            /* Barva textu */
    --cb-text-secondary: #6B7280;  /* Sekundární text */
    --cb-overlay: transparent;      /* Pozadí overlay (transparent = scrollovatelné pozadí) */
    --cb-slider-off: #D1D5DB;      /* Barva vypnutého toggleru */
    --cb-border: #E5E7EB;          /* Barva okrajů */
    --cb-accordion-bg: #F9FAFB;    /* Pozadí accordionu */
    --cb-accordion-hover-bg: #F3F4F6; /* Hover pozadí accordionu */
    --cb-accordion-text: #374151;  /* Text accordionu */
    --cb-focus: #2563EB;           /* Barva focus outlines */
    --cb-radius: 12px;             /* Border radius modalu */
    --cb-radius-sm: 8px;           /* Border radius tlačítek/accordionu */
    --cb-font: 'Inter', 'Segoe UI', system-ui, -apple-system, sans-serif;
}
```

Pro změnu barevného schématu stačí upravit tyto proměnné — celý banner se automaticky přizpůsobí.

## JavaScript API

Banner se inicializuje automaticky, ale můžete s ním interagovat programaticky:

```javascript
// Znovu otevřít banner (např. z odkazu "Nastavení cookies" v patičce)
window.OSCBR.modal.style.display = "block";

// Zjistit aktuální souhlas
window.OSCBR.instance.getCategories();
// => ["strictly-necessary", "analytical"]

// Cleanup (pro SPA nebo GTM preview)
window.OSCBR.cleanup();
```

### Odkaz pro znovuotevření banneru

Pokud chcete v patičce webu zobrazit odkaz „Nastavení cookies":

```html
<a href="#" onclick="if(window.OSCBR && window.OSCBR.modal){window.OSCBR.modal.style.display='block';}return false;">
    Nastavení cookies
</a>
```

## Bezpečnost

- **XSS ochrana** — veškerý HTML obsah z konfigurace prochází sanitizerem (DOMParser whitelist: `A, STRONG, B, EM, I, U, BR, UL, OL, LI, SUP, SUB, CODE, P, SPAN`)
- **Cookie security** — `SameSite=Lax; Secure` na všech cookies
- **rel="noopener noreferrer"** — automaticky přidáno na všechny `<a target="_blank">` linky
- **CSS izolace** — `all: unset` s prefixem `.oscbr` zabraňuje kolizím se styly webu
- **Žádné externí závislosti** — vše je v jednom souboru, žádné CDN/API volání


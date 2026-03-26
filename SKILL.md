---
description: Analyze placement report (CSV or API) and exclude spam/MFA websites and YouTube channels at account level in Google Ads. API mode includes conversion data — placements with conversions are protected from exclusion.
---

Analiza placementów i wykluczanie spamu (witryny + kanały YouTube) na poziomie konta.

## Kiedy używać

Użytkownik chce przeanalizować raport placementów lub wykluczyć spam/MFA witryny/kanały YT.

## Źródła danych

| Typ kampanii | CSV | API (`detail_placement_view`) |
|---|---|---|
| **PMax** | TAK (jedyne źródło) | NIE — Google nie udostępnia raportu "Miejsce docelowe" przez API |
| **GDN / Display** | TAK | TAK |
| **Demand Gen** | TAK | TAK |
| **Video** | TAK | TAK |

**Zapytaj użytkownika**: czy ma plik CSV, czy chce pobrać dane z API (tylko GDN/Demand Gen/Video).

## Wymagane dane

- **Tryb CSV**: Plik CSV — "Miejsce docelowe kampanii Performance Max" lub raport placementów GDN/DG/Video (encoding: UTF-16 TSV lub CSV lub XLS)
- **Tryb API**: Alias konta + opcjonalnie nazwa kampanii (domyślnie: wszystkie GDN/Demand Gen/Video z ostatnich 30 dni)
- **Konta** — aliasy kont na których dodać wykluczenia

## Procedura

### Krok 1a — Wczytaj z CSV

```python
import sys; sys.stdout.reconfigure(encoding='utf-8')
import urllib.request, json, re, time

filepath = r"<ŚCIEŻKA_DO_PLIKU>"
for enc in ['utf-16', 'utf-8-sig', 'utf-8', 'cp1250']:
    try:
        with open(filepath, encoding=enc) as f:
            lines = f.readlines()
        break
    except (UnicodeDecodeError, UnicodeError):
        continue

data = []
source_mode = "csv"
for line in lines[2:]:
    parts = line.strip().split('\t')
    if len(parts) >= 3:
        name = parts[0].strip().strip('"')
        url = parts[1].strip()
        impr_str = parts[2].strip().replace('\xa0', '').replace(' ', '').replace(',', '')
        try: impr = int(impr_str)
        except: impr = 0
        data.append({"name": name, "url": url, "impressions": impr, "conversions": 0.0})
```

### Krok 1b — Pobierz z API (tylko GDN / Demand Gen / Video)

**NIE działa dla PMax** — Google Ads API nie zwraca placementów PMax przez `detail_placement_view`.

```python
import sys; sys.stdout.reconfigure(encoding='utf-8')
import urllib.request, json, re, time
from bdos import connect

ctx = connect("<ALIAS>")
cid = ctx.customer_id

# Opcjonalnie: filtruj po kampanii
campaign_filter = ""  # lub: "AND campaign.id = 123456789"
days = 30

gaql = f"""
    SELECT
        detail_placement_view.display_name,
        detail_placement_view.target_url,
        detail_placement_view.placement_type,
        metrics.impressions,
        metrics.conversions
    FROM detail_placement_view
    WHERE segments.date DURING LAST_30_DAYS
        AND campaign.advertising_channel_type IN ('DISPLAY', 'DEMAND_GEN', 'VIDEO')
        {campaign_filter}
    ORDER BY metrics.impressions DESC
"""

rows = ctx.client.query(gaql, cid)

data = []
source_mode = "api"
for r in rows:
    name = r.get('display_name', r.get('detail_placement_view_display_name', ''))
    url = r.get('target_url', r.get('detail_placement_view_target_url', ''))
    impr = int(r.get('impressions', r.get('metrics_impressions', 0)))
    conv = float(r.get('conversions', r.get('metrics_conversions', 0)))
    ptype = r.get('placement_type', r.get('detail_placement_view_placement_type', ''))
    if impr > 0:
        data.append({"name": name, "url": url, "impressions": impr, "conversions": conv, "placement_type": ptype})

print(f"Pobrano {len(data)} placementów z API")
```

**Po kroku 1a lub 1b `data` ma ten sam format** — reszta procedury jest wspólna.

**WAŻNE: Krok 1 + Krok 2 + Krok 2b wykonuj w JEDNYM skrypcie** — nie dziel na osobne uruchomienia. Cały pipeline (pobranie danych → klasyfikacja domen → rozwiązanie kanałów YouTube) powinien być jednym skryptem, żeby uniknąć duplikowania kodu i wielokrotnego odpytywania API.

### Krok 2 — Klasyfikuj witryny

Kategorie: 1) Google owned (`Należące do Google`) → pomiń, 2) Mobile App → pomiń, 3) YouTube → krok 2b, 4) Spam/MFA (`is_spam(extract_domain(url))`) → tabela 1, 5) Podejrzane MFA (`is_suspect_mfa(extract_domain(url))`) → tabela 2, 6) Portale/polskie domeny → tabela 4.

**REGUŁA KONWERSJI (tylko tryb API):** Jeśli domena/kanał ma **>= 1 konwersję** → trafia do tabeli OK (4 lub 5) **niezależnie od klasyfikacji spam/MFA**. W kolumnie Uwagi dodaj oryginalną klasyfikację z dopiskiem "— konwertuje".

**Klasyfikacja po DOMENIE**: Przed wywołaniem `is_spam()` / `is_suspect_mfa()` wyciągnij domenę przez `extract_domain(url)`. Pełne URLe zawierają polskie słowa w ścieżkach artykułów (np. "odporna" matchuje "porn", "idealna" matchuje "deal") co daje false positives. Dane w tabelach pokazuj jako domenę z łączną liczbą wyświetleń (agregacja po domenie).

**ZAWSZE pokaż użytkownikowi pełną listę i czekaj na akceptację przed wykluczeniem.**

```python
from urllib.parse import urlparse

def extract_domain(url):
    """Wyciąga domenę z URL (bez www., bez ścieżki)."""
    if not url.startswith('http'):
        url = 'http://' + url
    parsed = urlparse(url)
    domain = parsed.netloc or parsed.path.split('/')[0]
    return domain.lower().removeprefix('www.')

def get_root_domain(domain):
    """Wyciąga root domain (ostatnie 2 segmenty, lub 3 dla .co.uk, .com.pl itp.)."""
    parts = domain.split('.')
    if len(parts) >= 3 and parts[-2] in ('co', 'com', 'net', 'org'):
        return '.'.join(parts[-3:])
    return '.'.join(parts[-2:]) if len(parts) >= 2 else domain

spam_tlds = ['.top', '.cc', '.vip', '.club', '.online', '.lat', '.cv', '.pw',
             '.su', '.in', '.cn', '.xyz', '.fun', '.site', '.ru', '.live']

spam_patterns = [
    'game', 'quiz', 'deals', 'coupon', 'coupons', 'tarot',
    'vpn', 'download', 'buzz', 'viral', 'celebrity',
    'novel', 'clearance', 'promo', 'arcade',
    'gamer', 'sdk', 'freegame', 'sportlit', 'modabright', 'cheerzone',
    'tipgalore', 'pemplay', 'dailyneuron', 'livenews', 'suburban', 'flying',
    'lawyer', 'thump', 'bonus', 'winbig', 'prizes',
    'xxx', 'naked', 'adult', 'sex', 'porn', 'escort', 'porno', 'randki',
    'crypto', 'coin', 'bitcoin', 'forex', 'payday', 'wealth',
    'casino',
    'minecraft', 'gta', 'solitaire', 'scrabble', 'roblox', 'fortnite', 'poki', 'gier', 'gry',
    'esport', 'pubg', 'valorant',
    'dowcipy', 'smieszne', 'bajki', 'wygraj', 'zarabiaj', 'tanio', 'pupa',
]

political_patterns = [
    'trump', 'biden', 'harris', 'obama', 'clinton', 'sanders', 'pelosi',
    'putin', 'jinping', 'vance', 'election', 'politics', 'political',
    'congress', 'senate', 'democrat', 'republican', 'liberal', 'conservative',
    'racism', 'racist',
    'kaczynski', 'tusk', 'holownia', 'mentzen',
    'wybory', 'polityka', 'sejm', 'senat',
]

known_spam = [
    'kaik.ai', 'petronu.com', 'uvthe.com', 'hwcopage.com', 'eoydo.com',
    'pptzs.com', 'koeeis.com', 'srpo.net', 'bartbash.org', 'bluelifer.com',
    'ulockpic.com', 'pastspedia.com', 'uznavator.com', 'directsharing.com',
    'nextwatch.net', 'housejogger.com', 'countingmypennies.com', 'luxarts.net',
    'reachymini.net', 'tuteehub.com', 'bookhike.com', 'lsmagazineimg.com',
    'comfortweather.com', 'drive-fact.com', 'modabright.net', 'verloarcade.com',
    'pulchritudeclothes.com', 'reportingly.com', 'thecelebritist.com',
    'modelmatic.net', 'appposts.com', 'thinkcrate.net', 'infobuono.com',
    'zodiacfortune.net', 'havenlyspaces.net', 'tasteezy.net', 'dobrezrodla.com',
    'gernarb.com', 'wonderluhst.net', 'mealse.com', 'azontree.com',
    'moroccoplaces.com', 'swidnw.com', 'travel2days.com', 'haberion.com',
    'echo-sphere-net.com', 'money-glamours.com', 'taboolanews.com',
    'poradyiwskazowki.pl', 'zjawiskaniewyjasnione.pl', 'liveradioworld.com',
]

known_legit = [
    # Polskie portale / media
    'onet.pl', 'wp.pl', 'interia.pl', 'gazeta.pl', 'o2.pl', 'polki.pl',
    'pomponik.pl', 'fakt.pl', 'rmf24.pl', 'rmf.fm', 'wprost.pl', 'planeta.pl',
    'dziennik.pl', 'tvn24.pl', 'tvrepublika.pl', 'se.pl', 'wyborcza.pl',
    'niezalezna.pl', 'wpolityce.pl', 'dorzeczy.pl', 'goniec.pl', 'plejada.pl',
    'pudelek.pl', 'plotek.pl', 'radiozet.pl', 'sport.pl', 'polsatnews.pl',
    'natemat.pl', 'forsal.pl', 'infor.pl', 'mp.pl', 'gloswielkopolski.pl',
    'dziennikzachodni.pl', 'pomorska.pl', 'naszemiasto.pl', 'gk24.pl',
    'styl.fm', 'ofeminin.pl', 'deccoria.pl', 'wlocie.pl', 'goracetematy.pl',
    'swiatgwiazd.pl', 'newspol.pl', 'tvn.pl', 'polsat.pl', 'tvp.info',
    'nczas.info', 'kulisy.net', 'wrzuc.info',
    # Polskie e-commerce / ogłoszenia
    'olx.pl', 'allegro.pl', 'ceneo.pl', 'empik.pl', 'morele.net', 'x-kom.pl',
    'mediaexpert.pl', 'euro.com.pl', 'sprzedajemy.pl', 'gazetki123.pl',
    'eperfumy.pl', 'notino.pl', 'rossmann.pl', 'hebe.pl', 'douglas.pl',
    'firmy.net', 'tomaszowiak.net',
    # Polskie portale — uzupełnienie
    'wykop.pl', 'bankier.pl', 'money.pl', 'medonet.pl', 'kafeteria.pl', 'parenting.pl', 'dobreprogramy.pl',
    'benchmark.pl', 'komputerswiat.pl', 'pepper.pl', 'fotoblogia.pl',
    'spidersweb.pl', 'antyweb.pl', 'telepolis.pl', 'tabletowo.pl',
    'instalki.pl', 'pclab.pl', 'purepc.pl', 'gry-online.pl', 'eurogamer.pl',
    'dlawas.info', 'infoprzasnysz.com', 'sadeczanin.info', 'faktykaliskie.info',
    'elubaczow.com', 'ki24.info', 'lowiczanin.info', 'budujesz.info',
    'zambrow.org', 'bialobrzegi24.net', 'rolnicy.com', 'polskapilka.net',
    'kwestiasmaku.com', 'transfery.info', 'lwowecki.info', 'tygodniksiedlecki.com',
    'rzeszow24.info', 'gniezno24.com', 'jelonka.com', 'szarada.net',
    'hokej.net', 'pleszew24.info', 'kkslech.com', 'kamienskie.info',
    'radiobiper.info', 'hasladokrzyzowek.com', 'grojec24.net', 'hdtvpolska.com',
    'siatka.org', 'gorzowianin.com', 'tychy24.net', 'tuwroclaw.com',
    'mojewypieki.com', 'halogorlice.info', 'bolec.info', 'widzewtomy.net',
    'disco-polo.info', 'fcbarca.com', 'bielsko.info', 'losice.info',
    'pzhgp.net', 'katowice24.info', 'dogomania.com', 'feszyn.com',
    'filmozercy.com', 'energetyka24.com', 'mytractorforum.com',
    # Globalne platformy
    'blogspot.com', 'wikipedia.org', 'reddit.com', 'facebook.com',
    'instagram.com', 'twitter.com', 'linkedin.com', 'pinterest.com',
    'amazon.com', 'ebay.com', 'youtube.com',
    # Whitelista użytkownika
    'ino.online',
]

def is_spam(domain):
    """Klasyfikuje DOMENĘ (nie pełny URL). Użyj extract_domain() przed wywołaniem.
    Sprawdza też root domain — subdomena known_spam = spam."""
    domain_lower = domain.lower()
    root = get_root_domain(domain_lower)
    if domain_lower in known_legit or root in known_legit: return False, ""
    if domain_lower in known_spam or root in known_spam: return True, "known_spam"
    for tld in spam_tlds:
        if domain_lower.endswith(tld): return True, f"spam_tld ({tld})"
    for pat in spam_patterns:
        if pat in domain_lower: return True, f"spam_pattern ({pat})"
    for pat in political_patterns:
        if pat in domain_lower: return True, f"political ({pat})"
    return False, ""

def is_suspect_mfa(domain):
    """Klasyfikuje DOMENĘ. Domena .com/.net/.org/.shop/.info lub zagraniczne TLD, nie na whiteliście = podejrzane MFA.
    Sprawdza też root domain — subdomena known_legit = legit."""
    domain_lower = domain.lower()
    root = get_root_domain(domain_lower)
    # Zagraniczne TLD → od razu podejrzane (chyba że root w known_legit)
    foreign_tlds = ['.lt', '.ua', '.vn', '.bg', '.by', '.kz', '.uz', '.az', '.ge', '.md', '.lv', '.ee']
    if any(domain_lower.endswith(t) for t in foreign_tlds):
        if domain_lower not in known_legit and root not in known_legit:
            return True
    suspect_tlds = ['.com', '.net', '.org', '.shop', '.info']
    if not any(domain_lower.endswith(t) for t in suspect_tlds):
        return False
    if domain_lower in known_legit or root in known_legit:
        return False
    # Sprawdź czy domena .pl (subdomena .com.pl itp.)
    if '.pl' in domain_lower:
        return False
    return True

# Agreguj po domenie i klasyfikuj
from collections import defaultdict
domain_data = defaultdict(lambda: {'impressions': 0, 'conversions': 0.0})
google_owned, youtube_entries = [], []

for entry in data:
    url = entry['url']
    ptype_str = str(entry.get('placement_type', '')).upper()
    if 'MOBILE' in ptype_str or 'APP' in ptype_str:
        continue  # Mobile App → pomiń (do tego exclude_mobile_apps())
    if 'youtube.com' in url.lower():
        youtube_entries.append(entry)
        continue
    if url.strip() == '--' or not url.strip():
        google_owned.append(entry)
        continue
    domain = extract_domain(url)
    domain_data[domain]['impressions'] += entry['impressions']
    domain_data[domain]['conversions'] += entry.get('conversions', 0.0)

table1_spam, table2_mfa, table4_legit = [], [], []
protected_count = 0  # domeny spam/mfa przeniesione do OK dzięki konwersjom

for domain, stats in sorted(domain_data.items(), key=lambda x: x[1]['impressions'], reverse=True):
    impressions = stats['impressions']
    conversions = stats['conversions']

    # REGUŁA KONWERSJI: >= 1 konwersja → tabela 4 (OK), niezależnie od klasyfikacji
    if source_mode == "api" and conversions >= 1.0:
        spam, reason = is_spam(domain)
        mfa = is_suspect_mfa(domain) if not spam else False
        note = f"{reason} — konwertuje" if spam else ("podejrzane MFA — konwertuje" if mfa else "")
        if spam or mfa:
            protected_count += 1
        table4_legit.append({'url': domain, 'impressions': impressions, 'conversions': conversions, 'note': note})
        continue

    spam, reason = is_spam(domain)
    if spam:
        table1_spam.append({'url': domain, 'impressions': impressions, 'reason': reason})
    elif is_suspect_mfa(domain):
        table2_mfa.append({'url': domain, 'impressions': impressions})
    else:
        table4_legit.append({'url': domain, 'impressions': impressions, 'conversions': conversions, 'note': ''})
```

### Krok 2b — Rozwiąż kanały YouTube (YouTube Data API)

Wyodrębnij video ID z URL (`youtube.com/video/<ID>`), batchowo odpytaj API (50/request), pogrupuj po kanale.

```python
# Klucz API z ~/.bdos/api_keys.yaml (wymaga YouTube Data API v3)
import yaml
from pathlib import Path
with open(Path.home() / '.bdos' / 'api_keys.yaml') as f:
    YT_API_KEY = yaml.safe_load(f)['youtube_data_api_v3']

# youtube_entries już zbudowane w kroku 2 (agregacja)
yt_with_vid = []
for entry in youtube_entries:
    m = re.search(r'youtube\.com/video/([a-zA-Z0-9_-]+)', entry['url'])
    if m:
        yt_with_vid.append({**entry, 'video_id': m.group(1)})

video_ids = list(set(e['video_id'] for e in yt_with_vid))
video_to_channel = {}

for i in range(0, len(video_ids), 50):
    batch = video_ids[i:i+50]
    api_url = f'https://www.googleapis.com/youtube/v3/videos?part=snippet&id={",".join(batch)}&key={YT_API_KEY}'
    try:
        with urllib.request.urlopen(urllib.request.Request(api_url)) as resp:
            for item in json.loads(resp.read()).get('items', []):
                video_to_channel[item['id']] = {
                    'channel_id': item['snippet']['channelId'],
                    'channel_title': item['snippet']['channelTitle'],
                }
    except Exception as e:
        print(f"Error batch {i}: {e}")
    time.sleep(0.1)

channels = {}
for e in yt_with_vid:
    if e['video_id'] in video_to_channel:
        ch = video_to_channel[e['video_id']]
        ch_id = ch['channel_id']
        if ch_id not in channels:
            channels[ch_id] = {'title': ch['channel_title'], 'impressions': 0, 'video_count': 0, 'conversions': 0.0}
        channels[ch_id]['impressions'] += e['impressions']
        channels[ch_id]['video_count'] += 1
        channels[ch_id]['conversions'] += e.get('conversions', 0.0)

sorted_channels = sorted(channels.items(), key=lambda x: x[1]['impressions'], reverse=True)
```

Klasyfikuj kanały — `yt_spam_patterns` na `channel_title`. Pasujące → tabela 3. Reszta → tabela 5. **Wyjątek: kanały z >= 1 konwersją → tabela 5 niezależnie od klasyfikacji.**

```python
yt_spam_patterns = [
    # Bajki / dzieci / piosenki
    'bajki', 'bajka', 'for kids', 'kids channel', 'kids songs', 'baby songs',
    'nursery rhymes', 'children', 'kinder', 'peppa', 'psi patrol',
    'śpiewające brzdące', 'krecik', 'masza', 'strażak sam', 'fireman sam',
    'bazylland', 'vlad', 'roma show', 'hejka tu lenka', 'yummy toys',
    'cocomelon', 'blippi', 'teletubbies', 'paw patrol', 'little angel',
    'super simple', 'chu chu tv', 'bounce patrol', 'tot school', 'tiny tots',
    'toddler', 'lullaby', 'zabawa', 'maluch', 'przedszkolak',
    'piosenki dla dzieci', 'śpiewanki', 'wierszyki', 'kolorowanki',
    'baby shark', 'pinkfong', 'bing polski', 'nick jr',
    # Gry / gaming
    'minecraft', 'roblox', 'fortnite', 'gta', 'poki', 'play game',
    'gameplay', 'walkthrough', 'lets play', "let s play", 'gaming', 'gamer',
    'valorant', 'brawl stars', 'among us',
    'fifa', 'ea fc', 'call of duty', 'cod mobile', 'apex legends',
    'league of legends', 'counter strike', 'cs2', 'csgo',
    'overwatch', 'pubg', 'free fire', 'clash royale', 'clash of clans',
    'pokemon', 'pokémon', 'world of warcraft', 'diablo', 'dota',
    'elden ring', 'hogwarts', 'streamer', 'twitch', 'esport', 'speedrun',
]

def has_foreign_script(text):
    """Sprawdza czy tekst zawiera znaki spoza łacińskiego alfabetu (cyrylica, chiński, arabski itp.)."""
    import unicodedata
    for ch in text:
        if ch.isalpha():
            name = unicodedata.name(ch, '')
            if any(s in name for s in ['CYRILLIC', 'CJK', 'ARABIC', 'THAI', 'DEVANAGARI', 'BENGALI', 'HANGUL']):
                return True
    return False

table3, table5 = [], []
yt_protected_count = 0  # kanały spam przeniesione do OK dzięki konwersjom

for ch_id, ch in sorted_channels:
    title_lower = ch['title'].lower()
    conversions = ch.get('conversions', 0.0)

    # REGUŁA KONWERSJI: >= 1 konwersja → tabela 5 (OK), niezależnie od klasyfikacji
    if source_mode == "api" and conversions >= 1.0:
        matched = next((p for p in yt_spam_patterns if p in title_lower), None)
        foreign = has_foreign_script(ch['title'])
        note = ""
        if matched:
            note = f"{matched} — konwertuje"
            yt_protected_count += 1
        elif foreign:
            note = "obcy alfabet — konwertuje"
            yt_protected_count += 1
        table5.append((ch_id, ch, note))
        continue

    matched = next((p for p in yt_spam_patterns if p in title_lower), None)
    if matched:
        table3.append((ch_id, ch, matched))
    elif has_foreign_script(ch['title']):
        table3.append((ch_id, ch, 'obcy alfabet'))
    else:
        table5.append((ch_id, ch, ''))
```

### Krok 3 — Pokaż wyniki

**Najpierw tabela zbiorcza**, potem 5 tabel szczegółowych z nagłówkami `###`. Sortuj po wyświetleniach malejąco.

**Zasady wyświetlania:**
- **Tabele 1, 2, 3** (spam/podejrzane) — wypisuj **WSZYSTKIE pozycje**, bez skracania
- **Tabele 4, 5** (OK) — wypisuj tylko pozycje z **>50 wyświetleń** (wyjątek: pozycje z >= 1 konwersją pokazuj zawsze, niezależnie od wyświetleń)

Tabela zbiorcza — ZAWSZE na początku, przed tabelami szczegółowymi:

```markdown
**Podsumowanie placementów — [źródła danych]**

| Kategoria | Ilość | Wyświetlenia |
|---|---:|---:|
| Google owned | — | X |
| Witryny SPAM | X | X |
| Witryny podejrzane | X | X |
| Witryny OK | X | X |
| Kanały YT SPAM | X | X |
| Kanały YT OK | X | X |
| Spam/podejrzane z konwersjami (przeniesione do OK) | X | X |
```

**Wiersz "Spam/podejrzane z konwersjami"** — pokazuj TYLKO w trybie API i gdy `protected_count + yt_protected_count > 0`. Suma domen i kanałów przeniesionych do OK dzięki konwersjom.

| Tabela | Treść | Próg wyświetleń | Nagłówki kolumn |
|--------|-------|---:|---|
| 1. Witryny SPAM - do wykluczenia | Znane spam, MFA, zagraniczne TLD, polityka | wszystkie | #, Witryna, Wyświetlenia, Powód |
| 2. Witryny podejrzane - do weryfikacji | Domeny .com/.net/.org/.shop/.info itd. spoza whitelisty | wszystkie | #, Witryna, Wyświetlenia |
| 3. Kanały Youtube SPAM - do wykluczenia | Kanały bajki/dzieci/gry | wszystkie | #, Kanał, Channel ID, Wyświetlenia, Filmów, Powód |
| 4. Witryny — OK | Whitelista + polskie domeny .pl + **spam z konwersjami** | >50 | **CSV**: #, Witryna, Wyświetlenia / **API**: #, Witryna, Wyświetlenia, Konwersje, Uwagi |
| 5. Kanały YouTube — OK | Pozostałe kanały + **spam YT z konwersjami** | >50 | **CSV**: #, Kanał, Channel ID, Wyświetlenia, Filmów / **API**: #, Kanał, Channel ID, Wyświetlenia, Filmów, Konwersje, Uwagi |

Każda tabela z nagłówkiem markdown `###` zawierającym nazwę, liczbę elementów i wyświetleń.

**OBOWIĄZKOWE opisy pod nagłówkami** — ZAWSZE dodawaj poniższe opisy między nagłówkiem `###` a tabelą. Nie pomijaj ich:

| Tabela | Opis (OBOWIĄZKOWY) |
|--------|---|
| 1 | Witryny jednoznacznie spamowe: znane domeny spam, podejrzane TLD (.top, .xyz, .club...), treści dla dorosłych, kasyno, crypto, gry, polityka. Rekomendacja: wykluczyć wszystkie. |
| 2 | Domeny .com/.net/.org/.shop/.info spoza whitelisty — mogą być MFA (Made For Ads) lub legit. Sprawdź ręcznie przed wykluczeniem. |
| 3 | Kanały z bajkami, grami, treściami dla dzieci lub w obcym alfabecie — nieadekwatne dla reklam. Rekomendacja: wykluczyć wszystkie. |
| 4 | Zweryfikowane witryny: polskie portale, media, e-commerce, znane domeny .pl. Nie wymagają wykluczenia. Pokazane powyżej 50 wyświetleń. **Tryb API**: kolumna Konwersje + Uwagi (witryny przeniesione ze spam/MFA dzięki konwersjom oznaczone w Uwagach). |
| 5 | Kanały YouTube niesklasyfikowane jako spam. Pokazane powyżej 50 wyświetleń. **Tryb API**: kolumna Konwersje + Uwagi (kanały przeniesione ze spam dzięki konwersjom oznaczone w Uwagach). |

Zapytaj użytkownika: które tabele wykluczyć i na których kontach?

### Krok 4 — Wyklucz na kontach

**Witryny** → `placement.url` (typ `PLACEMENT`). **Kanały YT** → `youtube_channel.channel_id` (typ `YOUTUBE_CHANNEL`).
**YouTube URLs NIE działają jako placement** — API zwraca błąd.

```python
from bdos import connect
from google.ads.googleads.v23.resources.types.customer_negative_criterion import CustomerNegativeCriterion
from google.ads.googleads.v23.services.types.customer_negative_criterion_service import CustomerNegativeCriterionOperation
from google.ads.googleads.v23.services.types.google_ads_service import MutateOperation

ctx = connect("alias")
cid = ctx.customer_id

def exclude_batch(ctx, spam_urls, channel_ids, dry_run=True):
    # --- Witryny (placement) ---
    existing = ctx.client.query(
        "SELECT customer_negative_criterion.placement.url "
        "FROM customer_negative_criterion WHERE customer_negative_criterion.type = 'PLACEMENT'", cid)
    existing_urls = {r.get('placement_url', r.get('url', '')).lower() for r in existing if r.get('placement_url', r.get('url', ''))}
    to_add = [u for u in spam_urls if u.lower() not in existing_urls]

    for i in range(0, len(to_add), 1000):
        ops = []
        for url in to_add[i:i+1000]:
            c = CustomerNegativeCriterion()
            c.placement.url = url
            ops.append(MutateOperation(customer_negative_criterion_operation=CustomerNegativeCriterionOperation(create=c)))
        ctx.client.mutate(ops, dry_run=dry_run)

    # --- Kanały YouTube (youtube_channel) ---
    existing_yt = ctx.client.query(
        "SELECT customer_negative_criterion.youtube_channel.channel_id "
        "FROM customer_negative_criterion WHERE customer_negative_criterion.type = 'YOUTUBE_CHANNEL'", cid)
    existing_ch = {r.get('youtube_channel_channel_id', r.get('channel_id', '')) for r in existing_yt
                   if r.get('youtube_channel_channel_id', r.get('channel_id', ''))}
    to_add_yt = [ch for ch in channel_ids if ch not in existing_ch]

    if to_add_yt:
        ops = []
        for ch_id in to_add_yt:
            c = CustomerNegativeCriterion()
            c.youtube_channel.channel_id = ch_id
            ops.append(MutateOperation(customer_negative_criterion_operation=CustomerNegativeCriterionOperation(create=c)))
        ctx.client.mutate(ops, dry_run=dry_run)

    print(f"{'Preview' if dry_run else 'Wykonano'}: {len(to_add)} witryn + {len(to_add_yt)} kanałów YT")
    return to_add, to_add_yt

# Krok 4a: dry_run=True — preview
to_add, to_add_yt = exclude_batch(ctx, spam_urls, channel_ids, dry_run=True)
```

**Pokaż użytkownikowi preview wykluczeń i CZEKAJ na potwierdzenie** ('tak', 'wykonaj', 'lecimy') przed wykonaniem.

```python
# Krok 4b: dry_run=False — wykonanie po potwierdzeniu
exclude_batch(ctx, spam_urls, channel_ids, dry_run=False)
```

### Krok 5 — Raport

| Konto | Witryny dodane | Kanały YT dodane | Było już | Status |
|-------|---:|---:|---:|---|

## Zasady

- **NIGDY** nie wykluczaj bez akceptacji użytkownika
- **NIGDY** nie wykluczaj witryn trafnych (branżowych) dla konta
- **NIGDY** nie wykluczaj domen/kanałów z >= 1 konwersją (tryb API) — przenieś do tabeli OK z adnotacją
- Pomiń aplikacje mobilne — do tego `exclude_mobile_apps()` z mutations.py
- Kanały YT: `youtube_channel.channel_id` (typ `YOUTUBE_CHANNEL`), NIE placement URL
- Wykluczenia na **poziomie konta** (`customer_negative_criterion`) — dotyczą WSZYSTKICH kampanii
- Raport: PMax → Statystyki → Miejsca docelowe → Pobierz
- Warto uruchamiać co 2-4 tygodnie

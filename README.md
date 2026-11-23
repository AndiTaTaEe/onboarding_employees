# Șablon UiPath pentru faza 2, în C#

Același șablon ca `SablonFaza2`, cu expresii C# în loc de VB: proiect Process, Cross-platform, cu pachetele `UiPath.System.Activities` și `UiPath.WebAPI.Activities`. Alegeți unul singur la început: limbajul expresiilor se stabilește când se creează proiectul și nu se mai poate schimba. Studio Web nu deschide proiecte Cross-platform în C#.

- `Main.xaml`: primește `in_BaseUrl`, citește `GET /context` (data de lucru, ca text și ca `DateTime`) și `GET /config` (parametrii regulilor), apoi se oprește. De aici continuați procesarea.
- `HubRequest.xaml`: un apel la hub, cu politica de reîncercare din „01 Ghid student”. Toate apelurile trec prin el.

## HubRequest.xaml

| Argument | Ce este |
|---|---|
| `in_Method` | `GET` sau `POST` |
| `in_Url` | URL-ul complet, de exemplu `baseUrl + "/inbox"` |
| `in_Body` | corpul JSON al unui POST, ca text: `corp.ToString(Formatting.None)` (gol la GET) |
| `in_IdempotencyKey` | cheia din enunț, la fiecare POST |
| `out_StatusCode` | statusul final; 0 dacă nu a venit niciun răspuns |
| `out_Failed` | True când politica a renunțat: 3 încercări eșuate sau peste 10 așteptări la 429 |
| `out_Body` | corpul răspunsului, ca text (JSON sau XML) |
| `out_Json` | corpul ca `JToken`; datele și sumele rămân text, exact ca în răspuns |
| `out_FilePath` | fișierul descărcat când răspunsul este un PDF (în `%TEMP%\bpa-documente`), altfel gol |

Reîncearcă singur 500, 502, 503, 504, timeout-ul și eroarea de rețea (cel mult 3 încercări) și 429 (cel mult 10 așteptări), după `Retry-After`, cu aceeași cheie și același corp. Fiecare reîncercare apare în log ca Warn; liniile „Reincercare dupa 429” apar și în rulările fără erori, fiindcă robotul trimite peste 20 de cereri pe secundă. Oprește robotul la 429 `RUN_REQUEST_LIMIT` și la 503 `RUN_PAUSED` (testul de repornire). Orice alt status îl întoarce: verificați `out_StatusCode` după fiecare apel.

Argumentele de care nu aveți nevoie (de exemplu `out_FilePath`) pot rămâne nelegate: Studio afișează avertismentul „Missing argument”, nu o eroare.

## Capcane în expresiile C#

Verificate în UiPath Studio 2026.0, pe un proiect Cross-platform.

- `int / int` taie zecimalele fără nicio eroare: `7 / 2` dă 3. Pentru rotunjirea în sus scrieți `Math.Ceiling((decimal)a / b)`.
- Sume: `decimal.Parse(s, CultureInfo.InvariantCulture)`, nu `Convert.ToDecimal(s)`: pe Windows în română, `Convert.ToDecimal("1234.50")` dă 123450. Formatați cu `x.ToString("0.00", CultureInfo.InvariantCulture)`; `$"{x}"` folosește cultura calculatorului.
- Câmpurile: `x.Value<string>("cheie")`; un câmp `null` dă `null`. Comparați text cu text: `x.Value<string>("sku") == "B"`, nu `x["sku"] == "B"` (eroarea CS0019).
- Expresiile sunt arbori de expresii, deci nu merg: `?.`, `out var`, `x switch { ... }`, `is null`, lambda cu acolade (`t => { return ...; }`) și argumentele opționale omise. `"a,b".Split(',')` nu compilează: scrieți `"a,b".Split(new[] { ',' })`.
- Merg: LINQ pe `JToken` și `JArray` (`First(t => ...)`, `Count(t => ...)`), `??`, `new JObject(new JProperty("sku", "A")).ToString(Formatting.None)`.
- Adăugați într-o listă sau într-un `JArray` cu Append Item to List.

## Rulare

Din Studio: `python bpa_runner.py --project A --suite public --manual`, puneți `BASE_URL` ca valoare implicită a lui `in_BaseUrl`, rulați `Main.xaml`, apoi apăsați Enter.

Din linia de comandă, cu [uip CLI](https://docs.uipath.com/coding-agents/standalone/latest/user-guide/install-and-set-up) instalat (Node.js 22 sau mai nou, apoi `npm install -g @uipath/cli`) și `uip login` făcut, runner-ul pornește robotul singur:

```bash
python bpa_runner.py --project A --suite public -- uip rpa run --file-path Main.xaml --project-dir "CALEA_PROIECTULUI" --input-arguments "in_BaseUrl={base_url}"
```

`CALEA_PROIECTULUI` este calea completă a dosarului proiectului. Fiecare pornire compilează întâi proiectul (20-100 s). La testul de repornire puneți înainte de `--`: `--kill-after-writes 5 --invocations 2 --stop-command "uip rpa execution cancel --project-dir 'CALEA_PROIECTULUI'"` (în `--stop-command` calea stă între apostrofuri, nu între ghilimele). Robotul se oprește singur la `RUN_PAUSED`, iar `--stop-command` îl oprește și dacă runner-ul ajunge primul: fără el, un robot oprit de runner continuă să ruleze în UiPath.

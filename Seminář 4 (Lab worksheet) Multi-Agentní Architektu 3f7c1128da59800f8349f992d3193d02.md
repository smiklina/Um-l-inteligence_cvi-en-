# Seminář 4 (Lab worksheet): Multi-Agentní Architektury, Multi-Model Routing & Agentic RAG

**Předmět:** ENC-AI - Umělá inteligence

**Téma:** Multi-Agentní Architektury, Multi-Model Routing & Agentic RAG

---

<aside>
<img src="https://app.notion.com/icons/light-bulb_gray.svg" alt="https://app.notion.com/icons/light-bulb_gray.svg" width="40px" />

### Cíl cvičení

Osvojit si konstrukci autonomních agentních smyček (*Reasoning + Acting*) a víceagentních týmů s využitím pokročilého prompt engineeringu, strukturovaných XML oddělovačů a definic systémových rolí.

</aside>

---

## Úkol 1.1: Návrh ReAct System Promptu s detekcí zacyklení (25 Minut)

Vaším úkolem je sestavit produkční **System Prompt** pro diagnostického agenta. Agent musí dodržovat cyklus `<thought>`, `<action>`, `<observation>` a být odolný vůči nekonečným smyčkám.

### Požadované vlastnosti promptu:

1. Striktní definice 3 dostupných pseudonástrojů: `vyhledat_v_logu(pattern)`, `restartovat_sluzbu(service_name)`, `dokoncit(sprava)`.
2. Pravidlo pro maximálně 4 kroky uvažování.
3. Explicitní detekce zacyklení: Pokud je výsledek v `<observation>` shodný s předchozím krokem, agent **nesmí** opakovat stejnou akci.

#### Sem doplňte váš návrh System Promptu:

```xml
<!-- VAŠE ŘEŠENÍ: Kompletní ReAct System Prompt -->
<system_instructions>
Jsi diagnostický agent pro produkční infrastrukturu. Tvým cílem je zjistit příčinu
nedostupnosti služby, pokud je to možné ji obnovit, a výsledek shrnout ve zprávě.
Pracuješ v cyklu ReAct: uvažuj (<thought>) → jednej (<action>) → počkej na výsledek
(<observation>), který ti dodá prostředí. Jednáš opatrně, věcně a nikdy si nevymýšlíš
data, která jsi nezískal z nástrojů.
</system_instructions>

<dostupne_nastroje>
1. vyhledat_v_logu(pattern: string)
   - Prohledá systémové logy podle textového vzoru nebo regulárního výrazu.
   - Vrací odpovídající řádky logu, nebo "NO_MATCH".
   - Příklad: vyhledat_v_logu("AuthService.*ERROR")

2. restartovat_sluzbu(service_name: string)
   - Restartuje uvedenou službu. Vrací nový stav služby (RUNNING / FAILED).
   - Smí být použit nejvýše JEDNOU pro každou službu a jen tehdy, když log
     ukazuje na chybu, kterou restart může vyřešit (TIMEOUT, HANG, OOM).
   - Příklad: restartovat_sluzbu("AuthService")

3. dokoncit(sprava: string)
   - Ukončí diagnostiku a odešle závěrečnou zprávu: zjištěná příčina,
     provedené kroky, aktuální stav služby, doporučení.
   - Po zavolání tohoto nástroje už negeneruj žádný další výstup.

Jiné nástroje neexistují. Nevolej žádný nástroj, který zde není uveden.
</dostupne_nastroje>

<pravidla_a_omezeni>
FORMÁT REACT:
- Každý krok obsahuje přesně jeden blok <thought> a přesně jeden blok <action>.
- Blok <observation> NIKDY negeneruješ sám. Po <action> se zastav a čekej na prostředí.
- Své úvahy zakládej výhradně na obsahu přijatých <observation>.

LIMIT KROKŮ:
- Máš k dispozici maximálně 4 kroky. Každý <thought> začíná počítadlem "Krok N/4".
- Krok 4 musí být VŽDY akce dokoncit(), i když problém není vyřešen.
  V takovém případě ve zprávě uveď, co zůstalo neobjasněné, a doporuč eskalaci.
- Pokud je problém vyřešen dříve, zavolej dokoncit() hned.

DETEKCE ZACYKLENÍ:
- Na začátku každého <thought> porovnej poslední <observation> s tou předchozí.
- Pokud jsou obsahově shodné, napiš do <thought> větu
  "DETEKCE SMYČKY: observation se shoduje s předchozím krokem."
- Po detekci smyčky NESMÍŠ zopakovat stejnou akci (stejný nástroj se stejným
  argumentem) ani akci se stejným účelem. Musíš:
  a) zvolit jiný nástroj, nebo
  b) výrazně změnit argument (jiný vzor, jiná služba), nebo
  c) zavolat dokoncit() se zprávou, že diagnostika uvízla.
- Nikdy neopakuj akci, kterou jsi už v této konverzaci provedl se stejnými argumenty.

BEZPEČNOST:
- restartovat_sluzbu() použij až poté, co log potvrdil chybu. Nikdy ne naslepo
  jako první krok.
</pravidla_a_omezeni>

<format_vystupu>
Každý krok musí přesně odpovídat této struktuře:

<thought>
Krok N/4. [Porovnání s předchozí observation / případná DETEKCE SMYČKY]
[Co víme, jaká je hypotéza, proč volím následující akci.]
</thought>
<action>nazev_nastroje("argument")</action>

Poté se zastav a čekej na <observation> od prostředí.
</format_vystupu>
```

**Popis návrhu:** Prompt řeší tři typická rizika ReAct agentů. (1) *Halucinace výsledků nástrojů:* agent má výslovně zakázáno generovat `<observation>` sám. (2) *Nekonečná smyčka:* dvojí pojistka, tedy tvrdý limit 4 kroků (4. krok je vždy `dokoncit`) a povinné porovnání observation s předchozí. Shodu musí agent nejdřív explicitně vyslovit („DETEKCE SMYČKY…“) a teprve pak změnit strategii. (3) *Nebezpečné zásahy:* restart je povolen jen jednou na službu a jen po potvrzení chyby v logu. Počítadlo „Krok N/4“ v každém `<thought>` pomáhá modelu limit hlídat a striktní formát umožňuje výstup strojově parsovat.

---

### Testovací simulace (Peer-to-Peer Review)

Ve dvojici se sousedem nasimulujte běh agenta v konverzačním rozhraní (jeden student hraje úlohu Agenta, druhý představuje Prostředí a doplňuje `<observation>`).

- **Testovací scenář:** Služba `AuthService` neodpovídá a log v prvním kroku vrací chybu `TIMEOUT`.
- **Záznam testovacího běhu (vložte získaný log):**

V kroku 2 prostředí záměrně vrátilo stejnou observation, abychom ověřili detekci zacyklení.

```xml
<thought>
Krok 1/4. Žádná předchozí observation neexistuje. Služba AuthService neodpovídá.
Nejprve potřebuji zjistit, co o ní říkají logy, a nic nerestartovat naslepo.
</thought>
<action>vyhledat_v_logu("AuthService.*(ERROR|TIMEOUT)")</action>
<observation>[2026-10-13 12:58:41] AuthService ERROR: request TIMEOUT after 30000 ms</observation>

<thought>
Krok 2/4. Log potvrzuje TIMEOUT. Chci ověřit, zda příčinou není nedostupná
závislost (databáze), protože pak by restart AuthService nepomohl.
</thought>
<action>vyhledat_v_logu("AuthService.*(database|connection)")</action>
<observation>[2026-10-13 12:58:41] AuthService ERROR: request TIMEOUT after 30000 ms</observation>

<thought>
Krok 3/4. DETEKCE SMYČKY: observation se shoduje s předchozím krokem.
Další hledání v logu nepřináší nové informace, proto ho nebudu opakovat.
Žádná chyba databáze ani spojení se nenašla, příčina je tedy pravděpodobně
v samotné službě (zaseknutý proces). Log chybu potvrdil, restart je odůvodněný.
</thought>
<action>restartovat_sluzbu("AuthService")</action>
<observation>AuthService restarted. Status: RUNNING</observation>

<thought>
Krok 4/4. Observation je nová (RUNNING), smyčka nenastala. Služba běží,
diagnostiku ukončuji závěrečnou zprávou.
</thought>
<action>dokoncit("Příčina: AuthService neodpovídala (TIMEOUT 30 s). Logy neukázaly chybu databáze ani spojení, šlo pravděpodobně o zaseknutý proces služby. Provedeno: 2× analýza logu, 1× restart. Stav: RUNNING. Doporučení: sledovat výskyt TIMEOUT v příštích 24 h; pokud se zopakuje, eskalovat na vývojový tým kvůli možnému úniku zdrojů.")</action>
```

**Vyhodnocení simulace:** Ve 3. kroku agent rozpoznal shodnou observation a místo dalšího hledání v logu změnil strategii, takže pravidlo zacyklení funguje. Restart proběhl až po potvrzení chyby v logu a agent se vešel přesně do 4 kroků zakončených akcí `dokoncit`.

---

## Úkol 1.2: Multi-Agent Collaboration Protocol (25 Minut)

Navrhněte protokol předávání správ (*Handoff Protocol*) pro tří-agentní vývojový tým:

- **Agent 1 (Architect):** Přijme požadavek a vytvoří XML specifikaci rozhraní.
- **Agent 2 (Developer):** Vygeneruje kód v Pythonu na základě specifikace.
- **Agent 3 (Security Auditor):** Prověří kód na zranitelnosti a buď schválí (`APPROVED`), nebo vrátí kód Vývojáři (`REJECTED`).

```
[System Architect] ──► [Senior Developer] ──► [Security Auditor] ──┐
                              ▲                                    │ (REJECTED)
                              └────────────────────────────────────┘
```

#### Sem doplňte návrh System Promptu pro Agenta 3 (Security Auditor):

```xml
<!-- VAŠE ŘEŠENÍ: System Prompt pro Security Auditora -->
<system_instructions>
Jsi Security Auditor ve tříčlenném vývojovém týmu (Architect → Developer → Auditor).
Tvým jediným úkolem je bezpečnostní revize Python kódu, který ti předá Developer.
Kód NEPÍŠEŠ ani neopravuješ. Hledáš zranitelnosti, hodnotíš jejich závažnost
a vydáš rozhodnutí APPROVED nebo REJECTED s konkrétním, opravitelným zdůvodněním.

Kontroluješ zejména (podle OWASP Top 10 a CWE):
- injekce (SQL, příkazy OS přes os.system / subprocess se shell=True, eval/exec),
- natvrdo zapsaná tajemství (hesla, API klíče, tokeny),
- chybějící validaci a sanitizaci vstupů, path traversal,
- nebezpečnou deserializaci (pickle, yaml.load bez SafeLoader),
- slabou kryptografii (MD5/SHA1 pro hesla, random místo secrets),
- únik citlivých dat v logách a chybových hláškách,
- nesoulad kódu se specifikací rozhraní od Architekta.

Hodnotíš pouze předaný kód a specifikaci. Nevymýšlíš si soubory ani závislosti,
které v předaných datech nejsou. Konverzační historii ostatních agentů nevidíš
a nepotřebuješ.
</system_instructions>

<vstupni_schema>
Od Developera přijímáš výhradně tuto XML strukturu:

<handoff from="developer" to="security_auditor">
  <iteration>1</iteration>                      <!-- číslo pokusu, max. 3 -->
  <specification><!-- XML specifikace rozhraní od Architekta --></specification>
  <code language="python" filename="service.py">
    <![CDATA[ ... zdrojový kód ... ]]>
  </code>
  <dependencies>requests==2.32.3, ...</dependencies>
  <developer_notes>Co bylo změněno oproti minulé iteraci.</developer_notes>
</handoff>

Pokud vstup tomuto schématu neodpovídá (chybí <code> nebo <specification>),
vrať REJECTED s kategorií "INVALID_INPUT" a kód neanalyzuj.
</vstupni_schema>

<pravidla_rozhodovani>
KLASIFIKACE ZÁVAŽNOSTI:
- CRITICAL: vzdálené spuštění kódu, injekce, natvrdo zapsaná tajemství.
- HIGH: chybějící validace vstupu, nebezpečná deserializace, slabá kryptografie.
- MEDIUM: únik informací v logu/chybách, chybějící timeouty u síťových volání.
- LOW: stylistická a obranná doporučení (best practices).

ROZHODNUTÍ:
- APPROVED: v kódu není žádné zjištění CRITICAL ani HIGH a kód odpovídá
  specifikaci. Nálezy MEDIUM/LOW uveď jako doporučení.
- REJECTED: existuje alespoň jedno zjištění CRITICAL nebo HIGH, nebo kód
  porušuje specifikaci rozhraní. Každé zjištění musí obsahovat řádek,
  popis, CWE a konkrétní návrh opravy.
- ESKALACE: pokud <iteration> = 3 a výsledek by byl opět REJECTED, vrať
  stav ESCALATED a předej případ Architektovi / člověku (ochrana proti
  nekonečné smyčce Developer ↔ Auditor).

VÝSTUP (vždy jen tento XML, žádný další text):

<audit_result to="developer">
  <status>APPROVED | REJECTED | ESCALATED</status>
  <iteration>N</iteration>
  <findings>
    <finding severity="CRITICAL" cwe="CWE-89" line="42">
      <description>SQL dotaz skládán z uživatelského vstupu přes f-string.</description>
      <fix>Použij parametrizovaný dotaz: cursor.execute("... WHERE id = %s", (user_id,)).</fix>
    </finding>
  </findings>
  <summary>Jedna až dvě věty pro Developera.</summary>
</audit_result>
</pravidla_rozhodovani>
```

**Popis návrhu:** Auditor má úzce vymezenou roli (jen hodnotí, nepíše kód), aby se nemíchaly odpovědnosti agentů. Vstup i výstup jsou striktní XML, takže se mezi agenty předává jen to podstatné (specifikace, kód, poznámky), nikoli celá historie. Rozhodnutí stojí na jasném prahu závažnosti (CRITICAL/HIGH → REJECTED), takže je opakovatelné. Každý nález musí obsahovat konkrétní opravu, aby Developer věděl, co změnit. Počítadlo `<iteration>` s limitem 3 a stav `ESCALATED` brání nekonečné smyčce REJECTED ↔ oprava, podobně jako limit kroků v Úkolu 1.1.

---

## Úkol 1.3: Analytické otázky k Labu 1 (10 Minut)

1. **Vysvětlete pojmy *Context Pollution* a *Attention Dispersion* v kontextu víceagentních systémů. Proč je výhodnější předávat mezi agenty pouze strukturovaný JSON/XML výstup namísto kompletní konverzační historie?**

**Context Pollution** (znečištění kontextu) nastává, když se do kontextového okna agenta dostanou informace, které pro jeho úlohu nejsou relevantní: úvahy jiných agentů, slepé uličky, zamítnuté verze kódu, staré chyby nebo instrukce určené jiné roli. Model je pak může mylně považovat za platné (např. Auditor začne hodnotit kód z první iterace nebo převezme styl uvažování Developera), roste riziko halucinací a míchání rolí.

**Attention Dispersion** (rozptýlení pozornosti) vyplývá z mechanismu self-attention. Softmax rozděluje váhu pozornosti mezi všechny tokeny v kontextu, takže čím delší a zašuměnější kontext, tím menší podíl připadá na skutečně důležité tokeny. Kvalita se zhoršuje zvlášť u informací uprostřed dlouhého kontextu (efekt „lost in the middle“).

**Proč strukturovaný JSON/XML:** Strukturovaný výstup funguje jako jasně definované rozhraní mezi agenty. Předává jen výsledek (specifikaci, kód, verdikt), ne cestu k němu. Kontext je proto krátký a relevantní (méně tokenů, nižší cena a latence, lepší pozornost), každý agent dostává jen data pro svou roli (oddělení odpovědností), výstup lze strojově validovat proti schématu a parsovat bez chyb a systém je lépe laditelný a auditovatelný, protože u každého předání je vidět, co přesně prošlo.

2. **Jakým způsobem generování tokenů v bloku `<thought>` ovlivňuje výpočetní kapacitu věnovanou řešení úlohy autoregresivním modelem $P(w_t \mid w_{<t})$?**
Autoregresivní model generuje každý token jedním průchodem sítí s pevným počtem vrstev, takže na jeden token připadá zhruba konstantní množství výpočtu. Kdyby měl model odpovědět rovnou, musel by celé řešení „spočítat“ v jediném průchodu, což u vícekrokových úloh nestačí.

Tokeny v bloku `<thought>` fungují jako **externí pracovní paměť (scratchpad)**. Každý vygenerovaný token se stane součástí podmínky $w_{<t}$ pro všechny další tokeny, takže model může mezivýsledky zapsat a v dalších krocích na ně navázat. Počet průchodů sítí, a tedy i celkový výpočet věnovaný úloze, roste **lineárně s délkou uvažování**. Jde o tzv. *test-time compute*: delší úvaha = více výpočtu nad stejným problémem. Složitý problém se tím rozloží na posloupnost jednodušších podmíněných predikcí, z nichž každá je pro model snáze řešitelná.

Důsledek: Chain-of-Thought / ReAct zlepšuje přesnost u úloh vyžadujících více kroků, ale za cenu vyšší latence a počtu tokenů. Kvalita výsledku zároveň závisí na kvalitě úvah: chyba zapsaná v `<thought>` se dál podmiňuje a šíří, proto je důležité úvahy zakládat na skutečných `<observation>` (viz Úkol 1.1).

---

## Kontrolní seznam pro odevzdání (Checklist)

- [x]  Vyplněn Úkol 1.1 (ReAct System Prompt & Log simulace)
- [x]  Vyplněn Úkol 1.2 (System Prompt pro Security Auditora)
- [x]  Zodpovězeny analytické otázky v Úkolu 1.3

---

**Krátká reflexe na cvičení - díky za pomoc!**

- Který úkol nebo koncept pro vás byl dnes nejjednodušší? 
Nejjednodušší byl ten úkol 1. Pak tu testovací část bych chtěla lépe uvedenou. jestli jsme měli dávat odpověď my nebo si jen nasdílet prompt a otestovat si to navzájem na nějakém LLM modelu. Musím ale uznat, že dnešní zadání úkolu je mnohonásobně lepší než z minulých cvičení, vidím důvod proč děláme a je dobře návodné, abychom se mohli soustředit na to co děláme a ne na to abychom vůbec pochopili zadání. 
- Která implementace kódu nebo statistický koncept pro vás představovaly největší výzvu? 
Pro mě osobně je nejtěžší zadání to, které obsahuje nějaký kód a mám pracovat pouze s ním. (nemyslím tím XML tagy, ale třeba minulé cvičení, kde jsme měli něco dělat  v Colabu, napojit to na Gemini a tam něco kontrolovat pomocí temperature a nějak hodnotit -> tenhle úkol jsem nepochopila a jelikož jsem se nedostala ani pořádně k výsledkům tak mi moc nedal, protože jsem nepochopila ani zadání, ani provedení a nakonec bych uvítala aspoň zobrzaení výsledků, abych aspoň tušila co mělo být výsledkem, ale ani to neproběhlo a to mě trochu demotivovalo) 

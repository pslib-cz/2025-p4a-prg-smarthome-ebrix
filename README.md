# Ebrix (IoT Simulace ohřevu šamotových cihel)

Tento projekt je případovou studií v rámci předmětu Internetu věcí (IoT) s využitím **Home Assistant**. Cílem je simulovat reálný scénář průmyslové/chytré domácnosti – řízení a monitorování nabíjení teplotního média (šamotové cihly), které se využívá v akumulačních kamnech. Systém je plně lokální a logicky odpovídá reálným fyzikálním procesům.

## 👥 Řešitelský tým
Na projektu pracuje tříčlenná skupina (**$N = 3**):
- **Radomír Mendřický**
- **Jan Kočnar**
- **Tomáš Kubíček**

**Splnění podmínek zadání pro $N=3:**
- **Min. lokálních integrací ($N-1 = 2):** Využito **ESPHome** pro lokální řízení a sběr dat z HW. Dále je využita komunitní správa **HACS** a integrace **CZ Energy Spot Prices** pro dynamické posuzování výhodnosti nabíjení podle aktuálních cen elektřiny na spotovém trhu.
- **Min. počet scén/automatizací ($N = 3):** Projekt obsahuje 4 řídicí scénáře (Nabíjení, Nabito, Přímý ohřev, Vybíjení).
- **Min. počet entit ($N*2 = 6):** Splněno několikanásobně – viz výčet HW. Možnost přepínání scén bude vytvořena jako pomocná (Helper) entita typu input_select pro UI.

---

## 🧠 Řídicí logika a architektura (Konfigurace File-only)
*Vše řešíme plně přes yaml konfigurační soubory a klon configu!*

- **Centrální mozek:** Home Assistant (lokální řízení, automatizace, UI dashboard). Vyhodnocuje stavy a přepíná režimy podle stavu "nabíjení".
- **Edge zařízení:** ESP32-C3 SuperMini rozběhnuté přes **ESPHome** pro přímou integraci do HA.
  - Sběr dat po OneWire (Teploměry), I2C (Prostředí) a UART (PZEM spotřeba).

## 👁️ Snímače a senzory (Vstupy/Entity)
- **Teplota média - Core / Out (DS18B20):** 2 vodotěsné kovové teplotní sondy integrované přímo do šamotu a na výstup.
- **Měření spotřeby (PZEM-004T-100A v3.0):** Měří napětí 230V, proud, výkon a frekvenci. Poskytuje data o energetické náročnosti ohřevu.
- **Vlivy prostředí (AHT20 + BMP280):** I2C senzor pro měření teploty místnosti, vlhkosti a tlaku vzduchu.
- *(Volitelně: IR senzor plamene jako fail-safe bezpečnostní prvek).*

## 🦾 Akční členy (Výstupy/Entity)
- **Topná soustava (Relé moduly 1 & 2):** Spínání PTC ohřívače (400W/230V) a jeho primárního ventilátoru.
- **Proudění vzduchu (Relé / Servo):** Otevírání / zavírání výstupní klapky boxu s cihlami.
- **Aktivní vybíjení (Malý DC větráček):** Sekundární odtahový ventilátor pro vybíjení (rozfuk) tepla do okolí ("fabriky").

---

## ⚙️ Provozní Scénáře (Přepínatelné z Dashboardu) 
Stavy a chování pece se kompletně řídí povely z webového rozhraní (dashboardu). Automatizace nezasahuje do fyzického přepínání relé na základě teploty sama od sebe – změna scénáře probíhá vždy **ručně** kliknutím uživatele.
Máme 4 základní scénáře, které přesně odpovídají reálnému využití (usecase):

### 1. ⚡ NABÍJENÍ (Charging)
Režim uložení energie do šamotu. Typicky se aktivuje, pokud je Core teplota **pod 80 °C**.
- **Ohřev a hlavní větrák:** ZAPNUTO (Obě relé ON)
- **Klapka boxu:** UZAVŘENA
- **Odtah (malý větráček):** VYPNUTO

### 2. 🔋 NABITO (Charged / Idle)
Tento stav slouží primárně jako notifikační a klidový. Jakmile Core teplota při nabíjení dosáhne cílových **80 °C**, systém v dashboardu **pouze vizuálně** zahlásí "NABITO" (bez přerušení sepnutých relátek). Uživatel pak na základě této informace ručně přepne pec do klidového nebo vybíjecího režimu.

### 3. 🔥 PŘÍMÝ OHŘEV / PRŮBĚŽNĚ (Pass-through Heating)
Ideální případ, když máme k dispozici přebytečnou energii, ale zároveň je rovnou potřeba teplo ve výrobě/fabrice. Teplý vzduch se generuje a ihned se žene ven za pomoci všech systémů.
- **Ohřev a hlavní větrák:** ZAPNUTO (Obě relé ON - topíme)
- **Klapka boxu:** OTEVŘENA
- **Odtah (malý větráček):** ZAPNUTO (Točí se dopředu pro maximální odtah a průtok vzduchu ven)

### 4. 🌬️ VYBÍJENÍ (Discharging)
Aktivuje se v momentě, kdy je ohřev přes elektřinu příliš drahý nebo nepotřebný, ale cihla je nabitá a my chceme místnost temperovat jen pomocí uloženého tepla.
- **Ohřev a hlavní větrák:** VYPNUTO (Obě hlavní relé OFF - ohřívač netopí)
- **Klapka boxu:** OTEVŘENA
- **Odtah (malý větráček):** ZAPNUTO (Točí se dopředu a vyfukuje uložené teplo z cihel do místnosti)

---

## 💻 Integrace a Dashboard
- **Monitoring a Interaktivní Dashboard** dostupný přes web klienta. Umožnuje vidět live spotřebu z PZEM-004T (grafy nabíjení), aktuální teploty z Dallas Core a Out sond a vizuálně přepínat scénáře.
- Veškeré spínání probíhá lokálně bez závislostí mimo Home Assistant (žádný cloud API třetích stran).

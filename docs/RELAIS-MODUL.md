# 🔌 Relais-Module aus der Bastelkiste – Beschreibung & Integration

**Version:** 0.1
**Status:** Komponentenbeschreibung, vor Einsatz am konkreten Board prüfen (Checkliste Abs. 8)
**Quelle:** Produktbeschreibung (PDF, beigelegt), Relais-Aufdruck am Modul, Hersteller-Datenblätter

---

## 1. Identifikation

Es handelt sich um chinesische Standard-Relais-Module (Bastelkiste, vermutlich AliExpress «BESTER»), in 1-Kanal- und 2-Kanal-Ausführung:

| Merkmal | Wert |
|---|---|
| Relais-Typ (Aufdruck) | **JQC-3F-03VDC-C** (BESTER) |
| Spulennennspannung | **3 VDC** (Widerstand 25 Ω ±10 %, ~0.36 W) |
| Kontaktausführung | **1C = Wechsler (SPDT)** → Klemmen NO / COM / NC |
| Kontaktbelastbarkeit | 10 A / 250 VAC, 10 A / 30 VDC (modulabhängig, Deckel-Aufdruck) |
| Treiber auf dem Modul | S8550 (PNP), Low-Level-Trigger **IN = LOW → Relais EIN** |
| Anschluss | KF301-Schraubklemme (Kontakte), Stiftleiste VCC / GND / IN |
| LEDs | Power (grün), Status (rot) |
| Besonderheit 2-Kanal-Modul | Jumper «VCC-JD» (Funktion am konkreten Board prüfen) |

> ⚠️ **Die Produktbeschreibung nennt «Input voltage 3V–5V». Das gilt für die Logik-Kompatibilität, **nicht** für die Versorgungsspannung der Spule: Das aufgeklebte Relais hat eine **3-V-Spule**. Bei VCC = 5 V läge die Spule bei 167 % der Nennspannung → ca. 1 W statt 0.36 W, Überhitzung vorprogrammiert.

---

## 2. Eckdaten des Relais (JQC-3F / T73-Familie, 3-VDC-Version)

| Parameter | Wert |
|---|---|
| Spulenwiderstand | 25 Ω ±10 % |
| Nennstrom Spule @ 3 V | 120 mA |
| Strom bei 3.3 V (rechnerisch) | **~132 mA pro Relais** |
| Ansprechspannung (max.) | 2.25 V (75 % Un) |
| Abfallspannung (min.) | 0.15 V |
| Max. Dauerspannung | 110 % Un = **3.3 V** (exakt unsere Rail-Spannung) |
| Ansprech-/Abfallzeit | ≤ 10 ms / ≤ 5 ms |
| Kontaktmaterial | Silberlegierung (AgSnO₂/AgCdO je nach Quelle) |
| Isolation Kontakt–Spule | 1500 VAC |
| Elektrische Lebensdauer | ~10⁵ Schaltspiele (bei Nennlast) |

Quellen: Hersteller-Spezifikationen JQC-3F/T73 (Shenler/Hungso/DBLECTRO), siehe Abs. 9.

---

## 3. Ansteuerung durch den Raspberry Pi – «direkt» funktioniert, aber mit Bedingungen

**Pauls Einschätzung ist richtig:** Diese Module können vom Pi **direkt** (ohne ULN2003A) angesteuert werden, **wenn** folgende drei Bedingungen eingehalten werden:

### 3.1 VCC = 3.3 V (Pflicht)

| VCC | Logikfolge | Bewertung |
|---|---|---|
| **3.3 V** | GPIO HIGH (3.3 V) = VCC → S8550 sperrt sauber; GPIO LOW (0 V) → S8550 leitet | ✅ **einzige saubere Variante** |
| 5 V | GPIO HIGH (3.3 V) liegt 1.7 V unter VCC → S8550 leitet **weiterhin** → Relais lässt sich nicht abschalten | ❌ Logikfehler |
| 5 V | zusätzlich: 3-V-Spule bei 167 % Un | ❌ Zerstörung der Spule (Überhitzung) |

### 3.2 Low-aktiv-Logik (Pflicht)

- **IN = LOW → Relais EIN**, IN = HIGH → Relais AUS.
- Für die Firmware heisst das: Protokollbefehl `RELn=ON` → GPIO n = **LOW** (invertierte Logik, z. B. Konstante `RELAY_ACTIVE_LOW = True`).
- Watchdog-SAFE-Zustand (alles AUS) = alle IN auf HIGH.

### 3.3 Strombudget der Spulen (Pflicht)

4 Relais gleichzeitig EIN: 4 × ~132 mA = **~528 mA @ 3.3 V** (+ ~10 mA Basisströme über die GPIOs).

- Die 3.3-V-Rail des Pi Zero W ist dafür **zu knapp** (dort hängt auch der RAM/WLAN-Bereich).
- **Empfehlung:** eigener Step-Down-Regler 5 V → 3.3 V (≥ 1 A) nur für die Relais-Module; GND gemeinsam mit dem Pi. Damit bleibt das GPIO-Interface (nur ~2–3 mA pro Pin) unkritisch.

### 3.4 Basisstrom

Der GPIO muss im EIN-Zustand nur den S8550-Basisstrom (über den Vorwiderstand des Moduls) senken: typ. **~2–3 mA pro Kanal** – unkritisch für den Pi (Limit 16 mA/Pin).

---

## 4. Definierter Zustand beim Boot (Start-Sicherheit)

- Der Pi konfiguriert GPIOs erst spät beim Boot; davor sind sie Eingänge (teils mit internem Pull).
- **Massnahme:** externer Pull-up **10 kΩ von IN nach 3.3 V** je Kanal → Relais bleibt beim Boot sicher AUS (Low-aktiv!).
- Zusätzlich im Python-Code: GPIOs **so früh wie möglich** als Output mit Initialwert HIGH setzen.

---

## 5. Auswirkung auf das Power-Budget (Änderung zur Projektstudie!)

Die Projektstudie rechnete mit Reed-Relais (~10 mA). Mit diesen EM-Modulen gilt:

| Szenario | Relais-Strom | Gesamtleistung (ca.) | Energie/Tag |
|---|---|---|---|
| Alle 4 AUS | ~0 mA | ~0.64 W (wie Studie) | ~15 Wh |
| 1 Relais EIN | ~132 mA | ~1.1 W | ~26 Wh |
| 4 Relais EIN | ~528 mA | **~2.4 W** | **~57 Wh (≈1.7 kWh/Monat)** |

- Bei Netzbetrieb (Netzteil ≥ 2 A) unproblematisch.
- Bei Solar/Akku-Betrieb wäre der Unterschied **wesentlich** → dort wären die ursprünglich geplanten Reed-Relais vorzuziehen (Offenpunkt).

---

## 6. Vorteile gegenüber den geplanten Reed-Relais (Standex SIL)

| Aspekt | Bastelkisten-Modul (JQC-3F) | Reed-Relais (Standex SIL 05-1A72-71D) |
|---|---|---|
| Kontaktleistung | 10 A / 250 VAC | ~0.5 A / 10 W (nur Signal/ Kleinlast) |
| Kontaktform | Wechsler (NO + **NC**) | nur Öffner oder Schliesser (Form A) |
| Spulenstrom | ~132 mA @ 3.3 V | ~10 mA |
| Treiber | **onboard (S8550)** – kein ULN2003A nötig | extern (ULN2003A) nötig |
| Preis | bereits vorhanden | zu beschaffen |
| Lebensdauer | ~10⁵ Spiele bei Nennlast | >10⁷ (Reed, nahezu verschleissfrei) |

**Konsequenz für den Schaltplan:** ULN2003A entfällt als Pflichtbaustein (nur als Reserve/Fallback für zukünftige 5-V-Module im Design behalten). Der **NC-Kontakt** erlaubt fail-safe Verdrahtung: Last wahlweise an NO (Relais EIN = an) oder NC (Relais EIN = aus / Stromlos = an) – je nach gewünschtem Verhalten bei Watchdog/SAFE.

---

## 7. Sicherheit / Lastseite

- 230 VAC an den KF301-Klemmen nur mit den üblichen Vorsichtsmassnahmen (berührungssicher, Trennung, keine spannungsführenden Teile offen zugänglich). Für die ersten Versuche und den Loopback-Test **Kleinspannung** (12/24 V) verwenden.
- Induktive Lasten (Magnetventile, Motoren): RC-Snubber (z. B. 100 Ω + 0.1 µF) oder VDR parallel zum Kontakt – steht so auch in der Produktbeschreibung.
- Keine galvanische Trennung zwischen Logik und Last (Modul ohne Optokoppler) – für unser Konzept (gemeinsame Masse) akzeptabel.

---

## 8. Prüf-Checkliste für die Bastelkiste-Exemplare (vor Einsatz)

- [ ] Relais-Aufdruck verifizieren: wirklich **03VDC**? (5VDC-Varianten sehen identisch aus!)
- [ ] Spulenwiderstand messen (Soll ~25 Ω, ±10 %) → Rechenstrom bei 3.3 V bestätigen
- [ ] Freilaufdiode auf dem Modul vorhanden? (bei fehlender Diode würde der S8550 durch Induktionsspitzen leiden – meist bestückt, prüfen)
- [ ] Pull-up auf IN vorhanden? (wenn nein: externen 10 kΩ nach 3.3 V einplanen – sowieso empfohlen)
- [ ] Verhalten bei schwebendem IN testen (muss AUS bleiben)
- [ ] 2-Kanal-Modul: Funktion des Jumpers «VCC-JD» klären (bei 3.3-V-Betrieb i. d. R. geschlossen lassen)
- [ ] Stromaufnahme pro Kanal bei 3.3 V messen (Soll ~130 mA)

---

## 9. Quellen

- Produktbeschreibung des Moduls (PDF, beigelegt in diesem Projekt)
- JQC-3F-Spezifikation (Shenler) und JQC-3F(T73)-Datenblatt (Hungso) für Spulendaten 3 VDC: 25 Ω, 120 mA, Ansprechen ≤ 2.25 V ([Sharvi Electronics JQC3F PDF](https://sharvielectronics.com/wp-content/uploads/2025/09/JQC3F-C-S-DC9V.pdf), [Hungso JQC-3F(T73) Specification](https://www.hungso.com/Content/upload/pdf/202127180/JQC-3F-T73-Specification.pdf), [DBLECTRO NT73/JQC-3FC](https://www.alldatasheet.com/datasheet-pdf/view/157071/DBLECTRO/JQC-3FC.html))

---

## 10. Offene Punkte

- [ ] Wie viele 1-/2-Kanal-Module liegen tatsächlich in der Bastelkiste? (für die Schaltplan-Topologie)
- [ ] Gemeinsamer externer 3.3-V-Regler: Typ wählen (z. B. Step-Down-Modul MP1584/mini360)
- [ ] Verdrahtung Last an NO oder NC je Kanal festlegen (fail-safe-Strategie, Abs. 6)
- [ ] Power-Budget in README/Studie konsolidieren, sobald feststeht, wie viele Relais dauerhaft EIN sind

---

Autor: Paul Simmen mit Vibe (GLM-5-latest-short), 28.10.2026

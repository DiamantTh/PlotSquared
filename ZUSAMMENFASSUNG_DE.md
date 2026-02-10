# Zusammenfassung: Architektur-Analyse Core/Bukkit für Folia-Unterstützung

## Aufgabenstellung

Die Aufgabe war, die Ordnerstruktur und Architektur von Bukkit und Core zu überprüfen:
1. Ob im Core wirklich nur plattformunabhängiger Code enthalten ist
2. Ob man theoretisch Folia-Unterstützung als separates Modul bereitstellen könnte

## Ergebnisse

### ✅ Bukkit-Modul: Saubere Architektur

Das Bukkit-Modul ist korrekt implementiert:
- Enthält nur Bukkit-spezifische Implementierungen
- Implementiert die Plattform-Interfaces aus Core korrekt
- Keine falsch platzierten Core-Logik

### ⚠️ Core-Modul: Probleme gefunden und behoben

#### Problem 1: ReflectionUtils (BEHOBEN ✅)

**Das Problem:**
- `ReflectionUtils.java` im Core-Modul enthielt Bukkit/NMS-spezifischen Reflection-Code
- Enthielt hartcodierte `org.bukkit.craftbukkit` und `net.minecraft.server` Paket-Referenzen
- Core's `PlotSquared.java` initialisierte `ReflectionUtils` mit der Version von der Plattform

**Die Lösung:**
1. Neue Klasse `BukkitReflectionUtils` im Bukkit-Modul erstellt
2. Alle Referenzen im Bukkit-Modul aktualisiert
3. Initialisierung nach `BukkitPlatform.onEnable()` verschoben
4. Alte `ReflectionUtils`-Klasse deprecated (für Rückwärtskompatibilität)
5. `PlotPlatform.serverNativePackage()` Methode deprecated

**Geänderte Dateien:**
- Neu: `Bukkit/src/main/java/com/plotsquared/bukkit/util/BukkitReflectionUtils.java`
- Geändert: `BukkitPlatform.java`, `SingleWorldListener.java`, `ChunkListener.java`
- Geändert: `Core/src/main/java/com/plotsquared/core/PlotSquared.java`
- Deprecated: `Core/src/main/java/com/plotsquared/core/util/ReflectionUtils.java`

#### Kein Problem: SERVICE_BUKKIT Konfiguration

**Analyse:**
- `Settings.UUID.SERVICE_BUKKIT` ist eine Benutzerkonfigurationsoption
- Teil eines Musters: `SERVICE_PAPER`, `SERVICE_LUCKPERMS`, `SERVICE_ESSENTIALSX`
- Wird nur von plattformspezifischem Code gelesen (Bukkit-Modul)
- Core hat keine Logik, die von diesem Wert abhängt

**Fazit:** Akzeptable Architektur. Settings sind benutzerseitig und der Plattform-Code liest sie.

## Folia-Unterstützung: Machbarkeit

### ✅ Theoretisch möglich

Nach dem Refactoring ist die Folia-Unterstützung theoretisch machbar:

1. **Neues Modul erstellen**: `Folia/` neben `Bukkit/`
   ```
   PlotSquared/
   ├── Core/          (plattformunabhängig)
   ├── Bukkit/        (Bukkit/Spigot/Paper)
   └── Folia/         (Folia-spezifische Implementierung)
   ```

2. **Folia-Modul-Struktur** (ähnlich wie Bukkit):
   - `FoliaPlatform.java` (implementiert PlotPlatform)
   - `listener/` (Folia-bewusste Event-Listener)
   - `util/` (Folia-spezifische Utilities)
   - `player/` (FoliaPlayerManager)
   - `inject/` (Dependency Injection Module)

### 🚧 Herausforderungen für Folia-Implementierung

1. **Threading-Modell**
   - Folia verwendet Regions-basierte Threads, nicht einen globalen Main-Thread
   - Operationen müssen auf der richtigen Region ausgeführt werden
   - Cross-Region-Operationen benötigen spezielle Behandlung

2. **API-Änderungen**
   - Folia hat Breaking Changes gegenüber Paper
   - Scheduler-API komplett anders
   - Entity/Chunk-Zugriffsmuster geändert

3. **Core-Abstraktionen müssen evtl. erweitert werden**
   - Aktueller `TaskManager` geht von Single-Threaded-Ausführung aus
   - Eventuell `RegionAwareTaskManager`-Abstraktion nötig
   - Location/Chunk-Operationen benötigen möglicherweise Regions-Kontext

4. **Shared State Management**
   - Plot-Daten werden von mehreren Threads zugegriffen
   - Datenbank-Operationen müssen Thread-sicher sein
   - Cache-Invalidierung über Regionen hinweg

## Änderungen durchgeführt

### 1. ReflectionUtils-Migration ✅
- ✅ `BukkitReflectionUtils` in `Bukkit/src/main/java/com/plotsquared/bukkit/util/` erstellt
- ✅ 3 Bukkit-Dateien aktualisiert:
  - `BukkitPlatform.java`
  - `SingleWorldListener.java`
  - `ChunkListener.java`
- ✅ Initialisierung von Core nach `BukkitPlatform.onEnable()` verschoben
- ✅ Original-`ReflectionUtils` im Core deprecated
- ✅ `serverNativePackage()`-Methode deprecated

### 2. Dokumentation ✅
- ✅ `FOLIA_READINESS_ANALYSIS.md` erstellt (auf Englisch) mit:
  - Vollständige Architektur-Analyse
  - Folia-Unterstützungs-Machbarkeitsanalyse
  - Implementierungs-Roadmap
  - Herausforderungen und Empfehlungen

## Zusammenfassung

✅ **Core-Modul ist jetzt plattformunabhängig**
✅ **Bukkit-Modul ist sauber isoliert**
✅ **Theoretischer Pfad zur Folia-Unterstützung existiert**
⚠️ **Folia-Implementierung würde erheblichen Aufwand erfordern**

Das Refactoring entfernt erfolgreich plattformspezifischen Code aus Core und macht es möglich, theoretisch Folia-Unterstützung als parallele Modul-Implementierung hinzuzufügen.

### Potenzielle Struktur mit Folia

```
PlotSquared/
├── Core/          (plattformunabhängig - gemeinsame Logik)
├── Bukkit/        (Bukkit/Spigot/Paper - aktuell)
└── Folia/         (Folia-spezifisch - zukünftig möglich)
```

Beide Module (Bukkit und Folia) würden:
- Die gleiche Core-Logik nutzen
- Ihre eigenen plattformspezifischen Implementierungen haben
- Unabhängig voneinander optimiert werden können
- Parallel existieren können

### Nächste Schritte (optional)

Falls Folia-Unterstützung gewünscht wird:

1. **TaskManager-Abstraktion überprüfen**
   - Sicherstellen, dass es verschiedene Ausführungsmodelle unterstützen kann
   - Regions-Kontext-Unterstützung hinzufügen

2. **Thread-Safety im Core prüfen**
   - Datenstrukturen auf Thread-Sicherheit überprüfen
   - Shared State identifizieren und schützen

3. **Folia-Modul implementieren**
   - Folia-spezifische Plattform-Implementierung
   - Regions-bewusste Scheduler-Wrapper
   - Folia-kompatible Event-Listener

Siehe `FOLIA_READINESS_ANALYSIS.md` (auf Englisch) für vollständige technische Details.

---

*Stand: 10.02.2026*
*Durchgeführt von: GitHub Copilot Code Agent*

# PlotSquared Economy Integration Analysis / Wirtschaftssystem-Integrationsanalyse

> **Version**: PlotSquared 7.x  
> **Ziel / Purpose**: Vollständige Analyse der Andockpunkte für Wirtschafts-Plugins wie TNE (The New Economy), Vault, etc.

---

## Inhaltsverzeichnis / Table of Contents

1. [Überblick / Overview](#überblick--overview)
2. [Architektur / Architecture](#architektur--architecture)
3. [Events für Plugin-Integration](#events-für-plugin-integration)
4. [EconHandler API](#econhandler-api)
5. [Konfiguration / Configuration](#konfiguration--configuration)
6. [Permissions / Berechtigungen](#permissions--berechtigungen)
7. [Flags System](#flags-system)
8. [Befehle mit Wirtschafts-Interaktion](#befehle-mit-wirtschafts-interaktion)
9. [Integrations-Beispiele / Integration Examples](#integrations-beispiele--integration-examples)

---

## Überblick / Overview

PlotSquared bietet ein umfangreiches Event-System und eine abstrakte Wirtschafts-API, die es Plugins wie TNE ermöglicht, sich an verschiedenen Punkten anzudocken:

| Komponente | Dateipfad | Zweck |
|------------|-----------|-------|
| `EconHandler` | `Core/src/main/java/com/plotsquared/core/util/EconHandler.java` | Abstrakte Wirtschafts-API |
| `BukkitEconHandler` | `Bukkit/src/main/java/com/plotsquared/bukkit/util/BukkitEconHandler.java` | Vault-Integration für Bukkit |
| `EventDispatcher` | `Core/src/main/java/com/plotsquared/core/util/EventDispatcher.java` | Event-Verteilung |
| `PriceFlag` | `Core/src/main/java/com/plotsquared/core/plot/flag/implementations/PriceFlag.java` | Plot-Preis-Flag |

---

## Architektur / Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     Economy Plugin (TNE)                     │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    Vault API (Economy)                       │
│         net.milkbowl.vault.economy.Economy                  │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│              BukkitEconHandler (Bukkit-Impl)                │
│    - withdrawPlayer(), depositPlayer(), getBalance()        │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                 EconHandler (Core/Abstract)                  │
│    - getMoney(), withdrawMoney(), depositMoney()            │
│    - isEnabled(PlotArea), isSupported(), format()           │
└─────────────────────────────────────────────────────────────┘
                              │
          ┌───────────────────┼───────────────────┐
          ▼                   ▼                   ▼
    ┌──────────┐       ┌──────────┐       ┌──────────┐
    │  Buy.java │       │Claim.java│       │Merge.java│
    │  Command  │       │ Command  │       │ Command  │
    └──────────┘       └──────────┘       └──────────┘
```

---

## Events für Plugin-Integration

### 🔴 **Kritische Events für Immobilien-System**

#### 1. `PlayerBuyPlotEvent` (Pre-Event)
**Datei**: `Core/src/main/java/com/plotsquared/core/events/PlayerBuyPlotEvent.java`

```java
// Wird VOR dem Kauf aufgerufen - kann abgebrochen oder modifiziert werden
public class PlayerBuyPlotEvent extends PlotPlayerEvent implements CancellablePlotEvent {
    
    // Preis anpassen (z.B. für Rabatte, Steuern)
    public void setPrice(@NonNegative final double price);
    
    // Aktuellen Preis abrufen
    public @NonNegative double price();
    
    // Event-Ergebnis setzen:
    // - Result.DENY = Kauf blockieren
    // - Result.FORCE = Kauf erzwingen (kostenlos)
    // - Result.ACCEPT / null = Normales Verhalten
    public void setEventResult(@Nullable final Result eventResult);
}
```

**Integration für TNE:**
```java
@Subscribe
public void onPlayerBuyPlot(PlayerBuyPlotEvent event) {
    PlotPlayer<?> buyer = event.getPlotPlayer();
    Plot plot = event.getPlot();
    double price = event.price();
    
    // Beispiel: Steuer hinzufügen
    double tax = price * 0.05; // 5% Steuer
    event.setPrice(price + tax);
    
    // Oder: Kauf blockieren bei Schulden
    if (hasDebt(buyer.getUUID())) {
        event.setEventResult(Result.DENY);
    }
}
```

#### 2. `PostPlayerBuyPlotEvent` (Post-Event)
**Datei**: `Core/src/main/java/com/plotsquared/core/events/post/PostPlayerBuyPlotEvent.java`

```java
// Wird NACH erfolgreichem Kauf aufgerufen - für Logging, Benachrichtigungen
public class PostPlayerBuyPlotEvent extends PlotPlayerEvent {
    
    // Vorheriger Besitzer
    public OfflinePlotPlayer previousOwner();
    
    // Bezahlter Preis (nach EventModifikationen)
    public double price();
}
```

**Integration für TNE:**
```java
@Subscribe
public void onPostPlayerBuyPlot(PostPlayerBuyPlotEvent event) {
    // Transaktion in TNE-Datenbank loggen
    UUID buyer = event.getPlotPlayer().getUUID();
    UUID seller = event.previousOwner().getUUID();
    double price = event.price();
    
    tneAPI.logTransaction(buyer, seller, price, "PLOT_PURCHASE");
}
```

#### 3. `PlayerClaimPlotEvent`
**Datei**: `Core/src/main/java/com/plotsquared/core/events/PlayerClaimPlotEvent.java`

```java
// Wird beim Claimen eines leeren Plots aufgerufen
public class PlayerClaimPlotEvent extends PlotPlayerEvent implements CancellablePlotEvent {
    
    // Schematic-Name für Claim
    public String getSchematic();
    public void setSchematic(String schematic);
    
    // Event-Ergebnis
    public void setEventResult(@Nullable Result eventResult);
}
```

#### 4. `PlotChangeOwnerEvent`
**Datei**: `Core/src/main/java/com/plotsquared/core/events/PlotChangeOwnerEvent.java`

```java
// Wird bei Besitzerwechsel aufgerufen (auch bei /plot setowner)
public class PlotChangeOwnerEvent extends PlotEvent implements CancellablePlotEvent {
    
    public PlotPlayer<?> getInitiator();  // Wer ändert den Besitzer
    public @Nullable UUID getOldOwner();  // Alter Besitzer
    public @Nullable UUID getNewOwner();  // Neuer Besitzer
    public void setNewOwner(@Nullable UUID newOwner); // Neuen Besitzer ändern
    public boolean hasOldOwner();
}
```

#### 5. `PostPlotChangeOwnerEvent`
**Datei**: `Core/src/main/java/com/plotsquared/core/events/post/PostPlotChangeOwnerEvent.java`

```java
// Nach erfolgreichem Besitzerwechsel
public class PostPlotChangeOwnerEvent extends PlotPlayerEvent {
    public @Nullable UUID oldOwner();
}
```

### 🟡 **Weitere relevante Events**

| Event | Beschreibung | Datei |
|-------|--------------|-------|
| `PlotMergeEvent` | Vor dem Zusammenführen von Plots | `PlotMergeEvent.java` |
| `PostPlotMergeEvent` | Nach dem Zusammenführen | `post/PostPlotMergeEvent.java` |
| `PlotDeleteEvent` | Vor Plot-Löschung | `PlotDeleteEvent.java` |
| `PostPlotDeleteEvent` | Nach Plot-Löschung | `post/PostPlotDeleteEvent.java` |
| `PlotFlagAddEvent` | Wenn Flag gesetzt wird (z.B. Preis) | `PlotFlagAddEvent.java` |
| `PlotFlagRemoveEvent` | Wenn Flag entfernt wird | `PlotFlagRemoveEvent.java` |
| `PlayerPlotTrustedEvent` | Vertrauenswürdiger Spieler hinzugefügt | `PlayerPlotTrustedEvent.java` |
| `PlayerPlotHelperEvent` | Helfer hinzugefügt/entfernt | `PlayerPlotHelperEvent.java` |
| `PlayerPlotDeniedEvent` | Spieler gebannt vom Plot | `PlayerPlotDeniedEvent.java` |
| `PlayerEnterPlotEvent` | Spieler betritt Plot | `PlayerEnterPlotEvent.java` |
| `PlayerLeavePlotEvent` | Spieler verlässt Plot | `PlayerLeavePlotEvent.java` |
| `PlayerPlotLimitEvent` | Plot-Limit wird geprüft | `PlayerPlotLimitEvent.java` |

---

## EconHandler API

### Abstrakte Methoden (müssen implementiert werden)

```java
public abstract class EconHandler {
    
    // Initialisierung
    public abstract boolean init();
    
    // Kontostand abrufen
    public abstract double getBalance(PlotPlayer<?> player);
    
    // Geld abziehen
    public abstract void withdrawMoney(PlotPlayer<?> player, double amount);
    
    // Geld einzahlen (Online-Spieler)
    public abstract void depositMoney(PlotPlayer<?> player, double amount);
    
    // Geld einzahlen (Offline-Spieler)
    public abstract void depositMoney(OfflinePlotPlayer player, double amount);
    
    // Prüfen ob Economy für PlotArea aktiviert
    public abstract boolean isEnabled(PlotArea plotArea);
    
    // Betrag formatieren (z.B. "100.00 €")
    public abstract @NonNull String format(double balance);
    
    // Prüfen ob Economy vom Server unterstützt wird
    public abstract boolean isSupported();
}
```

### Vault-Implementation (BukkitEconHandler)

```java
@Singleton
public class BukkitEconHandler extends EconHandler {
    
    private Economy econ; // Vault Economy Interface
    
    @Override
    public boolean init() {
        if (Bukkit.getServer().getPluginManager().getPlugin("Vault") == null) {
            return false;
        }
        RegisteredServiceProvider<Economy> economyProvider =
            Bukkit.getServer().getServicesManager().getRegistration(Economy.class);
        if (economyProvider != null) {
            this.econ = economyProvider.getProvider();
        }
        return this.econ != null;
    }
    
    @Override
    public void withdrawMoney(PlotPlayer<?> player, double amount) {
        this.econ.withdrawPlayer(getBukkitOfflinePlayer(player), amount);
    }
    
    @Override
    public void depositMoney(PlotPlayer<?> player, double amount) {
        this.econ.depositPlayer(getBukkitOfflinePlayer(player), amount);
    }
    
    @Override
    public boolean isEnabled(PlotArea plotArea) {
        return plotArea.useEconomy();
    }
}
```

---

## Konfiguration / Configuration

### worlds.yml - Economy-Einstellungen pro PlotArea

```yaml
worlds:
  plotworld:
    economy:
      use: true          # Economy aktivieren
      prices:
        claim: 100       # Preis für /plot claim
        merge: 100       # Preis für /plot merge
        sell: 100        # Verkaufspreis (nicht direkt verwendet)
```

### PlotArea.java - Preisabruf

```java
// Konfiguration laden
this.useEconomy = config.getBoolean("economy.use");

// Preise als PlotExpression (unterstützt Formeln)
ConfigurationSection priceSection = config.getConfigurationSection("economy.prices");
if (this.useEconomy) {
    this.prices = new HashMap<>();
    for (String key : priceSection.getKeys(false)) {
        String raw = priceSection.getString(key);
        // PlotExpression erlaubt z.B. "100 * {plots}" für dynamische Preise
        this.prices.put(key, PlotExpression.compile(raw, "plots"));
    }
}

// Verwendung in Commands
PlotExpression costExr = area.getPrices().get("claim");
double cost = costExr.evaluate(currentPlots); // "plots" Variable = Anzahl Plots
```

---

## Permissions / Berechtigungen

### Economy-relevante Permissions

| Permission | Beschreibung | Verwendung |
|------------|--------------|------------|
| `plots.buy` | Plot kaufen | Buy Command |
| `plots.claim` | Plot claimen | Claim Command |
| `plots.merge` | Plots zusammenführen | Merge Command |
| `plots.admin.econ.bypass` | Economy-Kosten umgehen | Alle Befehle |
| `plots.list.forsale` | Plots zum Verkauf auflisten | List Command |
| `plots.grant` | Plot-Grants verwenden | Claim ohne Zahlung |
| `plots.set.flag` | Flags setzen (inkl. Preis) | Flag Command |
| `plots.set.flag.price` | Preis-Flag setzen | Flag Command |

### Permission-Klasse

```java
// Core/src/main/java/com/plotsquared/core/permissions/Permission.java
public enum Permission {
    // ...
    PERMISSION_ADMIN_BYPASS_ECON("plots.admin.econ.bypass"),
    PERMISSION_LIST_FOR_SALE("plots.list.forsale"),
    // ...
}
```

---

## Flags System

### PriceFlag - Verkaufspreis pro Plot

```java
// Core/src/main/java/com/plotsquared/core/plot/flag/implementations/PriceFlag.java
public class PriceFlag extends DoubleFlag<PriceFlag> {
    
    public static final PriceFlag PRICE_NOT_BUYABLE = new PriceFlag(0D);
    
    protected PriceFlag(@NonNull Double value) {
        super(value, Double.MIN_NORMAL, Double.MAX_VALUE, 
              TranslatableCaption.of("flags.flag_description_price"));
    }
}
```

**Verwendung:**
```
/plot flag set price 1000    # Plot für 1000 zum Verkauf
/plot flag remove price      # Nicht mehr zum Verkauf
```

**Im Code:**
```java
// Preis aus Plot abrufen
double priceFlag = plot.getFlag(PriceFlag.class);
if (priceFlag <= 0) {
    // Plot ist nicht zum Verkauf
}
```

### Weitere relevante Flags

| Flag | Typ | Beschreibung |
|------|-----|--------------|
| `price` | Double | Verkaufspreis |
| `no-worldedit` | Boolean | WorldEdit verbieten |
| `deny-exit` | Boolean | Verlassen verbieten |
| `greeting` | String | Nachricht beim Betreten |
| `farewell` | String | Nachricht beim Verlassen |

---

## Befehle mit Wirtschafts-Interaktion

### `/plot buy` (Buy.java)

```java
// Vollständiger Kaufablauf:
1. Prüfen: Economy aktiviert? → econHandler.isEnabled(area)
2. Prüfen: Spieler kann kaufen? → Permission "plots.buy"
3. Prüfen: Plot hat Besitzer? → plot.hasOwner()
4. Prüfen: Nicht eigener Plot? → !plot.isOwner(player.getUUID())
5. Preis abrufen → plot.getFlag(PriceFlag.class)
6. Event auslösen → eventDispatcher.callPlayerBuyPlot()
7. Prüfen: Event nicht abgebrochen? → event.getEventResult() != Result.DENY
8. Guthaben prüfen → econHandler.getMoney(player) >= price
9. Geld abziehen → econHandler.withdrawMoney(player, price)
10. Geld einzahlen → econHandler.depositMoney(previousOwner, price)
11. Besitzer wechseln → plot.setOwner(player.getUUID())
12. Flag entfernen → plot.removeFlag(PriceFlag)
13. Post-Event → eventDispatcher.callPostPlayerBuyPlot()
```

### `/plot claim` (Claim.java)

```java
// Claim-Ablauf mit Kosten:
1. Event auslösen → eventDispatcher.callClaim()
2. Prüfen: Economy aktiviert? → econHandler.isEnabled(area)
3. Prüfen: Nicht bypass? → !player.hasPermission(PERMISSION_ADMIN_BYPASS_ECON)
4. Kosten berechnen → area.getPrices().get("claim").evaluate(currentPlots)
5. Guthaben prüfen → econHandler.getMoney(player) >= cost
6. Geld abziehen → econHandler.withdrawMoney(player, cost)
```

### `/plot merge` (Merge.java)

```java
// Merge-Ablauf mit Kosten:
1. Event auslösen → eventDispatcher.callMerge()
2. Kosten berechnen → area.getPrices().get("merge").evaluate(size)
3. Guthaben prüfen → econHandler.getMoney(player) >= price
4. Geld abziehen → econHandler.withdrawMoney(player, price)
5. Post-Event → eventDispatcher.callPostMerge()
```

---

## Integrations-Beispiele / Integration Examples

### Beispiel 1: TNE Integration Plugin

```java
package com.example.tneplotbridge;

import com.google.common.eventbus.Subscribe;
import com.plotsquared.core.PlotSquared;
import com.plotsquared.core.events.PlayerBuyPlotEvent;
import com.plotsquared.core.events.PlayerClaimPlotEvent;
import com.plotsquared.core.events.Result;
import com.plotsquared.core.events.post.PostPlayerBuyPlotEvent;

public class TNEPlotBridge {
    
    private final TNEEconomy tneEconomy;
    
    public void onEnable() {
        // Bei PlotSquared Event-System registrieren
        PlotSquared.get().getEventDispatcher().registerListener(this);
    }
    
    @Subscribe
    public void onBuyPlot(PlayerBuyPlotEvent event) {
        UUID buyer = event.getPlotPlayer().getUUID();
        double price = event.price();
        
        // TNE-spezifische Logik
        if (!tneEconomy.canAfford(buyer, price)) {
            event.setEventResult(Result.DENY);
            event.getPlotPlayer().sendMessage("§cNicht genug Geld auf deinem Konto!");
            return;
        }
        
        // Steuern hinzufügen
        double taxRate = tneEconomy.getTaxRate(buyer);
        event.setPrice(price * (1 + taxRate));
    }
    
    @Subscribe
    public void onPostBuyPlot(PostPlayerBuyPlotEvent event) {
        // Transaktion in TNE loggen
        tneEconomy.logTransaction(
            event.getPlotPlayer().getUUID(),
            event.previousOwner().getUUID(),
            event.price(),
            "PLOT_SALE"
        );
    }
}
```

### Beispiel 2: Custom EconHandler (ohne Vault)

```java
package com.example.directtne;

import com.plotsquared.core.util.EconHandler;
import com.plotsquared.core.player.PlotPlayer;
import com.plotsquared.core.plot.PlotArea;

public class TNEDirectHandler extends EconHandler {
    
    private final TNEEconomy tne;
    
    @Override
    public boolean init() {
        // TNE direkt initialisieren
        return TNE.getAPI() != null;
    }
    
    @Override
    public double getBalance(PlotPlayer<?> player) {
        return tne.getBalance(player.getUUID());
    }
    
    @Override
    public void withdrawMoney(PlotPlayer<?> player, double amount) {
        tne.withdraw(player.getUUID(), amount);
    }
    
    @Override
    public void depositMoney(PlotPlayer<?> player, double amount) {
        tne.deposit(player.getUUID(), amount);
    }
    
    // ... weitere Methoden
}
```

### Beispiel 3: Immobilien-Verwaltungssystem

```java
package com.example.plotrealestate;

import com.google.common.eventbus.Subscribe;
import com.plotsquared.core.events.*;
import com.plotsquared.core.events.post.*;

public class RealEstateManager {
    
    private final Database database;
    
    // === KAUF/VERKAUF ===
    
    @Subscribe
    public void onBuyPlot(PlayerBuyPlotEvent event) {
        // Prüfen ob Plot in Immobilien-System registriert
        if (database.isRegisteredProperty(event.getPlot().getId())) {
            // Zusätzliche Gebühren (Notar, etc.)
            double additionalFees = database.getPropertyFees(event.getPlot().getId());
            event.setPrice(event.price() + additionalFees);
        }
    }
    
    @Subscribe
    public void onPostBuy(PostPlayerBuyPlotEvent event) {
        // Eigentum in Datenbank aktualisieren
        database.updateOwnership(
            event.getPlot().getId(),
            event.previousOwner().getUUID(),
            event.getPlotPlayer().getUUID(),
            event.price()
        );
    }
    
    // === BESITZER-ÄNDERUNGEN ===
    
    @Subscribe
    public void onOwnerChange(PlotChangeOwnerEvent event) {
        // Prüfen ob Transfer erlaubt
        if (database.hasLien(event.getPlot().getId())) {
            // Pfandrecht auf Plot - Transfer blockieren
            event.setEventResult(Result.DENY);
            event.getInitiator().sendMessage("§cDieses Grundstück hat ein Pfandrecht!");
        }
    }
    
    @Subscribe
    public void onPostOwnerChange(PostPlotChangeOwnerEvent event) {
        // Grundbuch aktualisieren
        database.recordOwnershipChange(event.getPlot(), event.oldOwner());
    }
    
    // === PLOT-LÖSCHUNG ===
    
    @Subscribe
    public void onPlotDelete(PlotDeleteEvent event) {
        // Prüfen ob Löschung erlaubt
        if (database.hasMortgage(event.getPlot().getId())) {
            event.setEventResult(Result.DENY);
        }
    }
    
    // === TRUST-SYSTEM (Miteigentum) ===
    
    @Subscribe
    public void onPlayerTrusted(PlayerPlotTrustedEvent event) {
        if (event.wasAdded()) {
            database.addCoOwner(event.getPlot().getId(), event.getPlayer());
        } else {
            database.removeCoOwner(event.getPlot().getId(), event.getPlayer());
        }
    }
    
    // === FLAG-ÄNDERUNGEN ===
    
    @Subscribe
    public void onFlagAdd(PlotFlagAddEvent event) {
        if (event.getFlag() instanceof PriceFlag) {
            // Plot wird zum Verkauf angeboten
            double price = ((PriceFlag) event.getFlag()).getValue();
            database.listForSale(event.getPlot().getId(), price);
        }
    }
    
    @Subscribe
    public void onFlagRemove(PlotFlagRemoveEvent event) {
        if (event.getFlag() instanceof PriceFlag) {
            // Plot nicht mehr zum Verkauf
            database.removeFromSale(event.getPlot().getId());
        }
    }
}
```

---

## Zusammenfassung / Summary

### Hauptandockpunkte für Economy-Plugins:

| Priorität | Andockpunkt | Zweck |
|-----------|-------------|-------|
| 🔴 Hoch | `PlayerBuyPlotEvent` | Kaufpreis modifizieren, Kauf blockieren |
| 🔴 Hoch | `PostPlayerBuyPlotEvent` | Transaktionen loggen |
| 🔴 Hoch | `EconHandler` | Direkte Economy-Integration |
| 🟡 Mittel | `PlayerClaimPlotEvent` | Claim-Kosten modifizieren |
| 🟡 Mittel | `PlotChangeOwnerEvent` | Besitzerwechsel kontrollieren |
| 🟡 Mittel | `PlotMergeEvent` | Merge-Kosten modifizieren |
| 🟢 Niedrig | `PlotFlagAddEvent` | Preis-Flag überwachen |
| 🟢 Niedrig | `PlayerPlotTrustedEvent` | Miteigentum verwalten |

### Empfohlene Integration für TNE:

1. **Vault nutzen**: TNE über Vault registrieren, PlotSquared verwendet automatisch die `BukkitEconHandler`
2. **Events abonnieren**: `PlayerBuyPlotEvent` und `PostPlayerBuyPlotEvent` für Transaktions-Logging
3. **Permissions prüfen**: `plots.admin.econ.bypass` für Admin-Ausnahmen beachten

### Dateipfade für tiefere Analyse:

```
Core/src/main/java/com/plotsquared/core/
├── events/
│   ├── PlayerBuyPlotEvent.java
│   ├── PlayerClaimPlotEvent.java
│   ├── PlotChangeOwnerEvent.java
│   ├── PlotFlagAddEvent.java
│   └── post/
│       ├── PostPlayerBuyPlotEvent.java
│       └── PostPlotChangeOwnerEvent.java
├── util/
│   ├── EconHandler.java
│   └── EventDispatcher.java
├── command/
│   ├── Buy.java
│   ├── Claim.java
│   └── Merge.java
├── plot/
│   ├── PlotArea.java
│   └── flag/implementations/PriceFlag.java
└── permissions/
    └── Permission.java

Bukkit/src/main/java/com/plotsquared/bukkit/
└── util/
    └── BukkitEconHandler.java
```

---

*Erstellt für PlotSquared Economy/Real Estate Integration Analysis*

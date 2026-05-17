![JCO Amplitude on NAS100 5min](assets/preview.png)

# JCO Amplitude — TradingView

Indicateur d'amplitude pour TradingView (Pine Script v6), orienté trading
**contrarien** sur futures indices (NQ / MNQ / ES / YM / RTY par défaut).
Port de l'indicateur Tradovate
[`Tradovate_Indicator_JCO_Amplitude`](https://github.com/jcornierfra/Tradovate_Indicator_JCO_Amplitude).

L'indicateur marque sur le graphique, par un trait horizontal au prix
exact, les instants où le marché s'est suffisamment éloigné du dernier
swing pour signaler une zone de retournement potentielle :

- **Mouvement BAS → HAUT** → marque **rouge** (signal vente contrarien)
- **Mouvement HAUT → BAS** → marque **verte** (signal achat contrarien)

## Pourquoi le 1 minute

La version Tradovate travaille en tick. Sur TradingView, on utilise
les **high / low des bougies 1 min** pour approximer le tick-by-tick.
La bougie en cours de formation est utilisée pour le dernier high/low
et comme bougie de droite pour la détection de swing : déclenchement
disponible **intra-bar**.

> ⚠️ En realtime, Pine ré-exécute le script depuis l'état confirmé de la
> bougie précédente à chaque tick. Conséquence : **maximum 1 marque par
> bougie en intra-bar**. C'est acceptable en 1 min ; sur une session
> normale on n'aura jamais 2 marques dans la même minute. Si tu as besoin
> de plus de précision, passe en 30 s.

## Mécanique

### Phase 1 — `searching` (1ère marque du jour)

Détection du dernier swing strict via `leftBars` / `rightBars`. Depuis
le swing trouvé, on attend qu'`amp1` soit atteint dans le sens opposé
(au-dessus si swing low, en dessous si swing high). Marque posée au
prix exact `swing.price ± amp1` → transition vers la phase 2.

### Phase 2 — `tracking` (cycles suivants)

Machine d'états par cycle (direction `up` ; symétrique en `down`) :

| État              | Rôle                                                                                                            |
|-------------------|-----------------------------------------------------------------------------------------------------------------|
| `origin`          | Ancre du calcul des 30 %                                                                                        |
| `refExtreme`      | Plus haut atteint depuis la marque (monotone)                                                                   |
| `oppExtreme`      | Plus bas atteint depuis la marque (monotone, ne reset PAS quand `refExtreme` monte). Sert au **continuation**   |
| `localOpp`        | Plus bas depuis le **dernier reset de `refExtreme`**. Reset à `refExtreme` sur nouveau plus haut. Sert au **reversal** |
| `recoveryExt`     | Plus haut atteint depuis le dernier reset de `oppExtreme`                                                       |
| `currentPath`     | `amp1` \| `amp2`, **sticky**, figé au moment où `oppExtreme` baisse                                             |
| `retracementSeen` | `true` dès qu'`oppExtreme` a strictement bougé sous la marque                                                   |

**Path** : réévalué **uniquement** quand `oppExtreme` baisse.
- `retrace = refExtreme - oppExtreme`
- `retrace > 30 % × (refExtreme - origin)` → `path = amp1`, sinon `path = amp2`

**Deux règles, évaluées à chaque tick, par priorité :**

1. **Reversal** (seuil = `amp1` toujours, requiert `retracementSeen`)
   - `localRetrace = refExtreme - localOpp ≥ amp1`
   - → marque `amp1`, **couleur opposée**, direction inversée
   - `origin ← refExtreme`

2. **Continuation** (path-driven)
   - `recovery = recoveryExt - oppExtreme ≥ amp_path`
   - → marque `amp_path`, **même couleur** que la précédente
   - `origin ← oppExtreme`

### Reset journalier

À minuit (heure **Europe/Paris**) : oubli du swing actif et de la
dernière direction marquée, pour qu'après un gap éventuel la 1ère marque
du jour exige à nouveau `amp1`. Les marques déjà dessinées restent
visibles dans l'historique.

## Installation

1. Ouvrir TradingView → Pine Editor.
2. Coller le contenu de [`Indicator_JCO_Amplitude.pine`](Indicator_JCO_Amplitude.pine).
3. **Save** puis **Add to chart**.
4. Recommandé : timeframe **1 minute** sur les futures indices.

## Paramètres

### Amplitudes
| Paramètre   | Défaut | Description                                                                |
|-------------|--------|----------------------------------------------------------------------------|
| `Amp1`      | 80     | Seuil de la 1ère marque du jour ET du reversal                             |
| `Amp2`      | 60     | Seuil de continuation (quand `currentPath = amp2`)                         |
| `pointSize` | 1.0    | Prix d'1 point : NQ / MNQ / ES / YM / RTY = 1. À ajuster pour d'autres contrats |

### Swing
| Paramètre   | Défaut | Description                                  |
|-------------|--------|----------------------------------------------|
| `leftBars`  | 1      | Nombre de bougies à gauche du pivot          |
| `rightBars` | 1      | Nombre de bougies à droite du pivot (la bougie en cours compte) |

### Fenêtre horaire (Europe/Paris)
| Paramètre               | Défaut | Description              |
|-------------------------|--------|--------------------------|
| `Début (Paris) h / min` | 08:00  | Début de la fenêtre      |
| `Fin   (Paris) h / min` | 22:00  | Fin de la fenêtre (exclu) |

### Affichage
| Paramètre         | Défaut    | Description                                     |
|-------------------|-----------|-------------------------------------------------|
| `markerWidthBars` | 1.4       | Largeur du trait (en multiples de bougie)       |
| `lineWidth`       | 2         | Épaisseur du trait                              |
| `bullColor`       | `#00C853` | Couleur haussière (signal achat contrarien)     |
| `bearColor`       | `#FF1744` | Couleur baissière (signal vente contrarien)     |

## Astuce d'affichage

Sur TradingView, les drawings d'un indicateur overlay peuvent passer
sous les chandeliers selon l'ordre de rendu. Si nécessaire :
**clic droit sur le nom de l'indicateur dans le chart → Visual order →
Bring to front**.

## Changelog

### v2.1.3 — 2026-05-17 (Pine)
- Port initial Pine v6 de la version Tradovate v2.1.3.
- Adaptations TradingView :
  - Détection swing native via offsets de bars (`[i]`) sur la timeframe
    du chart (1 min recommandé) au lieu du tick-by-tick Tradovate.
  - Reset journalier basé sur `Europe/Paris` via une clé `year*10000 + month*100 + day`
    (Pine v6 n'expose pas `dayofyear()`).
  - Fenêtre horaire avec heures **et minutes** (alors que Tradovate n'avait
    que les heures).
  - Rendu via `line.new` (trait horizontal épaisseur paramétrable) au
    lieu du rectangle custom plotter de Tradovate, devenu inutile sur TV.

Pour le détail de la mécanique algorithmique (changelog Tradovate v1.0.0
à v2.1.3, qui décrit toutes les itérations sur les règles reversal /
continuation / path-evaluation), voir le
[changelog de la version Tradovate](https://github.com/jcornierfra/Tradovate_Indicator_JCO_Amplitude/blob/main/Indicator_JCO_Amplitude.js).

## Licence

[MPL-2.0](https://mozilla.org/MPL/2.0/) — © jcornier

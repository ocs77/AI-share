# SPEC — Crypto Portfolio Tracker
## Olivier & Aurore — Bitcoin DCA Tracker

> Version 1.1 — Mai 2026  
> Repo : `ocs77/AI-share`

---

## 1. Objectif

Script Python autonome qui calcule en temps réel :
- Le **coût total d'acquisition** en EUR (toutes sources)
- Le **prix moyen d'achat** (CUMP simplifié)
- La **valeur actuelle** du portefeuille BTC
- La **plus-value latente** et l'**impôt théorique** à 30%
- Un **bloc JSON** prêt à injecter dans le reporting patrimonial HTML

Le script tourne en local, sans serveur, sans dépendance cloud.  
La privacy est maximale : aucune donnée wallet n'est envoyée à un service tiers.

---

## 2. Sources de données

### 2.1 Kraken — API temps réel (source principale, dynamique)

Endpoint : `https://api.kraken.com/0/private/Ledgers`  
Authentification : clé API en lecture seule (stockée dans `.env`, jamais commitée)

**Permissions requises (et uniquement celles-ci) :**
- `Query Ledger Entries` — historique complet des mouvements
- `Query Funds` — solde actuel sur Kraken

**Logique d'extraction :**
- Paginer l'endpoint `/Ledgers` (50 résultats/appel) jusqu'à couvrir tout l'historique
- Filtrer les lignes `asset=EUR` + `type in (trade, spend)` + `amount < 0`
- Pour chaque sortie EUR, retrouver la crypto achetée via `refid` commun
- **Ne comptabiliser que les achats de BTC** (direct EUR→BTC)
- Gérer les cas ETH→BTC : utiliser le coût EUR de l'ETH si traçable dans le même `refid`

**Cache** : `data/kraken_ledger_cache.csv`  
Mis à jour incrémentalement — fetch uniquement les nouvelles lignes depuis le dernier `txid` connu.

---

### 2.2 Deblock — Import manuel figé (source secondaire, statique)

Fichier : `data/deblock_manual.csv`  
Format :

```csv
date,description,eur_spent,btc_received,notes
2025-01-01,onramp EUR→BTC,50.00,0.00056,ETH swap inclus
2025-02-19,onramp EUR→BTC,500.00,0.00544,
2025-02-25,onramp EUR→BTC,1000.00,0.01146,
2025-02-26,onramp EUR→BTC,500.00,0.00587,
2025-02-28,onramp EUR→BTC,500.00,0.00648,
2025-03-03,onramp EUR→BTC,500.00,0.00610,
2025-03-09,onramp EUR→BTC,500.00,0.00646,
2025-03-30,onramp EUR→BTC,3450.00,0.04512,
2025-05-02,onramp EUR→BTC,40.00,0.00046,
2025-05-13,onramp EUR→BTC,100.00,0.00105,
2025-06-29,onramp EUR→BTC,500.00,0.00540,
2025-08-01,onramp EUR→BTC,400.00,0.00399,
2025-08-29,onramp EUR→BTC,200.00,0.00211,
2025-10-11,onramp EUR→BTC,500.00,0.00512,
2025-10-29,onramp EUR→BTC,400.00,0.00416,
2025-11-04,onramp EUR→BTC,480.00,0.00544,
2025-11-22,onramp EUR→BTC,500.00,0.00678,
2025-11-26,onramp EUR→BTC,510.00,0.00653,
2025-12-15,onramp EUR→BTC,250.00,0.00338,
2025-12-22,onramp EUR→BTC,1000.00,0.01305,
2025-12-29,onramp EUR→BTC,530.00,0.00707,
2025-12-30,onramp EUR→BTC,600.00,0.00799,
2025-02-04,ETH→BTC swap,2500.00,0.02738,EUR 2500€ → ETH → BTC via Deblock
2026-01-02,onramp EUR→BTC,1700.00,0.02212,saisie manuelle (hors export 2025)
2026-01-05,onramp EUR→BTC,1200.00,0.01482,saisie manuelle (hors export 2025)
2026-04-12,onramp EUR→BTC,2900.00,0.04713,saisie manuelle (hors export 2025)
```

**Règles :**
- Fichier **figé dans le temps** — ne jamais écraser, uniquement append si nouvelle transaction
- Export annuel Deblock reçu par mail en début d'année suivante → append à ce moment
- Transactions 2026 saisies manuellement depuis `Classeur1.xlsx`
- Ce fichier est versionné sur GitHub (aucune donnée sensible)

---

### 2.3 Hard inputs — ETH externes et pré-historique

Fichier : `data/hard_inputs.json`

```json
{
  "description": "Transactions non traçables via CSV/API",
  "entries": [
    {
      "id": "eth_externe_fev_2025",
      "date": "2025-02-09",
      "eur_spent": null,
      "btc_received": 0.04366,
      "notes": "1.60 ETH reçu wallet externe 0x3d0427a7... → swap BTC. Coût EUR inconnu.",
      "include_in_cump": false
    },
    {
      "id": "eth_externe_mar_2025",
      "date": "2025-03-10",
      "eur_spent": null,
      "btc_received": 0.03493,
      "notes": "1.40 ETH reçu wallet externe 0x3d0427a7... → swap BTC. Coût EUR inconnu.",
      "include_in_cump": false
    },
    {
      "id": "eth_externe_jun_2025",
      "date": "2025-06-09",
      "eur_spent": null,
      "btc_received": 0.00935,
      "notes": "0.40 ETH reçu wallet externe 0x43Efdc94... → swap BTC. Coût EUR inconnu.",
      "include_in_cump": false
    },
    {
      "id": "pre_historique",
      "date": "2021-12-31",
      "eur_spent": 0,
      "btc_received": 0,
      "notes": "Placeholder transactions pré-2022. Mettre à jour si retrouvées.",
      "include_in_cump": true
    }
  ]
}
```

**Règle `include_in_cump` :**
- `true` : EUR inclus dans le coût d'acquisition
- `false` : BTC compté dans le volume total, EUR exclus du coût → améliore le CUMP

---

### 2.4 Solde BTC — Wallet self-custody via nœud Umbrel

**Principe privacy-first :** le zpub ne quitte jamais la machine locale.  
Toute la dérivation d'adresses se fait localement dans le script Python.

**Stack :**
- **Nœud** : Bitcoin Core sur Umbrel
- **Indexeur** : Electrs (serveur Electrum, app officielle Umbrel)
- **Protocole** : Electrum JSON-RPC sur TCP (port 50001)
- **Accès** : réseau local (`umbrel.local:50001`) ou Tailscale depuis l'extérieur

**Flow complet :**
```
zpub (local)
  ↓ dérivation des adresses (lib bip32utils, 100% local)
  ↓ conversion adresse → scripthash (sha256 reversed)
  ↓ requête TCP vers Electrs umbrel.local:50001
  ↓ blockchain.scripthash.get_balance → satoshis confirmés
  ↓ ÷ 100_000_000 → solde BTC
```

**Accès distant via Tailscale :**
```bash
# En local
ELECTRS_HOST=umbrel.local

# Depuis l'extérieur (IP Tailscale de l'Umbrel)
ELECTRS_HOST=100.x.x.x
```
Tor disponible en fallback mais non recommandé pour usage programmatique (latence).

**Nombre d'adresses dérivées :** gap limit = 20 (standard BIP44/84)  
Le script dérive les adresses external (réception) et internal (change) jusqu'à 20 adresses vides consécutives.

**Fallback** : si Electrs indisponible, lire `btc_volume_manual` dans `config.json`.

---

### 2.5 Prix BTC live

API : `https://api.coingecko.com/api/v3/simple/price?ids=bitcoin&vs_currencies=eur`  
Publique, sans clé.

**Fallback :** dernier prix connu en cache si CoinGecko indisponible.

---

## 3. Calcul CUMP

```
total_eur_investi = Σ(Kraken EUR→BTC)
                 + Σ(Deblock eur_spent)
                 + Σ(hard_inputs.eur_spent where include_in_cump=true)

volume_btc_total = solde_electrs_wallet
                 + solde_kraken_spot

prix_moyen_achat = total_eur_investi / volume_btc_total

valeur_actuelle  = prix_btc_live × volume_btc_total

pv_latente       = valeur_actuelle - total_eur_investi
pv_pct           = pv_latente / total_eur_investi × 100

impot_theorique  = max(0, pv_latente × 0.30)
valeur_nette     = valeur_actuelle - impot_theorique
```

**Note CUMP :** calcul simplifié (pas FIFO ligne par ligne).  
Suffisant pour le suivi patrimonial mensuel.  
Pour une vente réelle → snapshot du CUMP dans `data/ventes.csv` à ce moment précis.

---

## 4. Outputs

### 4.1 Console

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  BTC PORTFOLIO — 28 mai 2026
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  Volume BTC         :  1.0031 BTC
  Prix actuel        :  64 800 €
  Valeur actuelle    :  65 001 €

  EUR investi (CUMP) :  44 523 €
  Prix moyen achat   :  44 381 €/BTC

  Plus-value latente :  +20 478 € (+46.0%)
  Impôt théorique    :   -6 143 €  (30%)
  Valeur nette       :   58 858 €
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  Kraken API  : 48 tx · 26 363 €
  Deblock     : 26 tx · 13 260 € + 4 900 € (2026 manuel)
  Hard inputs :  3 tx ·  0 € (ETH externes exclus du coût)
  Wallet      : Electrs umbrel.local · 0.9954 BTC confirmés
  Kraken spot :                      · 0.0077 BTC
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  Dernière MAJ : 2026-05-28 14:32:15
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

### 4.2 JSON — `crypto_output.json`

```json
{
  "generated_at": "2026-05-28T14:32:15Z",
  "btc": {
    "volume": 1.0031,
    "price_eur": 64800,
    "value_eur": 65001,
    "cost_basis_eur": 44523,
    "avg_buy_price_eur": 44381,
    "unrealized_pnl_eur": 20478,
    "unrealized_pnl_pct": 46.0,
    "tax_theoretical_eur": 6143,
    "net_value_eur": 58858
  },
  "sources": {
    "kraken_tx_count": 48,
    "kraken_eur_total": 26363,
    "deblock_tx_count": 26,
    "deblock_eur_total": 18160,
    "hard_inputs_eur_total": 0,
    "btc_wallet_balance": 0.9954,
    "btc_kraken_balance": 0.0077
  }
}
```

### 4.3 Snapshot CUMP (en cas de vente)

```bash
python crypto_tracker/tracker.py --snapshot-vente --montant-btc=0.1
```

Écrit dans `data/ventes.csv` :
```csv
date,btc_vendu,prix_vente_eur,cump_eur,pv_eur,impot_eur
2026-06-15,0.1,68000,44381,2362,708
```

---

## 5. Structure des fichiers

```
ocs77/AI-share/
├── crypto_tracker/
│   ├── tracker.py           # script principal + CLI
│   ├── kraken_api.py        # client API Kraken (fetch + pagination + cache)
│   ├── deblock_parser.py    # lecture deblock_manual.csv
│   ├── wallet.py            # connexion Electrs via TCP + dérivation zpub
│   ├── calculator.py        # logique CUMP + outputs
│   └── config.py            # lecture .env + config.json
├── data/
│   ├── deblock_manual.csv      # ✅ versionné
│   ├── hard_inputs.json        # ✅ versionné
│   ├── kraken_ledger_cache.csv # ✅ versionné
│   ├── ventes.csv              # ✅ versionné (créé à la première vente)
│   └── config.json             # ✅ versionné
├── .env                        # ❌ NON versionné
├── .env.example                # ✅ versionné
├── .gitignore
├── requirements.txt
└── SPEC.md
```

---

## 6. Configuration

### `.env` (jamais commitée)

```bash
KRAKEN_API_KEY=xxxxxxxxxxxx
KRAKEN_API_SECRET=xxxxxxxxxxxx
BTC_ZPUB=zpubxxxxxxxxxxxxxxxx    # clé publique étendue du wallet self-custody
ELECTRS_HOST=umbrel.local        # ou IP Tailscale depuis l'extérieur
ELECTRS_PORT=50001
```

### `.env.example` (commitée)

```bash
KRAKEN_API_KEY=your_api_key_here
KRAKEN_API_SECRET=your_api_secret_here
BTC_ZPUB=zpub_your_extended_public_key_here
ELECTRS_HOST=umbrel.local
ELECTRS_PORT=50001
```

### `data/config.json`

```json
{
  "btc_volume_manual": null,
  "flat_tax_rate": 0.30,
  "cache_max_age_minutes": 60,
  "electrs_gap_limit": 20,
  "electrs_timeout_seconds": 10
}
```

---

## 7. Dépendances Python

```
requests>=2.31.0
python-dotenv>=1.0.0
openpyxl>=3.1.0
bip32utils>=0.3.post4     # dérivation zpub → adresses BTC localement
```

---

## 8. Utilisation

```bash
# Installation
pip install -r requirements.txt
cp .env.example .env
# → remplir KRAKEN_API_KEY, KRAKEN_API_SECRET, BTC_ZPUB, ELECTRS_HOST

# Exécution standard
python crypto_tracker/tracker.py

# Forcer le refresh complet du cache Kraken
python crypto_tracker/tracker.py --refresh

# Output JSON uniquement (injection reporting)
python crypto_tracker/tracker.py --json-only

# Snapshot CUMP avant une vente
python crypto_tracker/tracker.py --snapshot-vente --montant-btc=0.1
```

---

## 9. Sécurité & Privacy

| Donnée | Stockage | Exposition externe |
|--------|----------|--------------------|
| Clés API Kraken | `.env` local | ❌ jamais |
| zpub wallet | `.env` local | ❌ jamais |
| Dérivation adresses | RAM locale | ❌ jamais |
| Requêtes solde BTC | Electrs Umbrel via LAN/Tailscale | ❌ jamais |
| Prix BTC | CoinGecko API publique | ✅ OK (pas de wallet) |
| Historique Kraken | API Kraken + cache local | ✅ OK (déjà sur Kraken) |

---

## 10. Évolutions futures

- [ ] Snapshot automatique du CUMP lors d'une vente
- [ ] Calcul plus-value réalisée pour déclaration 2086
- [ ] Support multi-crypto (ETH, LINK, SOL) si besoin
- [ ] Injection automatique dans `reporting-patrimonial-MMMAAAA.html`
- [ ] GitHub Actions pour exécution mensuelle automatique

---

## 11. État des données au 28 mai 2026

| Source | Tx | EUR traçables | Mode |
|--------|-----|---------------|------|
| Kraken API | ~48 | ~26 363 € | ✅ dynamique |
| Deblock CSV 2025 | 24 | 13 260 € | ✅ figé |
| Deblock ETH swap | 1 | 2 500 € | ✅ figé |
| Deblock manuel 2026 | 3 | 4 900 € | ✅ hard input |
| ETH externes → BTC | 3 | inconnu | ⚠️ exclus du CUMP |
| Pré-historique | ? | à estimer | ⚠️ placeholder à 0 |
| **TOTAL traçable** | **~79** | **~46 023 €** | |

> Les ~0.089 BTC issus des ETH externes sont dans le volume total  
> mais pas dans le coût → ils améliorent le CUMP en ta faveur.

---

*Spec rédigée avec Claude Sonnet 4.6 — Mai 2026*

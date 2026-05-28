# SPEC — Bitcoin Portfolio Tracker

> Version 1.2 — Mai 2026  
> Repo : `ocs77/AI-share`

---

## 1. Objectif

Brique locale de suivi BTC destinée à alimenter le reporting patrimonial.

Le tracker calcule :
- le volume total de BTC détenu ;
- le coût d'acquisition traçable en EUR ;
- le capital d'acquisition restant après éventuelles cessions ;
- le prix moyen indicatif ;
- la valeur actuelle du portefeuille BTC ;
- la plus-value latente indicative ;
- une estimation patrimoniale d'impôt latent ;
- un JSON de sortie injectable dans le reporting patrimonial.

Le module doit rester autonome, local et privacy-first.

---

## 2. Règles de confidentialité

Le repo ne doit contenir aucune donnée patrimoniale réelle.

Ne jamais versionner :
- les clés API ;
- les clés publiques étendues du wallet ;
- les adresses dérivées ;
- les historiques d'achat ;
- les volumes BTC réels ;
- les montants EUR réels ;
- les fichiers de vente ;
- les caches ;
- les outputs JSON ou HTML contenant des chiffres réels.

Les fichiers versionnés doivent être uniquement des exemples fictifs ou des templates.

La clé publique étendue du wallet est une donnée sensible : elle ne doit jamais être loggée, affichée, exportée, commitée ou envoyée à un explorateur public.

---

## 3. Sources de données

### 3.1 Historique plateforme

Source dynamique via API en lecture seule.

Objectif : reconstruire les achats BTC traçables en EUR.

Règles :
- lecture seule ;
- cache local non versionné ;
- filtrage des mouvements qui correspondent à de véritables acquisitions ;
- exclusion des transferts internes ;
- conservation des frais si disponibles.

### 3.2 Import manuel

Fichier local non versionné : `data/private/deblock_manual.csv`  
Template versionné : `data/examples/deblock_manual.example.csv`

Colonnes attendues :

```csv
date,description,eur_spent,btc_received,fees_eur,notes
2025-01-01,onramp EUR vers BTC,100.00,0.00100000,0.50,EXEMPLE FICTIF
```

Règles :
- ne jamais écraser l'historique ;
- ajouter uniquement les nouvelles lignes documentées ;
- ne jamais commit le fichier réel.

### 3.3 Entrées à coût inconnu ou incomplet

Fichier local non versionné : `data/private/hard_inputs.json`  
Template versionné : `data/examples/hard_inputs.example.json`

Ces entrées servent à représenter des BTC présents dans le wallet mais dont le coût EUR n'est pas documenté.

Règles :
- le volume BTC est inclus dans le volume total ;
- le coût EUR est exclu du coût traçable si inconnu ;
- le reporting doit afficher un warning clair ;
- ne jamais présenter cela comme un avantage fiscal ;
- formulation attendue : coût non documenté, prix moyen indicatif artificiellement plus bas.

### 3.4 Solde wallet self-custody

Le solde wallet est obtenu localement via le nœud personnel et son indexeur.

Règles :
- dérivation d'adresses locale ;
- aucune clé publique étendue envoyée à une API publique ;
- aucun explorateur public ;
- fallback manuel possible avec warning explicite.

### 3.5 Prix BTC

Le prix BTC peut provenir d'une API publique de prix, car aucune information wallet n'est envoyée.

Prévoir un fallback sur le dernier prix connu en cache local.

---

## 4. Hypothèse fiscale de périmètre

Le calcul simplifié repose sur une hypothèse forte : le foyer ne détient que du BTC comme actif numérique.

Dans ce cas, la formule fiscale globale du portefeuille revient à une logique de prorata sur le volume BTC détenu.

Si le foyer détient plus tard d'autres actifs numériques, par exemple ETH, SOL, LINK ou stablecoins, il faudra abandonner le raisonnement BTC-only et recalculer la valeur globale du portefeuille d'actifs numériques au moment de chaque cession.

---

## 5. Calcul patrimonial courant

Champs recommandés :

```text
traced_cost_basis_eur
capital_acquisition_restant_eur
btc_total
btc_price_eur
current_value_eur
indicative_avg_buy_price_eur
indicative_unrealized_pnl_eur
indicative_unrealized_pnl_pct
patrimonial_tax_estimate_eur
net_value_after_tax_estimate_eur
```

Formules :

```text
btc_total = btc_wallet + btc_platform

indicative_avg_buy_price_eur = capital_acquisition_restant_eur / btc_total

current_value_eur = btc_total * btc_price_eur

indicative_unrealized_pnl_eur = current_value_eur - capital_acquisition_restant_eur

indicative_unrealized_pnl_pct = indicative_unrealized_pnl_eur / capital_acquisition_restant_eur

patrimonial_tax_estimate_eur = max(0, indicative_unrealized_pnl_eur * flat_tax_rate)

net_value_after_tax_estimate_eur = current_value_eur - patrimonial_tax_estimate_eur
```

Vocabulaire obligatoire :
- utiliser `traced_cost_basis_eur`, pas seulement `cost_basis_eur` ;
- utiliser `indicative_avg_buy_price_eur`, pas seulement `avg_buy_price_eur` ;
- utiliser `patrimonial_tax_estimate_eur`, pas `tax_theoretical_eur` ;
- afficher les coûts inconnus comme limite du calcul.

---

## 6. Calcul après cession

Le point clé est de maintenir un capital d'acquisition restant.

La seconde cession ne doit pas repartir du coût historique initial. Elle doit repartir du capital d'acquisition restant après la première cession.

### 6.1 Variables

```text
btc_total_avant_cession
btc_vendu
prix_cession_brut_eur
frais_cession_eur
capital_acquisition_restant_avant_eur
```

### 6.2 Formules BTC-only

```text
fraction_portefeuille_cedee = btc_vendu / btc_total_avant_cession

capital_consomme_eur = capital_acquisition_restant_avant_eur * fraction_portefeuille_cedee

prix_cession_net_eur = prix_cession_brut_eur - frais_cession_eur

plus_value_realisee_eur = prix_cession_net_eur - capital_consomme_eur

capital_acquisition_restant_apres_eur = capital_acquisition_restant_avant_eur - capital_consomme_eur

btc_total_apres_cession = btc_total_avant_cession - btc_vendu
```

### 6.3 Exemple pédagogique fictif

Avant première cession :

```text
BTC total = 1.00000000
Capital acquisition restant = 45 000 EUR
```

Première cession :

```text
BTC vendu = 0.10000000
Fraction vendue = 10 %
Capital consommé = 4 500 EUR
Capital restant après cession = 40 500 EUR
```

Deuxième cession plus tard :

```text
Valeur du portefeuille avant cession = 100 000 EUR
Montant vendu = 20 000 EUR
Fraction vendue = 20 %
Capital consommé = 8 100 EUR
Capital restant après cession = 32 400 EUR
```

Conclusion : le tracker doit enregistrer chaque cession et recalculer le capital d'acquisition restant après chaque vente.

---

## 7. Fichier des ventes

Fichier local non versionné : `data/private/ventes.csv`  
Template versionné : `data/examples/ventes.example.csv`

Colonnes recommandées :

```csv
date,btc_before,btc_sold,btc_after,btc_price_eur,portfolio_value_before_eur,sale_gross_eur,sale_fees_eur,sale_net_eur,capital_acquisition_before_eur,fraction_portfolio_sold,capital_consumed_eur,realized_pnl_eur,tax_estimate_eur,capital_acquisition_after_eur,notes
```

---

## 8. État fiscal local

Fichier local non versionné : `data/private/fiscal_state.json`

Champs recommandés :

```json
{
  "asset_universe": "BTC_ONLY",
  "capital_acquisition_restant_eur": 10000,
  "btc_total_after_last_tax_event": 0.15,
  "last_tax_event_date": "2026-05-28",
  "warnings": []
}
```

Cet état doit être recalculable à partir des achats, des entrées manuelles, des ventes et du solde BTC.

---

## 9. JSON de sortie

Fichier local non versionné par défaut : `output/crypto_output.json`

Exemple fictif :

```json
{
  "generated_at": "2026-05-28T14:32:15Z",
  "asset_universe": "BTC_ONLY",
  "btc": {
    "volume": 0.15,
    "price_eur": 90000,
    "value_eur": 13500,
    "traced_cost_basis_eur": 10000,
    "capital_acquisition_restant_eur": 10000,
    "indicative_avg_buy_price_eur": 66667,
    "indicative_unrealized_pnl_eur": 3500,
    "indicative_unrealized_pnl_pct": 35.0,
    "patrimonial_tax_estimate_eur": 1050,
    "net_value_after_tax_estimate_eur": 12450
  },
  "sources": {
    "unknown_cost_btc": 0.00123456,
    "unknown_cost_warning": true
  },
  "warnings": [
    "EXEMPLE: certains BTC ont un coût d'acquisition non documenté ; le prix moyen est indicatif."
  ]
}
```

---

## 10. Mode audit

Commande cible : audit local du portefeuille crypto.

Le mode audit doit afficher :
- nombre de transactions par source ;
- total EUR traçable par source ;
- volume BTC par source ;
- volume BTC à coût inconnu ;
- capital d'acquisition restant ;
- cessions passées ;
- capital consommé par cession ;
- warnings bloquants ou non bloquants ;
- cohérence entre solde wallet, solde plateforme et historique d'acquisition.

---

## 11. Structure des fichiers

```text
ocs77/AI-share/
├── crypto_tracker/
│   ├── tracker.py
│   ├── platform_api.py
│   ├── manual_import.py
│   ├── wallet.py
│   ├── calculator.py
│   ├── tax.py
│   ├── audit.py
│   └── config.py
├── data/
│   ├── examples/
│   │   ├── deblock_manual.example.csv
│   │   ├── hard_inputs.example.json
│   │   ├── fiscal_state.example.json
│   │   └── ventes.example.csv
│   └── private/
│       ├── deblock_manual.csv
│       ├── hard_inputs.json
│       ├── platform_ledger_cache.csv
│       ├── fiscal_state.json
│       └── ventes.csv
├── output/
│   ├── crypto_output.json
│   └── audit_crypto.html
├── .env
├── .env.example
├── .gitignore
├── requirements.txt
└── SPEC.md
```

`.gitignore` doit exclure au minimum :

```gitignore
.env
data/private/
output/*.json
output/*.html
```

---

## 12. Évolutions futures

À faire :
- rapport HTML d'audit crypto local ;
- injection contrôlée dans le reporting patrimonial ;
- bouton de mise à jour dans le cockpit local V8 ;
- exécution mensuelle locale ;
- support multi-actifs uniquement si le portefeuille crypto évolue au-delà du BTC.

À ne pas faire dans la logique privacy-first actuelle :
- exécution cloud avec secrets ou données wallet ;
- appel à un explorateur public avec la clé publique étendue ;
- commit de données réelles ;
- stockage cloud non chiffré des fichiers privés.

---

## 13. État des données

La spec ne doit contenir aucun état réel des données patrimoniales.

Les états réels sont produits localement par :
- `crypto_output.json` ;
- `audit_crypto.html` ;
- `fiscal_state.json` ;
- `ventes.csv`.

Ces fichiers restent non versionnés par défaut.

---

*Spec mise à jour — Mai 2026*

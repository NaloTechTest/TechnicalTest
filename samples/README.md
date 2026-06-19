# Samples — réponses de l'API de référence (découplage front)

Ces fichiers sont des **exemples réels** produits par l'API de référence Nalo. Ils servent à :

1. **Découpler le frontend du backend** : vous pouvez développer le front contre ces
   réponses (mock) même si votre backend n'est pas terminé.
2. **Tests d'acceptation backend** : votre API doit reproduire ces valeurs numériques.

> La forme exacte de votre réponse backend peut légèrement différer (c'est votre design) ;
> ces exemples fixent surtout **les valeurs métier** et un **contrat d'API** plausible.

## Contrat d'API

- `GET /contract/{id}/timeline` → liste d'événements normalisés, triés `(value_date, sequence)`.
  Forme normalisée par événement :
  ```
  { event_id, sequence, type, value_date, cash_impact }
  ```
  - `type` : enum stable. Le **libellé lisible** (« Versement »…) est une préoccupation de
    présentation, à dériver **côté front**.
  - `cash_impact` : mouvement de trésorerie signé, décimal en string (`"15000.00"`
    versement/dividende, `"-354.00"` retrait/taxe), `null` pour performance/frais/arbitrage
    (pas de cash direct).

  > `timeline.excerpt.json` ne contient que les **12 premiers** événements : assez pour fixer
  > la forme, à vous de produire la liste complète depuis votre backend.
- `GET /contract/{id}/state?value_date=YYYY-MM-DD` → état reconstruit à la date.
  Exemples : `state_2021-01-05.json`, `state_2021-12-31.json`, `state_2023-06-30.json`.

### Erreurs attendues
- contrat inconnu → `404`
- `value_date` invalide → `400`

Montants/poids sérialisés en **string** (décimal sans perte). Arrondi : montants 2 décimales,
poids 4 décimales.

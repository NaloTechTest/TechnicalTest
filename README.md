# 1. Test technique Full-Stack Nalo

Bienvenue, et merci pour le temps que vous consacrez à ce test.

Ce test vise à se rapprocher au maximum d’une situation réelle, telle qu’elle pourrait survenir dans un projet chez Nalo, afin de te donner un aperçu concret du quotidien d’un développeur et de ce à quoi il peut ressembler.
Le sujet est construit autour du calcul et de l’affichage de l’évolution d’un contrat client chez Nalo.

Ce test comporte **deux volets complémentaires** :

- **Partie 1 — Backend (Python / FastAPI)** : reconstruire l'état d'un contrat d'investissement à
  partir d'un historique d'événements, et l'exposer via une API.
- **Partie 2 — Frontend (Next.js / TypeScript / Tailwind)** : exposer une interface
  permettant à un utilisateur de visualiser l'évolution et la composition du contrat.

**Temps indicatif total** : entre une demi-journée et une journée.
On préfère **peu de choses bien faites** à beaucoup de choses bâclées.

**Outils d'IA : autorisés.** On vous demandera
simplement, en restitution, ce que vous avez délégué et comment vous l'avez vérifié.


### Données (communes aux deux parties)

Un contrat d'assurance-vie <sup><a href="#ref1">(1)</a></sup> vie est dite "multi-support" car le capital est réparti entre plusieurs "fonds", c’est-à-dire différents types de placements financiers pour diversifier le risque et le potentiel de rendement.
Les fonds sont identifiés par leur **ISIN**<sup><a href="#ref2">(2)</a></sup>.

L’état du contrat à un instant donné n’est pas stocké : il se reconstruit en rejouant l’historique des opérations stockées dans un **ledger**(<sup><a href="#ref3">(3)</a></sup>).

Le fichier `data/contract_ledger_events_5y.json` contient l'historique d'un contrat sur 5 ans :
`contract_id`, `currency`, et une liste `events` de 206 événements.
Chaque événement porte :

| Champ                                  | Description                                                                                            |
|----------------------------------------|--------------------------------------------------------------------------------------------------------|
| `event_id`                             | Identifiant technique (UUID) utilisé comme clé d’idempotence<sup><a href="#ref4">(4)</a></sup>         | 
| `value_date`                           | La date à laquelle une opération a un impact financier réel (format: `YYYY-MM-DD`)                     |
| `sequence`                             | Integer indiquant l’ordre des événements                                                               |
| `communication_date`, `execution_date` | Dates opérationnelles (informatives)                                                                   |
| `type`                                 | Chaque événement représente un type d’opération (<a href="#ref-dict">voir le dictionnaire suivant</a>) |
>❗ **Les événements sont ordonnés par `value_date`, puis par `sequence`**.

<h3 id="ref-dict"> Dictionnaire des types d'événements</h3>
Tous les montants / taux sont des chaînes à parser en **décimal exact**.

| Type                                          | Champs spécifiques                                                         | Effet sur l'état                                                                                                           |
|-----------------------------------------------|----------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------|
| `free_deposit`                                | `amount`, `allocation: [{isin, weight}]`<sup><a href="#ref5">(5)</a></sup> | Ajoute `amount`, réparti selon les `weight`. **Augmente le net investi.**                                                  |
| `free_withdrawal`                             | `gross_value`, `mode: "proportional"`                                      | Retire `gross_value` (le montant retiré du contrat, tel quel), au prorata des montants par fonds. **Diminue le net investi.** |
| `monthly_performance`                         | `returns: [{isin, rate}]`                                                  | Chaque fonds est multiplié par `(1 + rate)`.                                                                               |
| `management_fee`                              | `fee_rate`, `mode: "proportional"`                                         | Chaque fonds est multiplié par `(1 − fee_rate)`.                                                                           |
| `arbitrage`<sup><a href="#ref6">(6)</a></sup> | `target_allocation: [{isin, weight}]`                                      | Redistribue le **capital total** selon les poids cibles (target_allocation), sans changer le montant total investi         |
| `account_dividend`                            | `value`, `isin`                                                            | Ajoute `value` au fonds visé (dividende réinvesti).                                                                        |
| `account_social_tax`                          | `value`                                                                    | Retire `value`, au prorata des montants par fonds.                                                                         |

> `monthly_performance`, `management_fee`, `account_dividend` et `account_social_tax`
> n'affectent **pas** le net investi (ce ne sont ni des dépôts ni des retraits du client).

# 2. Architecture
## Partie 1: Backend (Python / FastAPI)

### API à implémenter

- `GET /contract/{id}/timeline`
  - La liste des événements **normalisés** et **triés** `(value_date, sequence)`
  - L'objectif : permettre au utilisateur de lire ce qui a impacté le
    contrat.
  - Forme normalisée indicative par événement :
    ```
    TimelineEntry
      event_id: string
      sequence: integer       # l’ordre des événements
      type: string            # enum stable
      value_date: date        # date de prise d'effet financière
      cash_impact: decimal?   # mouvement de trésorerie signé, null si non pertinent
    ```
- `GET /contract/{id}/state?value_date=YYYY-MM-DD`:
  - Reconstruit la situation (`state`) d’un contrat spécifique jusqu’à cette date, en prenant en compte tous les événements dont la `value_date` est inférieure ou égale à **T**.
    
    - **Date antérieure**: si `value_date` est antérieure au 1er événement, renvoyez `capital = 0`,
      `allocation = []`, `net_investi = 0`, `absolute_performance = 0`, `events_applied = 0`.
    - **Date future** : une `value_date` postérieure au dernier événement est valide (on applique
      simplement tous les événements).

> La trajectoire du capital dans le temps (user story utilisateur n°1) se construit côté front
> en interrogeant `/state` à plusieurs dates, à vous de décider lesquelles et comment.

Structure indicative de la state d'un contract :
```
ContractState
  contract_id: string
  value_date: date
  capital: decimal                 # = Σ amount (valeur totale du contrat à une date donnée)
  net_investi: decimal             # = Σ free_deposit.amount − Σ free_withdrawal.gross_value
  absolute_performance: decimal    # = capital − net_investi (capte la performance marché (dividendes − frais − taxes sociales))

  allocation: [AllocationLine]
  events_applied: integer

AllocationLine
  isin: string
  amount: decimal
  weight: decimal                  # = amount / capital` si `capital > 0`, sinon `0` (Chaque poids est arrondi indépendamment à 4 décimales)
  une somme exactement égale à 1.
```

### Contraintes techniques

1. Calculs financiers en **décimal exact** (pas de `float`).
2. Projection<sup><a href="#ref7">(7)</a></sup> **déterministe** : même entrée → même sortie, indépendamment de l'ordre du fichier.
3. Ordre d'application stable : `value_date` puis `sequence`.
4. **Idempotence** : un même `event_id` ne doit pas être appliqué deux fois.
5. Projection à T : uniquement les événements `value_date <= T`.
6. Allocation cohérente avec le capital total.
7. **Arrondi centralisé et explicite, sans perte**. Recommandé : arrondi
   **uniquement à la sérialisation** (montants 2 décimales, poids 4 décimales, half-up).

### Self-check (valeurs de référence)

Votre backend doit reproduire **exactement** ces états (arrondi de sortie : montants 2 déc.,
poids 4 déc.). Des exemples de réponses complètes sont fournis dans `samples/` (voir
`samples/README.md`).

- Après le seul 1er dépôt **`state?value_date=2021-01-05`** :

    ```
    capital=15000.00  net_investi=15000.00  absolute_performance=0.00  events_applied=1
    ```

- Fin de 1re année (tous types vus) **`state?value_date=2021-12-31`**:

    ```
    capital=16388.46  net_investi=16376.00  absolute_performance=12.46  events_applied=42
    ```

>💡 Astuce : juste après un `arbitrage`, l'allocation doit retomber **exactement** sur les
> poids cibles de l'événement.

## Partie 2: Frontend (Next.js / TypeScript / Tailwind)

### Objectif

À partir du backend que vous venez de construire, exposer une interface qui permet à un
**utilisateur** de comprendre la situation d'un contrat : comment il a évolué dans le temps,
et de quoi il est composé à une date donnée.

L'enjeu n'est **pas** la beauté de l'UI, mais comment vous **structurez la donnée côté
client, orchestrez les appels API, et organisez votre logique** (helpers, dérivations, état).

> **Découplage backend/frontend** : pour ne pas être bloqué si votre backend est incomplet,
> le contrat d'API et des **exemples de réponses réelles** sont fournis dans `samples/`.
> Vous pouvez développer le front contre ces exemples, puis le brancher sur votre backend.

### Contexte métier (user stories, par priorité)

Si le temps manque, **traitez-les dans cet ordre**,  on évalue aussi votre priorisation.

1. **(P0) Photo à une date** : me positionner à une **date de valeur** et voir le **capital**,
   le **net investi**, la **performance absolue** et l'**allocation par fonds** (montants, poids)
   à cet instant.
2. **(P1) Évolution dans le temps** : suivre la **trajectoire** du capital / net investi /
   performance sur la durée du contrat.
3. **(P2) Ce qui a impacté le contrat** : lire les **événements** (timeline) pour expliquer une
   variation au client.

### Périmètre minimal attendu

Au minimum, livrez la **P0 de bout en bout, proprement** : un écran avec un sélecteur de date de
valeur qui affiche capital / net investi / performance + l'allocation par fonds, avec les états
**chargement / erreur / vide** gérés. Une P0 carrée vaut mieux que P0+P1+P2 bâclées.

### Contraintes techniques

- **Next.js**, **TypeScript**, **Tailwind**.
- La donnée provient de **votre backend** (`/timeline` et `/state`). À vous de décider quoi
  appeler, quand, et comment composer ces appels.
- Il y a **un seul contrat** dans le jeu de données ; son `contract_id` (UUID) figure dans
  `data/…json` et dans les `samples/`. Vous pouvez le considérer comme connu/fixe.
- Le front parle à **votre** backend : rendez l'**URL configurable** (variable d'env) et gérez
  l'accès cross-origin (CORS côté API, ou proxy via une route Next) — c'est à vous.

### Livrables (frontend)

- Code dans le **même dépôt Git**.
- `README` frontend : lancement, choix d'architecture, hypothèses, et **ce que vous auriez
  fait avec plus de temps**.

# 3. Zone ouverte

Le ledger **évoluera** : de nouveaux types d'événements apparaîtront. À vous de décider
comment votre système réagit face à un `type` **non listé** dans le dictionnaire. 

Une implémentation n’est pas obligatoire pour cette situation, mais il nous intéresse de savoir quel choix vous feriez. N’hésitez pas à le documenter dans le README.


> Le jeu de données fourni ne contient que les types documentés : ce choix n'affecte donc pas
> les valeurs de référence, mais il nous intéresse autant que le reste.


# 4. Évaluation

 On est très attentifs aux éléments suivants :
- Qualité de la **modélisation/structuration de la donnée**.  
- Pertinence de l'**orchestration API**.
- L'**organisation et lisibilité du code** réutilisables, conventions claires.
- Capacité à **prioriser et argumenter** sur un sujet ouvert.
- Les **choix de structure** face à un besoin volontairement ouvert.

Les éléments suivants sont un plus, mais ne constituent pas l’objectif principal :
- La beauté visuelle, le pixel-perfect, le choix d'une lib de graphiques.
- L'exhaustivité : mieux vaut **peu de choses bien structurées** que beaucoup de bâclé.


# 5. Glossaire

<ol>
 <li id="ref1"><b>Assurance-vie</b> : un placement qui permet d’épargner et de faire fructifier de l’argent sur le long terme, tout en facilitant sa transmission.
</li> 
<li id="ref2"><b>ISIN (International Securities Identification Number)</b> : identifiant international unique d'un fonds.</li>
<li id="ref3"><b>Ledger</b> : registre comptable qui enregistre toutes les opérations financières d’un compte pour suivre son solde de façon fiable.</li>
<li id="ref4"><b>Idempotence</b> : rejouer un même événement ne change pas le résultat.</li>
<li id="ref5"><b>Allocation</b> : répartition d’un capital entre différents placements ou actifs (actions, obligations, fonds, etc.).
<li id="ref6"><b>Arbitrage</b> : transfert d'un support vers un autre, à capital constant.</li>
<li id="ref7"><b>Projection</b> : état calculé à partir de l'historique d'événements.</li>
</ol>


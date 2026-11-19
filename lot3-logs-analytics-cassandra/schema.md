# Schéma du lot 3 — Logs et analytics avec Cassandra

## Vue d’ensemble

```mermaid
flowchart TD
    SITE["Application e-commerce"] --> LOG["Événements de navigation"]
    SITE --> CMD["Commandes validées"]
    LOG --> RAW["evenements_par_jour_segment"]
    CMD --> HIST["commandes_par_client_mois"]
    RAW --> JOB["Script de calcul périodique"]
    HIST --> JOB
    JOB --> STAT["stats_par_produit_mois"]
    JOB --> TOP["top_vues_par_semaine"]
    JOB --> VENTE["ventes_par_jour"]
    HIST --> NEO["Lot 4 — Export des achats vers Neo4j"]
```

Les flèches représentent des écritures, des lectures et des calculs applicatifs. Ce ne sont pas des relations stockées ou des jointures entre tables Cassandra. Les identifiants `sku`, `client_id` et `commande_id` sont partagés avec le lot 4.

## Modèle des cinq tables

```mermaid
classDiagram
    direction TB

    class evenements_par_jour_segment {
        date jour
        int segment
        timestamp date_evenement
        uuid evenement_id
        text session_id
        text client_id
        text sku
        text type_evenement
        text terme_recherche
    }

    class commandes_par_client_mois {
        text client_id
        text mois
        timestamp date_commande
        text commande_id
        text sku
        int quantite
        decimal prix_unitaire_paye_ht
        decimal cout_unitaire_ht
    }

    class stats_par_produit_mois {
        text sku
        text mois
        date jour
        bigint nb_vues
        bigint nb_ajouts_panier
        timestamp calcule_a
    }

    class top_vues_par_semaine {
        date semaine
        int rang
        text sku
        bigint nb_vues
        timestamp calcule_a
    }

    class ventes_par_jour {
        date jour
        text sku
        bigint quantite_vendue
        decimal ca_ht
        decimal cout_achat_ht
        decimal marge_brute_ht
        timestamp calcule_a
    }
```

Ce diagramme utilise des blocs pour rendre les champs lisibles ; il représente des tables Cassandra, pas des classes applicatives. Les tables n’ont pas de clés étrangères et aucune jointure entre elles n’est supposée.

### Clés et tri

| Table | Clé de partition | Colonnes de tri dans la partition |
|---|---|---|
| `evenements_par_jour_segment` | `(jour, segment)` | `date_evenement ASC`, `evenement_id ASC` |
| `commandes_par_client_mois` | `(client_id, mois)` | `date_commande DESC`, `commande_id ASC`, `sku ASC` |
| `stats_par_produit_mois` | `(sku, mois)` | `jour ASC` |
| `top_vues_par_semaine` | `semaine` | `rang ASC` |
| `ventes_par_jour` | `jour` | `sku ASC` |

La clé de partition indique où retrouver les données. Les colonnes de tri ordonnent et distinguent les lignes de cette partition. Les segments répartissent une journée de logs ; leur nombre initial est une hypothèse à ajuster par mesure.

## Exemple fictif de parcours

```mermaid
flowchart TD
    CLIENT["Client C001 consulte CPU-7800X3D"] --> VUE["Événement vue_produit — 18 novembre 2026"]
    CLIENT --> ACHAT["Commande CMD001 — une unité achetée"]
    VUE --> TRAIT["Calcul périodique et publication"]
    ACHAT --> TRAIT
    TRAIT --> STAT["Statistiques du SKU pour la journée"]
    TRAIT --> TOP["Classement des vues de la semaine"]
    TRAIT --> MARGE["Vente HT 350 EUR — coût 280 EUR — marge brute 70 EUR"]
    ACHAT --> GRAPHE["Neo4j — C001 A_ACHETE CPU-7800X3D"]
```

Les valeurs sont fictives. Le rang dépend des vues de tous les produits et n’est pas déduit d’une seule consultation. Les résultats dérivés n’apparaissent qu’après exécution du script. Le prix payé et le coût de la vente sont des valeurs historiques, distinctes des valeurs actuelles du catalogue.

## Questions métier couvertes

| Question | Table consultée |
|---|---|
| Combien de vues pour un produit aujourd’hui ? | `stats_par_produit_mois` |
| Quels produits sont les plus consultés cette semaine ? | `top_vues_par_semaine` |
| Quelles commandes pour un client sur un mois ? | `commandes_par_client_mois` |
| Quelles ventes et quelle marge brute aujourd’hui ? | `ventes_par_jour` puis total applicatif |

## Limites à expliquer

- Le top est calculé par le script, puis écrit par rang. Cassandra ne classe pas arbitrairement tous les événements à chaque lecture.
- Les données dérivées peuvent avoir du retard ; `calcule_a` indique leur fraîcheur.
- L’historique possède une ligne par produit de commande. Le script regroupe les lignes de même `commande_id` pour l’affichage et l’export.
- Le stock actuel appartient au lot 1. Les ventes contribuent à analyser son écoulement, mais ne suffisent pas à reconstituer les réceptions et mouvements de stock.
- Les montants du scénario concernent des ventes validées avant retours tant que les ajustements ne sont pas implémentés.
- Schémas proposés pour la conception ; les tables et requêtes doivent encore être testées.

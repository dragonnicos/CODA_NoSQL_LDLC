# Refonte NoSQL de la plateforme LDLC

**Projet NSL NoSQL** : conception de l'architecture de données NoSQL de la nouvelle plateforme e-commerce LDLC.

## Membres

- HERAULT Nicolas
- DONFACK KENGUE Bertina
- RABOULI Sara

## Architecture (résumé)

La base relationnelle monolithique est remplacée par une architecture **polyglotte**, une technologie par besoin :

| Lot | Besoin | Technologie | Type |
|---|---|---|---|
| 1 | Catalogue produits | MongoDB | Document |
| 2 | Paniers et sessions | Redis | Clé-valeur |
| 3 | Logs et analytics | Cassandra | Colonnes |
| 4 | Recommandations | Neo4j | Graphe |

**Exigence complémentaire : stocks des boutiques**, rattachés au catalogue (lot 1) dans MongoDB : collection `stocks` (un document par couple magasin/référence) et résumé `stock_total` / `in_stock` dans la fiche produit. Voir [stocks-boutiques/](stocks-boutiques/).

> À compléter : description courte de l'architecture globale et du schéma.

## Sommaire des livrables

- [Schéma d'architecture globale](docs/architecture/)
- [Document de conception](docs/conception.md) (10 pages max)
- [Lot 1 : Catalogue produits (MongoDB)](lot1-catalogue-mongodb/)
- [Lot 2 : Paniers et sessions (Redis)](lot2-panier-sessions-redis/)
- [Lot 3 : Logs et analytics (Cassandra)](lot3-logs-analytics-cassandra/)
- [Lot 4 : Recommandations (Neo4j)](lot4-recommandations-neo4j/)
- [Stocks des boutiques](stocks-boutiques/)
- [Bonus : prototype d'API](api/)

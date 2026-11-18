# Lot 4 : Recommandations produits (Neo4j)

## Besoin

Suggérer aux clients des produits complémentaires ou similaires :

- « Les clients qui ont acheté ce processeur ont aussi acheté cette carte mère ».
- « Produits compatibles avec ce boîtier ».

Contraintes : relations complexes entre produits (compatibilité, gamme, marque) et parcours de relations sur plusieurs sauts (produits similaires, clients proches).

## Choix technologique : Neo4j (base de graphe)

| Critère | Neo4j | Alternative écartée |
|---|---|---|
| Parcours multi-sauts | Natif, sans jointure (coût proportionnel au sous-graphe parcouru) | SQL : jointures en cascade, coût qui explose avec la profondeur |
| Relations riches | Arêtes typées avec propriétés | MongoDB : références à suivre à la main, `$graphLookup` limité |
| Requêtes | Cypher, lisible pour les chemins | |
| Cohérence | Transactions ACID | |

Neo4j ne stocke que ce qui sert aux recommandations. La fiche produit complète reste dans MongoDB (lot 1) et l'historique des commandes dans Cassandra (lot 3). Le lien entre les bases est la référence `sku`.

## Modèle de données

Voir le diagramme dans [schema.md](schema.md).

### Nœuds

| Label | Propriétés | Rôle |
|---|---|---|
| `Produit` | `sku` (unique), `nom`, `prix`, `actif` | Cœur du graphe |
| `Categorie` | `nom` | CPU, carte mère, boîtier, RAM, etc. |
| `Marque` | `nom` | Regroupement par marque |
| `Gamme` | `nom` | Famille de produits (ex. Ryzen 7000) |
| `Client` | `client_id` (unique, pseudonymisé) | Base du « aussi acheté » |
| `Socket` | `nom` (AM5, LGA1700…) | Compatibilité CPU et carte mère |
| `FormatCarte` | `nom` (ATX, mATX, ITX) | Compatibilité carte mère et boîtier |
| `TypeRAM` | `nom` (DDR4, DDR5) | Compatibilité carte mère et barrette |

### Relations

| Relation | Propriétés | Rôle |
|---|---|---|
| `(Produit)-[:DANS_CATEGORIE]->(Categorie)` | | Classement |
| `(Produit)-[:DE_MARQUE]->(Marque)` | | |
| `(Produit)-[:DE_GAMME]->(Gamme)` | | |
| `(Client)-[:A_ACHETE]->(Produit)` | `date`, `qte`, `commande_id` | Historique d'achats |
| `(Produit)-[:UTILISE_SOCKET]->(Socket)` | | CPU |
| `(Produit)-[:SUPPORTE_SOCKET]->(Socket)` | | Carte mère |
| `(Produit)-[:AU_FORMAT]->(FormatCarte)` | | Carte mère |
| `(Produit)-[:ACCEPTE_FORMAT]->(FormatCarte)` | | Boîtier |
| `(Produit)-[:UTILISE_RAM]->(TypeRAM)` | | Barrette |
| `(Produit)-[:SUPPORTE_RAM]->(TypeRAM)` | | Carte mère |
| `(Produit)-[:ACHETE_AVEC]-(Produit)` | `nb_co_achats`, `maj` | Co-achat précalculé |

## Décisions de modélisation

### 1. Compatibilité via des nœuds intermédiaires

La compatibilité passe par un nœud (`Socket`, `FormatCarte`, `TypeRAM`) et non par une relation directe `COMPATIBLE_AVEC` entre produits.

- **Pourquoi** : une relation directe crée un nombre d'arêtes en O(n²) (chaque CPU relié à chaque carte mère compatible) et doit être mise à jour à chaque nouveau produit. Avec un nœud intermédiaire, un nouveau produit n'ajoute qu'une arête (O(n)).
- **Orientation** : `SUPPORTE_*` et `ACCEPTE_*` sont orientées, car la compatibilité n'est pas symétrique. Un boîtier ATX accepte une carte mATX, mais l'inverse est faux. Un boîtier peut avoir plusieurs `ACCEPTE_FORMAT`.
- **Limite** : une compatibilité qui ne se résume pas à un standard (exception ou incompatibilité constatée) n'est pas exprimable. On pourrait ajouter une relation `INCOMPATIBLE_AVEC` dédiée, non retenue à ce stade.

### 2. « Aussi acheté » précalculé

La relation `ACHETE_AVEC {nb_co_achats, maj}` est matérialisée par un job batch à partir de `A_ACHETE`.

- Stockée dans un seul sens (sku le plus petit vers le plus grand) pour éviter les doublons, lue sans direction.
- Seuil minimal de co-achats (par exemple 3) pour ne pas créer d'arêtes sans signification.
- **Avantage** : lecture en temps constant, adaptée à l'affichage d'une fiche produit.
- **Coût** : relation redondante à maintenir, et données cohérentes à terme (retard égal à l'intervalle du job).
- Le calcul à la volée (`Client → Produit ← Client → Produit`) reste possible et sert d'évolution ou de contrôle.

## Requêtes types

```cypher
// Cartes mères compatibles avec un boîtier
MATCH (:Produit {sku:'CASE-XXX'})-[:ACCEPTE_FORMAT]->(:FormatCarte)<-[:AU_FORMAT]-(cm:Produit)
RETURN cm.nom

// Cartes mères compatibles avec un CPU
MATCH (:Produit {sku:'CPU-7800X3D'})-[:UTILISE_SOCKET]->(:Socket)<-[:SUPPORTE_SOCKET]-(cm:Produit)
RETURN cm.nom

// Ce CPU tient-il dans ce boîtier ? (CPU, socket, carte mère, format, boîtier)
MATCH (cpu:Produit {sku:'CPU-7800X3D'})-[:UTILISE_SOCKET]->(:Socket)<-[:SUPPORTE_SOCKET]-(cm:Produit)
      -[:AU_FORMAT]->(:FormatCarte)<-[:ACCEPTE_FORMAT]-(b:Produit {sku:'CASE-XXX'})
RETURN cm.nom

// Les clients qui ont acheté ce CPU ont aussi acheté (précalculé)
MATCH (:Produit {sku:'CPU-7800X3D'})-[r:ACHETE_AVEC]-(p:Produit)
RETURN p.nom, r.nb_co_achats ORDER BY r.nb_co_achats DESC LIMIT 5

// Produits similaires : même gamme
MATCH (:Produit {sku:'CPU-7800X3D'})-[:DE_GAMME]->(:Gamme)<-[:DE_GAMME]-(p:Produit)
RETURN p.nom

// Recommandation pour un client : achats de clients qui lui ressemblent
MATCH (me:Client {client_id:'C001'})-[:A_ACHETE]->(:Produit)<-[:A_ACHETE]-(autre:Client)
      -[:A_ACHETE]->(reco:Produit)
WHERE NOT (me)-[:A_ACHETE]->(reco)
RETURN reco.nom, count(DISTINCT autre) AS score
ORDER BY score DESC LIMIT 5
```

## Job de précalcul des co-achats

```cypher
MATCH (p1:Produit)<-[:A_ACHETE]-(c:Client)-[:A_ACHETE]->(p2:Produit)
WHERE p1.sku < p2.sku
WITH p1, p2, count(DISTINCT c) AS nb
WHERE nb >= 3
MERGE (p1)-[r:ACHETE_AVEC]-(p2)
SET r.nb_co_achats = nb, r.maj = datetime()
```

Exécution périodique (par exemple chaque nuit). Sur un gros volume, traitement par lots ou par catégorie.

## Compromis (CAP et limites)

- **CAP** : Neo4j privilégie la cohérence (CP) avec des transactions ACID. Le clustering est disponible en édition Enterprise. Les recommandations tolèrent un léger retard, donc le précalcul asynchrone est acceptable.
- **Écritures** : Neo4j monte moins bien en écriture que Cassandra et ne se fragmente pas simplement. Le graphe est donc limité aux données utiles aux recommandations (les logs de navigation restent dans Cassandra).
- **Duplication** : `sku`, `nom` et `prix` sont copiés depuis MongoDB. Le catalogue fait foi, le graphe est alimenté par import.
- **Données personnelles** : le client est identifié par un `client_id` pseudonymisé, sans nom ni adresse dans le graphe.

## Alimentation du graphe

| Source | Cible | Fréquence |
|---|---|---|
| Catalogue MongoDB (lot 1) | Nœuds `Produit`, `Categorie`, `Marque`, `Gamme`, nœuds de compatibilité | À chaque évolution du catalogue (batch) |
| Commandes Cassandra (lot 3) | Relations `A_ACHETE` | Batch régulier |
| Job de co-achats | Relations `ACHETE_AVEC` | Nocturne |

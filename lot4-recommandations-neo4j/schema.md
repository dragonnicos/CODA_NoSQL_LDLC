# Schéma du graphe (Lot 4 : Neo4j)

## Vue d'ensemble

```mermaid
graph LR
    Client((Client))
    Produit((Produit))
    Categorie((Categorie))
    Marque((Marque))
    Gamme((Gamme))
    Socket((Socket))
    Format((FormatCarte))
    RAM((TypeRAM))

    Client -->|A_ACHETE<br/>date, qte, commande_id| Produit
    Produit -->|DANS_CATEGORIE| Categorie
    Produit -->|DE_MARQUE| Marque
    Produit -->|DE_GAMME| Gamme
    Produit -->|UTILISE_SOCKET / SUPPORTE_SOCKET| Socket
    Produit -->|AU_FORMAT / ACCEPTE_FORMAT| Format
    Produit -->|UTILISE_RAM / SUPPORTE_RAM| RAM
    Produit ---|ACHETE_AVEC<br/>nb_co_achats, maj| Produit
```

## Chaîne de compatibilité

```mermaid
graph LR
    CPU[Produit : CPU] -->|UTILISE_SOCKET| S((Socket AM5))
    CM[Produit : Carte mère] -->|SUPPORTE_SOCKET| S
    CM -->|AU_FORMAT| F((FormatCarte ATX))
    B[Produit : Boîtier] -->|ACCEPTE_FORMAT| F
    CM -->|SUPPORTE_RAM| R((TypeRAM DDR5))
    M[Produit : Barrette RAM] -->|UTILISE_RAM| R
```

Le chemin `CPU → Socket ← Carte mère → Format ← Boîtier` indique si un processeur tient dans un boîtier via une carte mère compatible.

## Exemple d'instance

```mermaid
graph LR
    c1((Client C001)) -->|A_ACHETE| cpu[CPU-7800X3D]
    c1 -->|A_ACHETE| cm[MB-B650-ATX]
    c2((Client C002)) -->|A_ACHETE| cpu
    c2 -->|A_ACHETE| cm
    cpu ---|ACHETE_AVEC nb=2| cm

    cpu -->|UTILISE_SOCKET| am5((AM5))
    cm -->|SUPPORTE_SOCKET| am5
    cm -->|AU_FORMAT| atx((ATX))
    case[CASE-XXX] -->|ACCEPTE_FORMAT| atx
    case -->|ACCEPTE_FORMAT| matx((mATX))
```

## Contraintes et index prévus

| Cible | Type |
|---|---|
| `Produit.sku` | Contrainte d'unicité |
| `Client.client_id` | Contrainte d'unicité |
| `Categorie.nom`, `Marque.nom`, `Gamme.nom` | Contrainte d'unicité |
| `Socket.nom`, `FormatCarte.nom`, `TypeRAM.nom` | Contrainte d'unicité |
| `A_ACHETE.date` | Index sur propriété de relation (filtre sur période) |

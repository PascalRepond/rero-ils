# Devis — Export de bibliographies structurées

## 1. Description de la fonctionnalité à développer

Ce devis détaille l'implémentation par la Fondation RERO+, le prestataire, d'une
nouvelle fonctionnalité dans RERO ILS à la demande de la Médiathèque Valais, le
client. Il s'agit de permettre la **génération automatisée d'une bibliographie
mise en page** à partir des notices du catalogue.

Concrètement, un script paramétrable est lancé en ligne de commande pour
**exporter une sélection de notices au format Markdown structuré**. Ce Markdown
est ensuite transformé, via l'outil **pandoc** et un gabarit de style, en un
**document Word (`.docx`) structuré et mis en forme**, dont l'aspect reprend
celui de la *Bibliographie valaisanne*.

Le script couvre :

- **La sélection des notices** sur la base des champs locaux (critère de
  sélection, p. ex. `valais` avec une fenêtre de dates de traitement), du type de
  document, de la langue et d'une liste de types/sous-types à exclure.
- **La mise en forme de chaque référence** selon le chablon défini par le client :
  ordre des champs, ponctuation conditionnelle (la ponctuation associée à un
  champ vide n'est pas affichée), séparateurs propres à chaque champ, cascade de
  choix de l'identifiant (Ean → Isbn → Issn → Ismn → Doi → Upc → PublisherNumber),
  affichage du premier auteur (rôle `aut`), gestion du « classement au titre »,
  surcharges de libellés (p. ex. « film documentaire »), et lien dynamique vers la
  notice publique du catalogue.
- **Le classement thématique** des notices selon le plan de classement propre à
  chaque site (VS, NE/Dewey, JU/CDU), dans l'ordre numérique des rubriques, les
  rubriques vides étant conservées.
- **La génération des index** (systématique, géographique, biographique, sujets,
  collectivités, auteur, anonymes, revues), chaque entrée renvoyant par un lien
  actif à la référence concernée.
- **L'assemblage Markdown et le gabarit Pandoc/`reference.docx`** : table des
  matières à liens actifs, disposition sur deux colonnes, styles, marges et
  polices proches de la bibliographie valaisanne.

Ne sont **pas** compris dans le développement (conformément aux specs) : page de
couverture, page de titre, introduction, en-têtes et pieds de page.

Une **variante simplifiée** (bibliographie du *Walliser Jahrbuch*) est également
décrite : une seule colonne, chablon réduit, sans index ni liens actifs, avec une
sélection restreinte (monographies et supports audio, langue allemande). Comme
indiqué dans les specs, **les frais de cette variante supplémentaire sont à la
seule charge de la Médiathèque Valais** ; elle figure donc en option séparée
ci-dessous.

### Points à clarifier avant le démarrage

Le développement dépend de quelques décisions à prendre avec le client, qui
seront traitées lors de la phase d'analyse :

- la convention exacte de stockage dans les champs locaux du **critère de
  sélection** et de la **classification** (les champs locaux sont du texte libre,
  la notation `$a/$b/$2` est une convention à formaliser) ;
- la stratégie de **filtrage par fenêtre de dates** (les champs locaux ne sont pas
  indexés comme des dates) ;
- la fourniture des **plans de classement complets** de chaque site (NE et JU ne
  sont que partiellement décrits dans les specs) ;
- la validation du **gabarit de mise en page Word** (sur la base de la
  bibliographie valaisanne 2023).

## 2. Prix

### Prestation principale — Bibliographie complète

- **Analyse des besoins et spécification de la fonctionnalité : 16h**
  - Discussions avec le client sur le besoin et les modalités d'implémentation
    (conventions des champs locaux, plans de classement, filtrage par dates,
    règles des index, gabarit de mise en page).
  - Discussions internes sur les modalités d'implémentation.
  - Rédaction des spécifications techniques.

- **Développement de la fonctionnalité : 132h**
  - Infrastructure du script en ligne de commande, gestion des paramètres et
    configuration par site (plans de classement, exclusions, séparateurs) : 14h
  - Moteur de sélection des notices (champs locaux, fenêtre de dates, filtres
    type/langue, exclusions, règles de classement au titre) : 22h
  - Moteur de mise en forme des références — le chablon (ponctuation
    conditionnelle, cascade d'identifiants, séparateurs multivalués, règles
    d'auteur, surcharges de libellés) : 32h
  - Regroupement par plan de classement (VS, NE/Dewey, JU/CDU), rubriques
    ordonnées et rubriques vides conservées : 16h
  - Génération des index (8 index, ancres, liens de retour, tri, règles de rôle) : 24h
  - Émission du Markdown et gabarit Pandoc/`reference.docx` (deux colonnes,
    styles, table des matières à liens actifs, hyperliens vers le catalogue) : 24h

- **Tests, vérifications et support technique : 28h**

**Total prestation principale : 176h**

### Option — Variante simplifiée *Walliser Jahrbuch* (à la charge de la MV)

- **Développement de la variante simplifiée** (une colonne, chablon réduit, sans
  index ni liens, sélection restreinte) **: 12h**
- **Tests et vérifications : 4h**

**Total option : 16h**

---

**Total général (principale + option) : 192h**

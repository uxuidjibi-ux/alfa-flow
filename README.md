# ALFA Flow

**Plateforme de gestion de stock, de locations et de retours pour une entreprise de
matériel d'échafaudage.** Démonstrateur fonctionnel, React + TypeScript.

> 🔗 **[Voir la démonstration en ligne](#)** — *remplacez ce lien par votre adresse*
> 📄 [Cahier des charges → décisions techniques](docs/ASSUMPTIONS.md) ·
> [Ce qui marche, ce qui est simulé](docs/MVP_VS_PRODUCTION.md) ·
> [Documentation technique](docs/PROJET.md)

---

## Le problème

Groupe Alfa loue du matériel de coffrage et d'échafaudage depuis une cour à
Sainte-Sophie. Le même article part par vagues vers plusieurs chantiers, revient
partiellement, et revient dans des états différents.

Aujourd'hui : une feuille dans la cour, une retranscription le soir, deux feuilles
comparées à la main au retour. Entre le geste et la donnée, il y a une mémoire et un
délai — et c'est là que naissent les pertes.

## Ce que fait l'application

Seize écrans couvrant le cycle complet : demande → vérification → réservation →
préparation → sortie → bon de livraison → location → retour → rapprochement →
écart → clôture.

| | |
|---|---|
| **227 articles** | extraits du PDF client par script, avec codes, familles, dimensions |
| **16 écrans** | application monopage, du tableau de bord à l'administration |
| **4 rôles** | administration, cour, secrétariat, direction — permissions distinctes |
| **22 tests** | règles métier, tous verts |
| **0 dépendance backend** | tourne dans le navigateur, prêt à migrer vers une API |

---

## La décision d'architecture qui structure tout

> **Le stock n'est pas un nombre modifiable.**

Aucun solde n'est stocké. Chaque quantité affichée — disponible, réservé, en
préparation, chez les clients, endommagé — est **repliée depuis un registre de
mouvements immuable**.

```ts
// src/services/stock.ts
export const EFFETS_MOUVEMENT: Record<TypeMouvement, EffetMouvement> = {
  SORTIE:         { enPreparation: -1, physique: -1, enCirculation:  1 },
  RETOUR_ACCEPTE: { enCirculation:  -1, physique:  1 },
  DOMMAGE:        { enCirculation:  -1, physique:  1, endommage: 1 },
  VENTE:          { enPreparation: -1, physique: -1 },  // ne revient jamais
  // …
};
```

Une correction n'écrase jamais une écriture : elle en ajoute une, avec motif et
auteur. Toute opération qui rendrait un compartiment négatif est refusée, avec un
message qui nomme l'article et les deux quantités.

**Pourquoi c'est le bon choix ici :** en cas de litige, le client doit pouvoir
reconstituer chaque solde depuis les mouvements. Un compteur mutable rend ça
impossible.

---

## Trois problèmes intéressants

### 1. Extraire un catalogue d'un PDF de photos de téléphone

Le client a fourni une liste produits en PDF — en réalité des captures d'écran, avec
des libellés concaténés par l'extraction (`GARDE-CORPS GRADIN ARRIÈRELIMON 2 MARCHES`).

Un script découpe sur les concaténations réelles — un mot-clé collé au précédent,
jamais séparé par une espace —, normalise la typographie, classe en 11 familles,
génère des SKU uniques et des quantités déterministes. **227 articles, zéro doublon.**

Les 10 photos exploitables ont été détourées par masque alpha et associées par
recoupement du texte de page et de la couleur des embouts. Les articles sans photo
fiable affichent un espace neutre — **jamais une fausse image**.

→ [`outils/generer-catalogue.mjs`](outils/generer-catalogue.mjs)

### 2. Un bon de livraison qui vaut mieux que le papier

Le client a fourni la photo d'un bon réel. Elle a répondu à trois questions que le
cahier des charges laissait ouvertes : la numérotation (`L-18203`, séquence continue —
repartir de zéro aurait créé des doublons sur des documents de preuve), l'existence de
codes articles, et la durée minimale de location.

Elle a surtout révélé une règle absente de la spécification : **une ligne peut être en
location ou en achat**. Conséquence sur le modèle — une ligne vendue quitte le parc
sans entrer en circulation, n'apparaît dans aucune location et n'est jamais attendue au
retour. Sans cette distinction, chaque vente aurait gonflé indéfiniment le « matériel
chez les clients ».

La version en ligne va plus loin que le papier : le client **vérifie chaque ligne**
avant de signer et signale un écart avec motif obligatoire. L'écart remonte le jour de
la livraison, pas trois semaines plus tard.

### 3. Le rapprochement des retours

Huit compteurs conservés séparément et jamais fusionnés. L'écart est calculé et
**présenté avant toute décision** — le système compare, l'humain tranche.

Les cas d'exception sont traités explicitement : quantité supérieure à l'attendu
(surplus en quarantaine), référence non conforme (aucune acceptation avant
vérification), photo obligatoire pour tout dommage, clôture refusée tant qu'un écart
attend une décision.

---

## Choix techniques

**React 18 · TypeScript strict · Vite · Tailwind · Zustand · Vitest · Playwright**

**Logique métier sans dépendance à React.** Tout `src/services/` est du TypeScript pur :
testable sans DOM, et portable tel quel côté serveur le jour de la migration. Aucun
composant ne calcule une quantité ni n'écrit dans le stockage.

**Persistance derrière une interface.** `DepotDonnees` a trois implémentations —
localStorage, mémoire pour les tests, et demain une API. Le reste du code l'ignore.

**Accessibilité prise au sérieux.** Cibles tactiles de 44 points minimum — l'application
s'utilise avec des gants dans une cour. Aucun statut porté par la seule couleur : chaque
badge a un marqueur de forme. Messages d'erreur nommant le champ et l'action
corrective — jamais « une erreur est survenue », plutôt *« 12 unités sont préparées
alors que 10 unités ont été approuvées »*.

---

## Vérification

```
22/22 tests métier          calcul de stock, refus de survente, retours
                            partiels successifs, permissions, persistance
18/18 tests d'interaction   Chromium 390×844, mode tactile
0     erreur JavaScript
```

Les tests d'interaction ont attrapé deux bugs invisibles à la lecture du code : un rôle
perdu à l'actualisation, et une porte d'entrée qui bloquait le lien client.

```bash
npm install && npm run dev    # http://localhost:5173
npm test                      # tests unitaires
npm run e2e                   # parcours Playwright
```

---

## Ce qui n'est pas fait, et pourquoi

Ce dépôt est un **démonstrateur**, pas un système de production. Ce qui manque relève de
l'infrastructure et est documenté ligne par ligne dans
[`docs/MVP_VS_PRODUCTION.md`](docs/MVP_VS_PRODUCTION.md) :

authentification · vérification des permissions côté serveur · signature à valeur
juridique · journal d'audit infalsifiable · sauvegardes · isolation multi-organisation.

Le sélecteur de rôle adapte l'affichage. **Ce n'est pas un contrôle d'accès**, et
l'application le dit à l'utilisateur.

[`docs/ASSUMPTIONS.md`](docs/ASSUMPTIONS.md) consigne chaque hypothèse prise faute
d'information, chaque écart assumé avec la spécification, et les questions qui restent
ouvertes chez le client.

---

## Structure

```
src/
  models/      types et référentiel — libellés, permissions, transitions
  services/    règles métier, calcul du stock, alertes, PDF, CSV, persistance
  state/       magasin Zustand et sélecteurs dérivés
  features/    un dossier par module métier
  components/  mise en page et composants réutilisables
  tests/       tests unitaires et de parcours
docs/          cahier de décisions, script de démonstration, recette
```

---

## Contexte

Conception et développement : **Djigo Djibi** — consultant CX, designer stratège UX/UI.
Validation métier : responsable opérationnel de Groupe Alfa.

Le cahier des charges fonctionnel et technique (30 sections, exigences numérotées,
matrice de traçabilité) a été rédigé en amont dans le cadre du même mandat.

Voir [LICENSE](LICENSE) — tous droits réservés.

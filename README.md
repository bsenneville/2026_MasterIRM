# TP — Estimation du mouvement respiratoire sur une IRM dynamique

**Master 2 · Traitement d'images médicales · Durée : 2 h**

Bienvenue ! Ce TP vous fait construire, pas à pas, une petite chaîne de traitement qui part d'une **séquence IRM** et aboutit à une **courbe de respiration**. Vous coderez vous-mêmes les briques essentielles (normalisation, filtre gaussien, gradient, diffusion, estimation du mouvement) et vous apprendrez à **vérifier** vos résultats.

> ⚠️ L'algorithme de mouvement utilisé est une **version pédagogique simplifiée**. Ce n'est **pas** une méthode de recalage validée pour un usage clinique.

---

## 1. Ce que vous allez apprendre

À la fin du TP, vous saurez :

1. **charger et inspecter** une séquence IRM au format NIfTI (`.nii.gz`) et vérifier vos hypothèses sur ses dimensions ;
2. **normaliser** les intensités (percentiles + `np.clip`) ;
3. **construire un filtre gaussien** (noyau, normalisation, convolution séparable) ;
4. **extraire des contours** avec le gradient, et comprendre pourquoi il faut lisser avant de dériver ;
5. relier **diffusion, Laplacien et régularisation** (diffuser = descendre le gradient d'une énergie) ;
6. **reconstruire une zone manquante** en combinant fidélité aux données et régularité ;
7. **estimer un champ de déplacement** entre deux images avec un terme de données et un terme de régularisation ;
8. **suivre le mouvement dans le temps** de deux façons (mouvement **relatif cumulé** ou **référence absolue**) et discuter la contrainte du **temps réel**.

Fil rouge du TP :

> intensités → filtre gaussien → gradients → diffusion et régularisation → champ de déplacement → signal temporel → budget de calcul

---

## 2. Ce dont vous avez besoin

| Élément | Détail |
|---|---|
| Notebook | `TP_mouvement_respiratoire_IRM_ETUDIANT.ipynb` |
| Données | `abdomen_dynamique.nii.gz` : **200 images** 2D, une image toutes les **0,3 s** (soit 60 s d'enregistrement) |
| Python | 3.11 ou plus récent |
| Bibliothèques | `numpy`, `scipy`, `matplotlib`, `nibabel`, `ipywidgets` |

### Installation

Si une bibliothèque manque, exécutez dans la **première cellule** du notebook (ou dans un terminal) :

```python
%pip install -q nibabel scipy matplotlib ipywidgets
```

### Où placer le fichier de données ?

Le notebook cherche `abdomen_dynamique.nii.gz` dans cet ordre :

1. le **même dossier** que le notebook ;
2. un sous-dossier `./data/` ;
3. `../data/`.

Si le fichier n'est pas trouvé, une erreur vous indique le dossier courant : déplacez simplement le fichier au bon endroit.

---

## 3. Comment travailler dans le notebook

1. Ouvrez le notebook dans Jupyter / JupyterLab.
2. Exécutez les cellules **dans l'ordre**, avec `Shift + Entrée`. Ne sautez pas de cellule : les suivantes en dépendent.
3. Les cellules marquées **`À COMPLÉTER`** contiennent des `...` : **remplacez-les par votre code**. Seules ces lignes sont à écrire ; le reste est fourni.
4. Après chaque exercice, une cellule d'**auto-vérification** affiche :
   - ✅ votre réponse est cohérente ;
   - ❌ il y a un problème, avec une **piste** pour le corriger.
5. Répondez aux **questions** en double-cliquant sur la cellule *« ✍️ Vos réponses »* puis en écrivant sous le texte.
6. À la toute fin, une cellule **récapitule** toutes les auto-vérifications.

### Légende

| Symbole | Signification |
|---|---|
| 🟩 cellule de code | à exécuter (et à compléter si elle contient `...`) |
| 🟦 cellule de texte | explications, rappels de cours, questions |
| ✅ / ❌ | résultat d'une auto-vérification |
| 💡 | astuce |
| ⚠️ | piège à éviter |

### Barres de couleur

**Toutes les images affichées ont une barre de couleur.** Prenez l'habitude de la lire : elle indique la **plage d'intensité** de l'image. Deux images qui *paraissent* identiques peuvent avoir des échelles très différentes.

---

## 4. Les huit missions

| # | Mission | Vous codez | Vous devez comprendre |
|---|---|---|---|
| 1 | **Charger et inspecter** | le choix de l'axe temporel, le kymogramme | l'ordre des axes d'un NIfTI, comment vérifier ses hypothèses, où se situe le mouvement |
| 2 | **Normalisation** | la normalisation par percentiles avec `np.clip` | pourquoi une IRM n'a pas d'échelle absolue ; ce qu'apporte (et ne crée pas) la normalisation |
| 3 | **Filtre gaussien** | le noyau gaussien normalisé | rôle de $\sigma$ ; compromis bruit / contours ; filtre passe-bas ; séparabilité |
| 4 | **Gradient et contours** | `np.gradient`, la norme du gradient, le lissage avant dérivation | pourquoi le bruit crée de faux contours |
| 5 | **Diffusion et régularisation** | la mise à jour $u \leftarrow u + \alpha\,\Delta u$ | lien Laplacien ↔ énergie de régularité ; courbe d'énergie ; stabilité ($\alpha \le 1/4$) ; idée de Perona-Malik |
| 6 | **Zone manquante** | la mise à jour des pixels manquants et le maintien des pixels connus | problème inverse : fidélité aux données + régularité ; limites de la reconstruction |
| 7 | **Champ de déplacement** | une descente de gradient (résidu, Laplaciens, mises à jour) | modèle de mouvement linéarisé, rôle de $\lambda$, problème d'ouverture, limites |
| 8 | **Suivi temporel** | le signal dans une région d'intérêt, la somme cumulée (approche A), la comparaison à une image de référence (approche B) | avantages et inconvénients de A et B ; dérive ; temps réel |

### Mission 8 en deux mots

- **Approche A — relatif cumulé** : on estime le mouvement entre **images consécutives** (petits déplacements), puis on **additionne**. Simple et rapide, mais les erreurs **s'accumulent**.
- **Approche B — référence absolue** : on compare **chaque image à une même image de référence**. Pas d'accumulation, mais les déplacements à estimer sont **plus grands**.

Vous tracerez les deux courbes et leur différence pour **juger par vous-mêmes**.

---

## 5. Ce qui est attendu de vous

À la fin de la séance, vous devez avoir :

- [ ] **exécuté le notebook en entier**, de haut en bas, **sans erreur** ;
- [ ] **complété toutes les cellules `À COMPLÉTER`** (plus aucun `...` restant) ;
- [ ] obtenu des **✅ partout** dans la cellule de bilan final (ou, pour chaque ❌, **compris pourquoi**) ;
- [ ] **répondu aux questions** de chaque mission, **avec vos propres mots**, en vous appuyant sur ce que vous avez observé dans les figures ;
- [ ] été capable d'**expliquer** : pourquoi on lisse avant de dériver, ce que représente l'énergie d'une diffusion, le rôle de $\lambda$, et la différence entre les approches A et B.

Les modalités de **rendu et d'évaluation** (fichier à déposer, date, barème) sont précisées par votre enseignant.

### Ce qui est apprécié

- Des réponses **argumentées** : citez ce que vous voyez (« la barre de couleur montre que… », « l'énergie décroît puis… »).
- Un **esprit critique** : l'algorithme est simplifié, dites **où** et **pourquoi** il est limité.
- Le **diagnostic** : en cas de ❌, relisez le message d'aide avant de demander de l'aide.

---

## 6. Conseils pratiques

- **Ne cherchez pas à tout optimiser** : les formules sont données, concentrez-vous sur le sens.
- Si une cellule échoue avec l'erreur `unsupported operand type(s) … 'ellipsis'`, il reste un `...` à remplacer.
- Si une cellule complétée plus tôt vous semble « cassée », **ré-exécutez le notebook depuis le début** (menu *Kernel → Restart & Run All*, une fois tous les `...` remplacés).
- Les boucles de la Mission 8 font environ **200 estimations de mouvement** et peuvent prendre **plusieurs dizaines de secondes** : c'est normal. Pour tester vite, vous pouvez temporairement les limiter à quelques images, puis remettre la plage complète.
- La **région d'intérêt** de la Mission 8 est un rectangle par défaut : adaptez-la à la zone où le mouvement est visible (voir la carte d'écart-type de la Mission 1).
- Si l'image apparaît **couchée** : passez `TRANSPOSER = True` dans la cellule de la Mission 1.

## 7. Erreurs fréquentes

| Symptôme | Cause probable | Remède |
|---|---|---|
| `FileNotFoundError` | le fichier n'est pas au bon endroit | le placer à côté du notebook ou dans `./data/` |
| ❌ « H2 : un seul axe de longueur 200 » | plusieurs axes de même taille | demander à l'enseignant quel axe est le temps |
| ❌ Mission 4 « ordre des dérivées » | `gx, gy = np.gradient(...)` | `np.gradient` renvoie d'abord **gy** (lignes), puis **gx** (colonnes) |
| Mission 5 : l'énergie **remonte** ou l'image explose | mauvais signe ou $\alpha > 1/4$ | `u + alpha * lap`, avec $\alpha \le 0{,}25$ |
| Mission 7 : l'énergie **remonte** ou valeurs `NaN` | erreur de signe dans la descente de gradient | revoir $u \leftarrow u - \eta\,(I_x r - \lambda\,\Delta u)$ |
| Mission 8 : signal plat | la région d'intérêt est hors de la zone qui bouge | repositionner la ROI |
| Animation absente | environnement sans JavaScript | utiliser Jupyter ou JupyterLab dans un navigateur |

---

## 8. Pour réviser (autoévaluation)

À la fin du TP, essayez de répondre **sans regarder le notebook** :

1. Pourquoi les percentiles 1 et 99 plutôt que le minimum et le maximum ?
2. Que devient un contour quand $\sigma$ augmente ? Et le bruit ?
3. Pourquoi le gradient amplifie-t-il le bruit ?
4. Pourquoi *diffuser* revient-il à faire une **descente de gradient** sur $\tfrac12\|\nabla u\|^2$ ?
5. Qu'indique une courbe d'énergie qui **remonte** ?
6. Dans $E(u,v)=\tfrac12\sum r^2+\tfrac{\lambda}{2}\sum(|\nabla u|^2+|\nabla v|^2)$, quel terme utilise les images ? lequel favorise un champ lisse ? que fait $\lambda$ ?
7. Pourquoi un champ plus lisse n'est-il pas forcément plus exact ?
8. Quel est l'avantage de l'approche A ? son défaut ? Même question pour B.
9. Pourquoi « calcul plus rapide que l'acquisition » ne suffit-il pas pour parler de **temps réel** ?

---

Bon TP ! 🫁

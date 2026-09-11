# TD DORA : Réponses

Dépôt étudié : `excalidraw/excalidraw`. Fenêtre retenue : 90 jours glissants (depuis le 2026-06-13, calculée par le collecteur au moment de l'exécution).

## Phase 0 : le contrat, avant l'outil 

Le contrat `dora-definitions.yml` est rendu à part, dans le même dépôt. Figé le 11/09/2026, avant toute lecture de données API.


| Métrique | Valeur obtenue |
|---|---|
| Deployment frequency | 0,167 déploiement/jour (15 déploiements sur la fenêtre) |
| Délai médian entre deux déploiements | 108,6 h, soit 4,5 jours |
| Change lead time P50 | 43,8 h |
| Change lead time P90 | 214,8 h, soit 8,9 jours |
| Commits analysés / nombre de lots | 79 / 14 |

---

## Phase 1 : reconnaissance

**1. Combien de valeurs différentes d'`environment` trouvez-vous ? Listez-les.**

Il y en a huit :

- `Preview – excalidraw`
- `Production – excalidraw`
- `Preview – excalidraw-package-example`
- `Production – excalidraw-package-example`
- `Preview – excalidraw-package-example-with-nextjs`
- `Production – excalidraw-package-example-with-nextjs`
- `Preview – docs`
- `Production – docs`

**2. Lesquelles correspondent à une mise en production au sens de DORA ? Lesquelles doivent être exclues, et pourquoi ?**

Seule `Production – excalidraw` correspond au produit lui-même. Tout le reste doit être exclu :

- les trois `Preview – *` 
- `Production – excalidraw-package-example` et `Production – excalidraw-package-example-with-nextjs`
- `Production – docs`

**3. Si vous comptiez tous ces déploiements, de quel facteur votre deployment frequency serait-elle surestimée ?**

Un seul des 100 dernier déploiements est un vrai déploiement en production (excalidraw). Sans filtrage, la fréquence calculé est 100 fois supérieure à la réalitée

**4. Retrouvez la ligne correspondante dans le tableau des erreurs d'implémentation (7.3 du support).**

On remarque un écart de 100x sur ce cas précis, soit bien plus que le facteur 10 généralement indiqué par le support. Cela prouve que l'impact dépend directement du projet et de sa gestion des prévisualisations

**5. Que contient le tableau des statuts d'un déploiement ? Pourquoi l'existence d'un déploiement ne suffit-elle pas à établir qu'un changement est arrivé en production ?**

Le tableau des statuts contient la succession des états qu'a traversés un déploiement (`pending`, `in_progress`, `success`, `failure`, etc.), chacun avec son propre horodatage. La création d'un déploiement n'est qu'une déclaration d'intention : un déploiement peut être créé puis échouer, rester bloqué, ou être remplacé sans jamais atteindre l'état `success`. Seul le statut `success` atteste qu'un changement est réellement arrivé en production

**6. Quel horodatage faut-il retenir : le `created_at` du déploiement, ou celui du statut `success` ? Votre contrat de phase 0 avait-il tranché ?**

Conformément au contrat de phase 0 , on se base sur le created_at du statut success. Sur les cas réels analysés comme excalidraw, les deux dates tombent exactement au même moment. Par contre, ce n'est pas une règle absolue : sur un déploiement avec approbation manuelle ou en plusieurs étapes, l'écart peut être de plusieurs heure

---

## Phase 2 : collecte outillée

**7. Rapportez le nombre de commits au nombre de lots. Que vaut la taille moyenne d'un lot ? Que dit le chapitre 6.1 du support de ce chiffre ?**

79 commits pour 14 lots, soit environ 5,6 commits par lot. . Une taille de lot de l'ordre de 5 à 6 commits est plutôt favorable : elle reste loin des lots de dizaines de commits qui font exploser le lead time

**8. Comparez P50 et P90. Quel est le rapport entre les deux ? D'après la section « médiane et percentiles » de l'annexe B, que signale un tel écart, et que ne signale-t-il pas ?**

P50 = 43,8 h, P90 = 214,8 h. Le rapport est d'environ 4,9. L'annexe B indique qu'un P50 stable avec un P90 qui s'envole est le signal d'une catégorie de changements qui coince (migrations, changements inter-équipes, validations externes), pas d'une dégradation générale de tout le flux. Ce n'est donc pas le signe que « tout va mal » en moyenne, mais plutôt une piste de cartographie du flux de valeur pour identifier quelle sous-catégorie de changements traîne.

**9. La deployment frequency vous place dans quel ordre de grandeur au regard de la distribution 2024 (4.2) ? Tenez compte du chapitre 4.1 avant de conclure.**

Avec 0,167 déploiement/jour, soit un déploiement tous les six jours environ, on est loin du cluster *elite* et plus proche des clusters *medium* ou *low* si on ne regarde que ce chiffre isolémeent. Le chiffre donne un ordre de grandeur, pas un verdict de classement

**10. Le collecteur affiche aussi le délai médian entre deux déploiements. Pourquoi cette formulation est-elle préférable à la fréquence brute pour une équipe qui déploie peu (2.3) ?**

Le support indique en 2.3 : « Pour des équipes qui déploient rarement, la fréquence brute est trompeuse. Préférez le délai médian entre deux déploiements, plus lisible et moins sensible aux fenêtres arbitraires. » Avec seulement 15 déploiements sur 90 jours, un déploiement de plus ou de moins à la marge de la fenêtre change sensiblement le ratio brut, alors que le délai médian (4,5 jours ici) reste une lecture stable du rythme réel.

**11. Trois métriques sur cinq s'affichent `n/a`. Lesquelles ? Qu'ont-elles en commun ?**

Failed deployment recovery time, change fail rate, et deployment rework rate. Les deux premières exigent un rattachement entre un incident et le déploiement qui l'a causé, absent en phase 2 (pas encore de `--incident-label`). La troisième exige un marqueur de retravail (`--rework-prefix`), lui aussi absent à ce stade. Les trois ont en commun de dépendre d'une donnée qui n'existe pas dans l'historique public brut du dépôt : elles se construisent, elles ne se lisent pas.

**12. Le support désigne un maillon faible (5.1). Lequel, et pourquoi ne peut-il pas être reconstitué à partir des données publiques du dépôt ?**

Le maillon faible désigné par le support est « le lien entre un incident et le déploiement qui l'a causé » . Ce lien n'existe dans aucun champ natif de l'API GitHub publique. Un déploiement n'a pas de pointeur vers les incidents qu'il a provoqués, et une issue n'a pas de pointeur natif vers le déploiement fautif. Reconstituer ce lien suppose soit une convention déclarative que l'équipe alimente elle-même (label, champ `caused_by`), soit une intégration avec un outil d'astreinte, deux choses qu'un dépôt tiers non exploité en production ne peut pas fournir

**13. Un collègue propose : un déploiement suivi d'un autre moins de 24 h après est un échec. Donnez deux situations où cette règle se trompe, une dans chaque sens.**

Faux positif : sur l'environnement `Preview – excalidraw`, des dizaines de déploiements se succèdent en quelques heures à chaque PR poussée. Ce n'est pas un échec, c'est un flux de travail par petits lots parfaitement sain, et la règle le classerait à tort en échec systématique.

Faux négatif : un déploiement qui casse la production peut rester en l'état plusieurs jours si l'équipe met du temps à diagnostiquer le problème avant de livrer le correctif. Le prochain déploiement arriverait alors après 24 h, et la règle ne détecterait aucun échec alors qu'il y en a bel et bien eu un

---

## Phase 3 : le proxy et ses limites

**14. Combien d'issues `bug` le collecteur trouve-t-il sur la fenêtre ? Combien sont rattachées à un déploiement ?**

23 issues portant le label `bug`, dont 0 rattachées à un déploiement (aucune ne contient de ligne `caused_by:` pointant vers un identifiant de déploiement connu)

**15. Dans l'interface GitHub : combien d'issues toutes catégories ont été ouvertes sur ces 90 jours ? Combien portent le label `bug` depuis la création du dépôt ?**

141 issues toutes catégories ouvertes depuis le 13/06/2026 (`created:>=2026-06-13`). 765 issues portent le label `bug` depuis la création du dépôt

**16. Confrontez les trois nombres. Que s'est-il passé dans ce projet ?**

La recherche croisée `label:bug created:>=2026-06-13` renvoie 0 résultat. Autrement dit, sur les 141 issues ouvertes ces 90 derniers jours, aucune ne porte le label `bug`, alors que le label a été appliqué 765 fois dans toute l'histoire du dépôt. Le label `bug` n'est donc plus utilisé sur les nouvelles issues, alors qu'il l'a été massivement par le passé. Les 23 issues que le collecteur trouve ne sont pas des bugs récents. Ce sont donc d'anciennes issues `bug`, rouvertes ou commentées récemment, pas de nouveaux incidents

**17. Le chapitre 7.3 énumère trois explications à un taux d'échec de 0 %. Aucune ne décrit ce cas : formulez la quatrième.**

Le problème c'est que le label bug n'est simplement plus utilisé par l'équipe sur les nouvelles issues. La quatrième explication, c'est que le marqueur a arrêté d'être appliqué

**18. Quelle métrique le support décrit-il comme la plus sensible à la discipline de saisie (2.8) ? Ce rapprochement vous paraît-il fortuit ?**

Le support décrit "le deployment rework rate". Le rapprochement n'est pas un hasard. On retrouve exactement le même problème qu'aux questions 14 à 17 : une métrique qui dépend d'un label posé à la main. Si l'équipe arrête de l'utiliser correctement, la métrique se fausse sans que rien ne le signale

---

## Phase 4 : produire la donnée manquante

**19. Que pouvez-vous calculer maintenant que vous ne pouviez pas calculer avant ?**

Sur mon fork, avec 7 déploiements dont un hotfix et un incident rattaché, les trois métriques qui restaient à `n/a` affichent maintenant une valeur. Change fail rate à 14,3 % (1 déploiement sur 7 a causé un incident), failed deployment recovery time à environ 9 minutes 22 (l'écart entre l'ouverture et la fermeture de mon issue), et deployment rework rate à 14,3 % (1 déploiement sur 7 porte le préfixe `hotfix/`). Le rattachement `caused_by:` dans le corps de l'issue, c'est exactement ce qui manquait en phase 3, le lien entre l'incident et le déploiement qui l'a causé

**20. Combien de temps a demandé la production de cette donnée, comparé au temps passé à tenter de la déduire en phase 3 ?**

La phase 3 n'a rien donné d'exploitable, juste un diagnostic négatif. La phase 4 a demandé d'écrire un workflow, six déploiements espacés, un label, une issue, et une attente volontaire avant de la fermer. Environ 1 heure de manipulation, surtout de l'attente et de la vérification. C'est plus long que la simple lecture de la phase 3, mais c'est la seule des deux qui donne un chiffre exploitable. Ça montre bien ce que coûte une donnée manquante, ce n'est pas cher à mesurer, c'est cher à ne pas avoir

**21. Votre change fail rate est-il représentatif ? Que faudrait-il pour qu'il le devienne ?**

Non. 14,3 % calculé sur 7 déploiements et un seul incident simulé, ça ne représente rien statistiquement. Le fait que change fail rate et deployment rework rate tombent pile sur la même valeur une coincidence, pas un vrai résultat. Pour que ce soit représentatif il faudrait un vrai historique sur plusieurs mois, des déploiements et incidents naturels, un rattachement tenu à jour en continu par l'équipe, et assez de volume pour qu'un ou deux évènements isolés ne fassent pas bouger le chiffre de plusieurs points

---

## Phase 5 : lecture critique des outils

**22. Ces outils calculeraient-ils le change fail rate d'excalidraw ? Sur quelle donnée s'appuieraient-ils ? Votre conclusion de la phase 2 change-t-elle parce que l'outil est professionnel plutôt qu'un script de 200 lignes ?**

Le maillon faible n'est pas dans l'outil, il est dans la donnée source. Un outil pro affiche le chiffre avec plus de confort (dashboard, historique), il ne le rend pas plus vrai

**23. Le chapitre 5.5 propose l'ordre suivant : Quick Check, puis conversation d'équipe, puis instrumentation. Au vu de votre demi-journée, pourquoi l'outillage arrive-t-il en dernier ?**

Parceque l'essentiel du travail de cette demi-journée s'est joué avant l'outil

**24. Four Keys était la référence citée dans la plupart des tutoriels jusqu'en 2024. Quelle habitude de travail cela suggère-t-il avant d'adopter un outil trouvé en ligne ?**

Vérifier l'état réel du dépôt avant de s'y fier, date du dernier commit, date du dernier tag publié, présence ou non d'un bandeau d'archivage. Un tutoriel a une date de publication mais pas de date de péremption affichée

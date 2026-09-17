# TD DORA - réponses

## dora-definitions.yml 

```yaml
application: excalidraw

deploiement:
  compte_comme_deploiement: "un évènement de l'API Deployments dont l'environment est exactement Production – excalidraw et qui a atteint le statut success"
  exclut: ["les environnements Preview (builds de preview à chaque PR), les Production des packages d'exemple (excalidraw-package-example et excalidraw-package-example-with-nextjs), et Production – docs"]

  horodatage: "le created_at du premier statut success, pas le created_at du déploiement"

changement:
  point_de_depart: "committer date sur la branche par défaut, pas l'ouverture de la PR"

incident:
  definition: "proxy : une issue portant le label bug. Ce n'est pas la métrique DORA, on n'exploite pas la production d'excalidraw, on ne peut pas savoir ce qui gêne vraiment les utilisateurs"
  source: "les issues du dépôt"
  debut: "l'ouverture du ticket"
  fin: "la fermeture du ticket"
  rattachement_deploiement: "une mention caused_by: <id> dans le corps de l'issue, qui rattache l'incident au déploiement responsable"

rework:
  marqueur: "une ref de déploiement qui commence par hotfix/"

fenetre_de_reference: "90 jours glissants"
agregation: "médiane et P90, jamais la moyenne"

revision_envisagee: |
  La partie que je changerais si je devais refaire le contrat, c'est surtout le
  rattachement entre un incident et le déploiement responsable.

  Pour la détection des incidents, le proxy avec le label bug fonctionnait :
  le collecteur en a trouvé 23 sur la fenêtre. Par contre, aucune de ces issues
  n'était reliée à un déploiement. Du coup, impossible de calculer le change fail
  rate, le recovery time et le rework rate : les trois sortaient en n/a.

  En fait, le problème n'était donc pas vraiment la détection des incidents,
  mais le lien entre l'incident et le déploiement qui l'aurait causé. Je ne
  m'en suis rendu compte qu'en regardant les résultats.

  Si c'était à refaire, je vérifierais avant de figer le contrat que la donnée
  existe réellement. Par exemple, je regarderais sur un échantillon combien
  d'issues contiennent déjà un caused_by.

  J'ajouterais aussi que le proxy bug peut évoluer dans le temps. Le 16/09,
  j'avais 23 issues avec ce label sur la fenêtre, puis une recherche faite deux
  jours plus tard n'en retournait plus aucune. Les mainteneurs peuvent retirer
  le label pendant le triage ou à la fermeture.

  Ce qui m'a surtout surpris, c'est qu'on peut avoir beaucoup de données
  disponibles dans GitHub et quand même ne pas pouvoir calculer une métrique
  simplement parce qu'il manque un petit lien entre deux types de données.
```

## Phase 1 - reconnaissance

1. J'ai trouvé huit environnements différents dans l'API Deployments :

   - Production – excalidraw
   - Production – excalidraw-package-example
   - Production – excalidraw-package-example-with-nextjs
   - Production – docs
   - Preview – excalidraw
   - Preview – excalidraw-package-example
   - Preview – excalidraw-package-example-with-nextjs
   - Preview – docs

2. Pour moi, une seule correspond réellement à une mise en production au sens DORA : Production – excalidraw.

   Les autres doivent être exclues :

   - Preview – excalidraw correspond aux builds de preview générés à chaque PR. Il y en avait 892 sur les 1000 derniers déploiements.
   - Production – excalidraw-package-example et Production – excalidraw-package-example-with-nextjs concernent des applications d'exemple qui utilisent le package npm, et pas directement le produit Excalidraw.
   - Production – docs correspond au déploiement de la documentation.

3. On arrive à une surestimation d'au moins ×67 si on prend tous les environnements.

   Sur la fenêtre de 90 jours, j'ai trouvé 1000 déploiements tous environnements confondus, avec en plus la limite de l'API qui m'a arrêté avant la fin de la fenêtre, contre seulement 15 vrais déploiements de Production – excalidraw.

   En regardant le rythme des previews, avec environ une centaine par jour contre une dizaine de déploiements de production par mois, l'ordre de grandeur serait plutôt autour de ×100.

   C'est donc cet ordre de grandeur que je retiens.

4. Dans le tableau des erreurs d'implémentation (7.3), l'erreur serait de compter les déploiements des environnements de préproduction comme des déploiements de production.

   Ça gonflerait artificiellement la fréquence de déploiement.

5. Le tableau des statuts contient au moins in_progress, puis success ou failure.

   Le simple fait qu'un déploiement existe ne suffit donc pas. Il faut vérifier qu'il a bien atteint le statut success pour pouvoir considérer que le changement est réellement arrivé en production.

   Un déploiement qui reste in_progress ou qui finit en failure ne doit pas être compté comme une mise en production réussie.

6. J'ai choisi le created_at du statut success.

   C'est ce que j'avais défini dans mon contrat en phase 0 : l'horodatage correspond au premier statut success, et pas au moment où le déploiement a été créé.

## Phase 2 - collecte outillée

J'ai utilisé une fenêtre de 90 jours, à partir du 18/06/2026, et uniquement l'environnement Production – excalidraw.

| Métrique | Valeur obtenue |
|---|---|
| Deployment frequency | 0,167 / jour (15 déploiements) |
| Délai médian entre deux déploiements | 4,1 jours (97,7 h) |
| Change lead time P50 | 43,2 h |
| Change lead time P90 | 8,5 jours (202,9 h) |
| Commits analysés / lots | 74 / 14 (5,3 commits par lot) |

7. J'ai trouvé 74 commits répartis sur 14 lots, soit environ 5,3 commits par lot.

   Le chapitre 6.1 présente le travail en petits lots comme un levier important sur les cinq métriques. Ici, on voit qu'Excalidraw travaille déjà avec des lots relativement regroupés, puisque plusieurs commits peuvent appartenir au même lot.

8. Le P50 est de 43,2 h alors que le P90 est de 8,5 jours.

   Le rapport entre les deux est d'environ 4,7. D'après l'annexe B, un écart de cette taille montre qu'il y a une queue assez longue dans la distribution.

   En gros, une bonne partie des lots passe assez rapidement, mais certains restent beaucoup plus longtemps en attente.

   Par contre, ces deux chiffres ne permettent pas de savoir pourquoi. Ils ne disent pas si l'attente vient de la review, de la CI, de la priorisation ou d'autre chose.

   Il faudrait donc regarder les lots individuellement pour comprendre ce qui distingue ceux qui prennent longtemps des autres.

9. La fréquence est de 0,167 déploiement par jour, soit environ un déploiement tous les 6 jours.

   Si je compare uniquement cette valeur à la distribution 2024, elle se situe entre les catégories medium et low performer pour la fréquence. Mais le chapitre 4.1 précise bien que ces catégories sont surtout des repères et pas une grille de notation fixe.

   Les seuils peuvent changer selon les années et surtout, on ne peut pas juger une situation à partir d'une seule métrique. Il faut regarder les cinq métriques ensemble.

10. Je trouve le délai médian entre deux déploiements plus parlant que la fréquence brute quand les déploiements sont peu nombreux.

    0,167 déploiement/jour est assez abstrait. Dire qu'il y a environ un déploiement tous les 6 jours est plus facile à comprendre et à comparer.

    La fréquence brute devient surtout intéressante quand on arrive à des déploiements quotidiens ou plusieurs fois par jour.

11. Les trois métriques qui sortent en n/a sont :

    - le change fail rate ;
    - le failed deployment recovery time ;
    - le deployment rework rate.

    Elles ont toutes besoin du lien entre un incident et un déploiement, ou entre un déploiement de correction et le déploiement initial.

    Comme ce lien n'existe pas dans les données disponibles, il manque les informations nécessaires pour calculer ces trois métriques.

12. Le maillon faible que j'ai identifié est donc le rattachement entre un incident et le déploiement qui l'a causé.

    Je ne pouvais pas le reconstituer correctement à partir des données publiques du dépôt. Un vrai incident de production est normalement identifié grâce au monitoring, aux erreurs, aux retours utilisateurs, etc.

    Le dépôt GitHub me permet de voir les déploiements et les issues, mais il ne me permet pas de savoir si un problème rencontré par les utilisateurs vient réellement d'un déploiement précis.

13. Une règle du type « un déploiement suivi d'un autre moins de 24 h après correspond à un échec » peut facilement se tromper.

    Premier cas, un faux positif : deux déploiements peuvent être faits le même jour pour deux changements totalement indépendants. Par exemple, une petite correction de texte puis une feature. La règle compterait quand même le deuxième comme un correctif alors qu'il n'y a pas eu d'échec.

    Deuxième cas, un faux négatif : une régression peut être introduite lundi mais seulement détectée et corrigée trois jours plus tard. Comme le correctif arrive 72 h après, la règle des 24 h ne détecterait rien.

## Phase 3 - le proxy et ses limites

14. Le collecteur trouve 23 issues avec le label bug sur la fenêtre étudiée, mais aucune n'est rattachée à un déploiement.

15. Dans l'interface GitHub, j'ai trouvé :

    - 140 issues toutes catégories confondues créées sur les 90 jours, dont 122 ouvertes et 18 fermées ;
    - 765 issues avec le label bug depuis la création du dépôt, dont 211 ouvertes et 554 fermées.

    Il y a aussi un point qui m'a posé problème : le 16/09, la recherche avec label:bug sur la fenêtre donnait 23 issues, alors qu'une recherche faite deux jours plus tard n'en donnait plus aucune.

    Cela montre que le label peut être retiré par les mainteneurs pendant le triage ou à la fermeture d'une issue. Le résultat dépend donc aussi de l'état actuel des labels.

16. Les trois chiffres montrent bien le problème.

    Le label bug existe et permet de retrouver des issues, mais aucune de celles que j'ai trouvées n'était reliée à un déploiement.

    Donc les incidents sont bien tracés sous forme d'issues, mais la partie qui manque est le lien vers le déploiement responsable.

    Autrement dit, quelqu'un peut créer une issue pour signaler un bug sans jamais indiquer quel déploiement l'a provoqué.

    C'est exactement le maillon faible que j'avais identifié à la question 12.

17. Le chapitre 7.3 donne trois explications possibles à un fail rate de 0 %. Aucune ne correspond vraiment à mon cas.

    Ici, le problème est plutôt que le marqueur permettant de relier l'incident au déploiement n'a jamais été produit.

    Donc un résultat n/a ne signifie pas « il n'y a pas eu d'échec ». Il signifie simplement que je n'ai pas les données nécessaires pour le déterminer.

    C'est un point important : un outil peut toujours afficher un résultat, mais ce résultat ne représente pas forcément la réalité si la donnée de départ est incomplète.

18. Le support (2.8) présente le deployment rework rate comme une métrique particulièrement dépendante de la qualité de la saisie des données.

    Ça correspond bien à ce que j'ai observé ici. Pour calculer le rework rate, il faut savoir quel déploiement correspond à un correctif après incident.

    C'est donc le même lien incident → déploiement qui manque également pour le fail rate.

    Les deux métriques de stabilité deviennent donc impossibles à calculer dès que cette information n'est pas renseignée.

## Phase 4 - produire la donnée manquante

19. Une fois que j'ai ajouté la donnée manquante, les trois métriques deviennent calculables :

    - change fail rate : 1/6 = 16,7 % ;
    - failed deployment recovery time : 1,2 h ;
    - deployment rework rate : 1/6 = 16,7 %.

    La différence avec la phase 2 vient donc uniquement du fait que j'ai enfin produit le lien entre l'incident et le déploiement.

20. Sur le fork, j'ai passé environ 2 heures à mettre en place le workflow, produire les 5 déploiements puis le hotfix, simuler l'incident avec caused_by dans le corps de l'issue et nettoyer les doublons.

    À l'inverse, j'ai passé environ 40 minutes en phase 3 à essayer de déduire la même information à partir des issues avec le label bug, pour finalement obtenir n/a.

    La différence est assez claire : produire directement la donnée demande un peu de travail au départ, mais le workflow peut ensuite être réutilisé. Essayer de reconstruire cette donnée après coup ne permet pas forcément d'obtenir un résultat.

21. Le change fail rate de mon fork, avec 1 échec sur 6 déploiements, n'est évidemment pas représentatif.

    Six déploiements sont beaucoup trop peu pour tirer une conclusion, et surtout ils ont été créés artificiellement le même jour pour les besoins du TD.

    Pour obtenir quelque chose de plus représentatif, il faudrait avoir des déploiements répartis sur plusieurs semaines ou plusieurs mois, un volume beaucoup plus important, plusieurs vrais incidents et des délais de résolution qui correspondent à une situation réelle.

## Phase 5 - lecture critique des outils

22. Non, utiliser DevLake, Middleware ou Four Keys ne permettrait pas automatiquement de calculer le change fail rate d'Excalidraw.

    Ces outils auraient toujours besoin des données disponibles dans les sources, notamment les déploiements et les issues GitHub.

    Si le lien entre un incident et le déploiement responsable n'existe pas dans la donnée source, l'outil ne peut pas l'inventer. Il risque donc de produire lui aussi un n/a.

    C'est ce que j'ai retenu de cette phase : un outil plus professionnel ne règle pas forcément un problème de donnée qui existe en amont.

23. L'outillage arrive finalement en dernier pour une bonne raison.

    Dans mon cas, le Quick Check et la discussion sur les définitions auraient déjà permis de voir que les métriques de stabilité allaient poser problème.

    Il fallait d'abord définir précisément les métriques et vérifier que les données nécessaires existaient. Ensuite seulement, l'outil pouvait servir à automatiser la collecte.

    Sinon, on automatise surtout la production de n/a, ou pire, de chiffres trompeurs comme le ×67 de la phase 1.

    Le workflow deploy.yml que j'ai utilisé pendant le TD m'a finalement été plus utile qu'un dashboard sophistiqué pour produire la donnée qui manquait.

24. Four Keys était une référence dans beaucoup de tutoriels jusqu'en 2024, mais le projet a été archivé en janvier 2024.

    Ce que j'en retiens, c'est qu'avant d'utiliser un outil trouvé dans un tutoriel ou un article, il faut vérifier son état actuel : date du dernier commit, dernière release, activité des issues et surtout si le dépôt est encore actif ou archivé.

    Un outil peut rester présent dans des tutoriels pendant plusieurs années alors qu'il n'est plus réellement maintenu.

## Restitution collective

- L'écart entre les résultats peut s'expliquer par la ligne du contrat concernant le point de départ du changement. J'ai travaillé seul, donc je n'ai pas vraiment de binôme avec qui comparer mes résultats. Par contre, si un autre groupe obtient un lead time différent, cette ligne peut expliquer l'écart. Dans mon contrat, j'ai choisi la « committer date sur la branche par défaut, pas l'ouverture de la PR ». Si un autre groupe a pris l'ouverture de la PR, son lead time sera mécaniquement plus long, puisque ça inclut le temps de review avant le merge. Au final, deux contrats différents donnent deux séries de résultats différentes, donc les chiffres ne sont pas directement comparables (5.2).

- Pour le lead time, le P90 est plusieurs fois supérieur à la médiane. Avant de chercher une solution, je regarderais donc la distribution complète des lots pour voir lesquels restent bloqués. La question que je me poserais serait : « qu'est-ce qui différencie les lots qui prennent longtemps des autres ? ». Ça permettrait ensuite de vérifier si le problème vient de la review, de la CI, de la priorisation ou d'une attente de validation. Les métriques permettent surtout de savoir où regarder, elles ne donnent pas directement la solution.

- Pour la performance de l'équipe qui maintient Excalidraw, je ne pense pas qu'on puisse tirer une conclusion à partir de ces seules données. Les métriques DORA doivent être regardées ensemble, sur une fenêtre donnée et pour une application donnée. Ici, je peux dire que les lots regroupent en moyenne 5,3 commits et que la moitié des lots ont un lead time inférieur à 43,2 heures. Par contre, je ne peux pas transformer ces chiffres en jugement sur l'équipe elle-même. Les métriques de livraison ne mesurent pas à elles seules la performance globale d'une équipe.

## Métriques du fork - phase 4

Résultat du collecteur sur Ahmedsouissii/TD-DORA, environnement production, fenêtre de 90 jours :

| Métrique | Valeur |
|---|---|
| Deployment frequency | 0,067/jour (6 déploiements) |
| Lead time P50 | 0 h |
| Lead time P90 | 0 h |
| Commits / lots | 4 / 4 |
| Change fail rate | 16,7 % (1 échec sur 6) |
| Failed deployment recovery time | 1,2 h |
| Deployment rework rate | 16,7 % (1 hotfix sur 6) |

L'incident utilisé pour produire la donnée manquante est l'issue #3, « Regression apres deploiement en production ».

Elle a été ouverte à 12:27, fermée à 13:41, et rattachée au déploiement hotfix 6480687688 grâce au champ caused_by ajouté dans le corps de l'issue.

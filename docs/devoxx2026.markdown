---
layout: default
title: 'Devoxx France 2026 : ce que je retiens'
description: Un résumé de ce qui m'a marqué et ce que je retiens de cette édition
---

* Do not remove this line (it will not be displayed)
{:toc}

-----

# IA 🤖

Sans surprise, un grand nombre de présentations avaient pour sujet l'IA et plus particulièrement son aspect agentique, dont on ne cesse d'entendre parler ces derniers mois.

Ce qui m'a peut-être le plus marqué, c'est le fait que tout le monde semble l'avoir acceptée comme un outil incontournable, alors que je m'attendais peut-être à davantage de débat sur le sujet.
Les taux d'adoption sont toutefois très variables selon les équipes.
Entre les deux extrêmes que sont le "vibe codeur fou" (celui qui ne code plus, ne relit rien et "ship" en prod automatiquement) et le développeur totalement réfractaire à utiliser ces outils, la majorité se situe quelque part au milieu du spectre.

## Où en est-on ?

Les 4 keynotes auxquelles j'ai assisté portaient sur ce sujet avec des angles assez différents.

### Le code devient moins important

[Nicolas Grenié](https://nicolasgrenie.com/) (Developer Advocate), qui n'a plus écrit une seule ligne depuis un an, fait ce constat : les ingénieurs que nous sommes se transforment en "**ingénieurs avec un esprit produit**". Le code devient secondaire. La valeur d'un développeur est dans sa capacité à co-construire le produit avec les utilisateurs et/ou les équipes métier.

L'IA accélère notre capacité à prototyper. Cela permet de prendre des décisions produit pertinentes sans engager des développements coûteux. **Le cycle de développement devient "prompt -> prototypage -> décision"** là où, avant, c'était "spécifications -> ticket -> implémentation -> démonstration".

Nicolas fait aussi le constat, que je retrouve autour de moi, que l'IA aide les PO/PM à explorer davantage de choses de manière plus autonome, avant de passer la main aux équipes de développement pour l'industrialisation. En revanche, les aspects sécurité, conformité ou scalabilité nécessitent toujours une expertise humaine.

Enfin, il insiste : on doit absolument relire ce que fait l'IA !

### Penser produit

Marjory Cannone, qui accompagne les PME et collectivités dans l'intégration de ces technologies, fait le constat suivant : beaucoup d'entreprises arrivent avec une solution avant d'avoir un besoin. Elles veulent intégrer de l'IA sans réellement savoir pourquoi.

Elle rejoint le discours de la précédente keynote sur le fait qu'il faut repartir du produit : que souhaite-t-on construire ? Parfois, la réponse est simplement davantage d'automatisation, sans aller jusqu'à l'IA.

La technologie n'est plus limitante aujourd'hui : **nous devons d'abord répondre à des besoins produit et avoir une compréhension du business**.

Marjory remarque, elle aussi, que l'IA est excellente pour prototyper mais il faut industrialiser ensuite.

### L'impact sur nos sociétés

Jean-Gabriel Ganascia (ingénieur et philosophe) s'interroge avec nous sur les impacts de l'IA dans nos sociétés.

Il repart du [_Droit à la paresse_ de Paul Lafargue](https://fr.wikipedia.org/wiki/Le_Droit_%C3%A0_la_paresse) pour présenter l'idée selon laquelle l'IA, comme toute nouvelle technologie, devrait nous rendre plus heureux : c'est la promesse de moins de travail après tout, non ?

La réalité est bien différente : si nous n'avions réellement plus rien à faire grâce à l'IA, l'ennui et l'angoisse nous guetteraient. Comment occuper notre esprit ? De toute façon, nous libérer du temps est une utopie. Nul ne doute que nous aurons davantage de tâches à accomplir si l'IA nous permet d'aller plus vite.

Jean-Gabriel nous amène aussi à réfléchir sur les "règles de morale" données aux IA pour par exemple ne pas nous répondre lorsqu'on demande "comment fabriquer une bombe ?" : c'est une forme de censure, gouvernée par certains biais.

Il termine son intervention sur cette question rhétorique : **après les âges du bronze ou du fer, ne sommes-nous pas dans l'âge du contrôle et de la falsification ?**

### Et la géopolitique ?

Loup Cellard (chercheur au CNRS) nous donne des exemples d'enjeux très concrets liés aux infrastructures nécessaires à l'IA (data centers, câbles sous-marins, ...). Il évoque notamment les conflits d'usages autour de l'électricité et de l'immobilier.

Il pose aussi la question d'une souveraineté très floue sur ces sujets, majoritairement aux mains du secteur privé dont nous dépendons pour tous nos usages numériques et IA. Or la souveraineté est une notion du "monde réel", difficilement applicable dans le "monde numérique" où tout est interconnecté. Peut-être devrions-nous imaginer de nouveaux concepts, plus adaptés, pour nous protéger.

De manière plus anecdotique, j'ai appris que Marseille est dans le TOP 10 mondial des villes les plus connectées au reste du monde (via les câbles principaux sous marins et terrestres) alors que ce n'est pas une capitale.

## Context engineering 🧠

Benoît Fontaine (architecte) nous présente le nouveau "skill" (pun intended) à maîtriser : le context engineering. Oui, l'étape d'après le prompt engineering dont on parlait il y a quelques années :)

L'idée est que **le (bon) contexte est essentiel pour bénéficier pleinement de la puissance des agents IA**.

**Lorsque vous atteignez 60 % de la fenêtre de contexte de l'agent, la pertinence et la performance se dégradent fortement**. À ce moment-là, il faut "compacter" le contexte (certains outils comme Claude Code permettent d'indiquer les éléments les plus pertinents à conserver). Si vous dépassez ce seuil, la session est foutue. Vous aurez fortement intérêt à repartir d'une nouvelle session.

Parmi les indicateurs que votre contexte est saturé : l'agent hallucine des APIs, il oublie les conventions qu'il utilisait précédemment, il boucle sur un même problème, ...

Autre point intéressant : **le contexte donné en amont de la session** (via le prompt ou les fichiers `AGENTS.md`) **est beaucoup plus utile que le contexte découvert automatiquement** par l'exploration de l'agent.

Selon une étude récente, certaines entreprises voient 2x plus d'incidents avec l'usage de l'IA, d'autres 2x moins ! La conclusion proposée par Benoît : **l'outil n'est pas en cause, c'est la façon dont on s'en sert qui compte !**

## Nouvelles technos, nouvelles attaques... 🔐

Boussad Addad (expert en IA) et Hugo Bechu (ingénieur en cybersécurité) nous proposent différents vecteurs d'attaque possibles autour des modèles IA.

Ils ont notamment présenté des **attaques par évasion de modèle** sur des modèles qui analysent les images. Par exemple, **comment une image de char d'assaut peut être identifiée comme une ambulance** en perturbant de manière imperceptible à l'oeil nu l'image originale. Même chose avec un panneau STOP, qui peut être vu comme un panneau de limitation de vitesse. Les mêmes techniques peuvent aussi s'appliquer aux modèles de texte : un peu de bruit dans un prompt peut rendre inefficaces les protections censées empêcher d'obtenir des informations pour fabriquer une bombe.

On imagine malheureusement assez facilement les impacts réels de ces cyberattaques.

# Les value types arrivent en Java (JEP 401) ☕

Clément de Tastes (tech lead) et Rémi Forax (développeur, enseignant-chercheur et contributeur Java) nous font un résumé en 45 min de l'état actuel de l'implémentation des value types en Java : comment allons-nous les utiliser, qu'allons-nous gagner ?

Faisant partie du projet Valhalla démarré en 2014 (il y a 12 ans !), la promesse est simple : _"coder comme avec une classe, obtenir les performances d'un type primitif"_.

L'implémentation n'est pas définitive et pourrait complètement changer, mais les résultats sont à la hauteur des promesses. Côté usage, il suffit d'ajouter le mot-clé `value` devant `record` ou `class`, rien de plus. Sur des calculs de fractales, un benchmark donne un temps de calcul de :
- 19ms/op pour une implémentation à base de types primitifs
- 200ms/op pour une implémentation à base de classes (`record`)
- 20ms/op pour une implémentation à base de `value record` ; soit à peine plus que l'implémentation primitive

Ce que j'ai retenu et que je n'avais pas lu jusqu'à maintenant : l'implémentation se fait au runtime par le JIT et non à la compilation. Gros avantage : la rétrocompatibilité. Il n'y aura pas besoin de recompiler du code externe qui utiliserait des classes du JDK désormais marquées comme `value class` (exemple : `java.lang.Integer`, `java.util.Optional`, `java.time.LocalDate`, ...).

# Eloge de la simplicité ✨

Frédéric Leguédois (coach agile) nous livre un one-man-show assumé où il remet en cause nos pratiques actuelles, qui semblent loin de la simplicité promue initialement par le mouvement agile.

Parmi les différents sujets évoqués, il propose un parallèle entre le monde du logiciel et les urgences hospitalières en matière de priorisation (l'organisation des urgences étant un sujet d'études scientifiques). Les urgences s'apparentent à un système complexe avec plusieurs équipes qui doivent fonctionner ensemble. Pourtant, vous n'y verrez pas de backlog, pas de planning... il est impossible d'y prévoir le futur. C'est l'utilisateur (le patient) qui détermine la priorité des sujets, de manière très instinctive et naturelle.

**Il n'y a pas de corrélation entre la performance d'un système et sa prédictabilité : laissons nos utilisateurs nous guider sur les priorités** (de manière imprédictible) et délivrons ce qui est en haut de la liste (et pour lesquelles nous avons les moyens d'agir), sans perdre du temps à faire des plannings déjà obsolètes le lendemain de leur écriture. C'est, en essence, le message que Frédéric veut nous faire passer.

# Les communautés de pratique 🤝

Edwige Fiaclou et Pauline Cresson (manager) nous partagent leur expérience des communautés de pratique (d'après [le concept d'Etienne Wenger](https://fr.wikipedia.org/wiki/Communaut%C3%A9_de_pratique)) chez Swissquote (fintech).

Chez elles, **25 % des 1700 personnes dans l'engineering sont impliquées dans 23 communautés différentes**, un nombre qui croît régulièrement.

Leur recette pour des communautés réussies ?
1. l'alchimie : une communauté doit réunir des passionnés autour d'un sujet pouvant alimenter des discussions pendant de longues heures. Idéalement, elle doit avoir une double vision : **améliorer le quotidien et explorer de nouveaux territoires**.
2. un soupçon de cadre : chaque communauté doit avoir une gouvernance et des facteurs de succès bien définis. Le contenu discuté doit s'axer autour de 3 piliers : **apprendre, partager, évoluer**.
3. de l'**autonomie** : liberté d'initiative et de gouvernance
4. du partage : **ce qui n'est pas partagé est perdu**
5. des opportunités de s'investir de différentes manières : **facilitateur, contributeur actif ou auditeur libre**
6. de la **reconnaissance** : valoriser le temps investi, le rendre visible et le célébrer

Le plus grand défi ? Trouver le temps car tout cela doit se faire en dehors du temps des projets business. En moyenne c'est 2h/semaine qui y sont dédiées chez elles.


# C'est quoi un bon dashboard ? 📊

Kamila Giedrojc (lead product designer) veut nous sensibiliser aux éléments importants lors de la construction d'un dashboard. Objectif : éviter qu'il ne finisse aux oubliettes quelques jours après sa création.

Tout d'abord, tout n'a pas besoin d'être monitoré. **Quelle action l'utilisateur de ce dashboard va-t-il personnellement prendre si un indicateur monte ou descend ?** Si la réponse est "aucune", cet indicateur n'a pas sa place sur le dashboard.

Ensuite, d'autres éléments :
- qui est l'utilisateur du dashboard ? Quand et comment va-t-il le regarder (sur son smartphone plusieurs fois par jour, ou moins régulièrement sur un grand écran partagé) ?
- un chiffre seul n'a pas de sens, il doit y avoir du contexte (augmentation ou diminution, quelle est la cible ?)
- utiliser la couleur avec parcimonie pour mettre en valeur les indicateurs demandant une action immédiate, éviter le sapin de Noël !

Enfin, Kamila nous partage une très bonne référence pour choisir **la bonne visualisation en fonction du message que l'on souhaite faire passer : le [Visual Vocabulary du Financial Times](https://ft-interactive.github.io/visual-vocabulary/)**.

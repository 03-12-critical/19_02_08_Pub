---
layout: layouts/blog-post.njk
title: Gagner en sécurité et en confidentialité en partitionnant le cache
description: |2

  Le partitionnement du cache HTTP de Chrome améliore la sécurité et la confidentialité.
authors:
  - agektmr
date: '2020-10-06'
updated: '2020-10-24'
---

En général, la mise en cache peut améliorer les performances en stockant les données afin que les demandes futures pour les mêmes données soient traitées plus rapidement. Par exemple, une ressource mise en cache du réseau peut éviter un aller-retour vers le serveur. Un résultat de calcul mis en cache peut omettre le temps nécessaire pour effectuer le même calcul.

Dans Chrome, le mécanisme de cache est utilisé de différentes manières et HTTP Cache en est un exemple.

## Fonctionnement actuel du cache HTTP de Chrome

Depuis la version 85, Chrome met en cache les ressources extraites du réseau, en utilisant leurs URL de ressources respectives comme clé de cache. (Une clé de cache est utilisée pour identifier une ressource mise en cache.)

L'exemple suivant illustre comment une seule image est mise en cache et traitée dans trois contextes différents :

<figure class="float-left">
  {% Img src="image/T4FyVKpzu4WKF1kBNvXepbi08t52/zqkRCKG9jR3uBtcEwPgV.png", alt="Cache Key: https://x.example/doge.png", width="570", height="433" %}
  <figcaption>
    <b>Cache Key</b>: { <code>https://x.example/doge.png</code> }
  </figcaption>
</figure>

Un utilisateur visite une page ( `https://a.example` ) qui demande une image ( `https://x.example/doge.png` ). L'image est demandée au réseau et mise en cache en utilisant `https://x.example/doge.png` comme clé.

<figure class="float-left">
  {% Img src="image/T4FyVKpzu4WKF1kBNvXepbi08t52/sXZTOs9iABokE7VsOoXT.png", alt="Cache Key: https://x.example/doge.png", width="570", height="433" %}
  <figcaption>
    <b>Cache Key</b>: { <code>https://x.example/doge.png</code> }
  </figcaption>
</figure>

Le même utilisateur visite une autre page ( `https://b.example` ), qui demande la même image ( `https://x.example/doge.png` ).
 Le navigateur vérifie son cache HTTP pour voir s'il a déjà cette ressource en cache, en utilisant l'URL de l'image comme clé. Le navigateur trouve une correspondance dans son cache, il utilise donc la version mise en cache de la ressource.

<figure class="float-left">
  {% Img src="image/T4FyVKpzu4WKF1kBNvXepbi08t52/c8jEGuxXemlwMbezevOc.png", alt="Cache Key: https://x.example/doge.png", width="570", height="433" %}
  <figcaption>
    <b>Cache Key</b>: { <code>https://x.example/doge.png</code> }
  </figcaption>
</figure>

Peu importe si l'image est chargée à partir d'un iframe. Si l'utilisateur visite un autre site Web ( `https://c.example` ) avec une iframe ( `https://d.example` ) et que l'iframe demande la même image ( `https://x.example/doge.png` ), le navigateur peut toujours charger l'image à partir de son cache car la clé de cache est la même sur toutes les pages.

Ce mécanisme fonctionne bien du point de vue des performances depuis longtemps. Cependant, le temps qu'un site Web prend pour répondre aux requêtes HTTP peut révéler que le navigateur a accédé à la même ressource dans le passé, ce qui expose le navigateur à des attaques de sécurité et de confidentialité, comme les suivantes :

- **Détecter si un utilisateur a visité un site spécifique** : Un adversaire peut détecter l'historique de navigation d'un utilisateur en vérifiant si le cache contient une ressource qui pourrait être spécifique à un site particulier ou à une cohorte de sites.
- **[Attaque de recherche intersites](https://portswigger.net/daily-swig/new-xs-leak-techniques-reveal-fresh-ways-to-expose-user-information)** : un adversaire peut détecter si une chaîne arbitraire se trouve dans les résultats de recherche de l'utilisateur en vérifiant si une image "pas de résultats de recherche" utilisée par un site Web particulier se trouve dans le cache du navigateur.
- **Suivi intersite** : le cache peut être utilisé pour stocker des identifiants de type cookie en tant que mécanisme de suivi intersite.

Pour atténuer ces risques, Chrome partitionnera son cache HTTP à partir de Chrome 86.

## Comment le partitionnement du cache affectera-t-il le cache HTTP de Chrome ?

Avec le partitionnement du cache, les ressources mises en cache seront codées à l'aide d'une nouvelle "clé d'isolation réseau" en plus de l'URL de la ressource. La clé d'isolation réseau est composée du site de niveau supérieur et du site de trame en cours.

{% Aside %} Le "site" est reconnu à l'aide de " [scheme://eTLD+1](https://web.dev/same-site-same-origin/) " donc si les requêtes proviennent de pages différentes, mais qu'elles ont le même schéma et le domaine de premier niveau effectif+1, elles utiliseront la même partition de cache . Pour en savoir plus à ce sujet, lisez [Comprendre "même site" et "même origine"](https://web.dev/same-site-same-origin/) . {% endAside %}

Reprenez l'exemple précédent pour voir comment le partitionnement du cache fonctionne dans différents contextes :

<figure class="float-left">
  {% Img src="image/T4FyVKpzu4WKF1kBNvXepbi08t52/zqkRCKG9jR3uBtcEwPgV.png", alt="Cache Key { https://a.example, https://a.example, https://x.example/doge.png}", width="570", height="433" %}
  <figcaption>
    <b>Cache Key</b>: { <code>https://a.example</code>, <code>https://a.example</code>, <code>https://x.example/doge.png</code> }
  </figcaption>
</figure>

Un utilisateur visite une page ( `https://a.example` ) qui demande une image ( `https://x.example/doge.png` ). Dans ce cas, l'image est demandée au réseau et mise en cache à l'aide d'un tuple composé de `https://a.example` (le site de niveau supérieur), `https://a.example` (le site du cadre actuel) et `https://x.example/doge.png` (l'URL de la ressource) comme clé. (Notez que lorsque la demande de ressource provient du cadre de niveau supérieur, le site de niveau supérieur et le site du cadre actuel dans la clé d'isolation réseau sont identiques.)

<figure class="float-left">
  {% Img src="image/T4FyVKpzu4WKF1kBNvXepbi08t52/sXZTOs9iABokE7VsOoXT.png", alt="Cache Key { https://a.example, https://a.example, https://x.example/doge.png}", width="570", height="433" %}
  <figcaption>
    <b>Cache Key</b>: { <code>https://b.example</code>, <code>https://b.example</code>, <code>https://x.example/doge.png</code> }
  </figcaption>
</figure>

Le même utilisateur visite une page différente ( `https://b.example` ) qui demande la même image ( `https://x.example/doge.png` ). Bien que la même image ait été chargée dans l'exemple précédent, puisque la clé ne correspond pas, il ne s'agira pas d'un accès au cache.

L'image est demandée au réseau et mise en cache à l'aide d'un tuple composé de `https://b.example` , `https://b.example` et `https://x.example/doge.png` comme clé.

<figure class="float-left">
  {% Img src="image/T4FyVKpzu4WKF1kBNvXepbi08t52/kr9TtbDQPQfR86rNSrJX.png", alt="Cache Key { https://a.example, https://a.example, https://x.example/doge.png}", width="570", height="433" %}
  <figcaption>
    <b>Cache Key</b>: { <code>https://a.example</code>, <code>https://a.example</code>, <code>https://x.example/doge.png</code> }
  </figcaption>
</figure>

Maintenant, l'utilisateur revient à `https://a.example` mais cette fois l'image ( `https://x.example/doge.png` ) est intégrée dans une iframe. Dans ce cas, la clé est un tuple contenant `https://a.example` , `https://a.example` et `https://x.example/doge.png` et un accès au cache se produit. (Notez que lorsque le site de niveau supérieur et l'iframe sont le même site, la ressource mise en cache avec le cadre de niveau supérieur peut être utilisée.

<div class="clearfix"></div>

<figure class="float-left">
  {% Img src="image/T4FyVKpzu4WKF1kBNvXepbi08t52/BIJfNKd7YfXuXR3xdafb.png", alt="Cache Key { https://a.example, https://a.example, https://x.example/doge.png}", width="570", height="433" %}
  <figcaption>
    <b>Cache Key</b>: { <code>https://a.example</code>, <code>https://c.example</code>, <code>https://x.example/doge.png</code> }
  </figcaption>
</figure>

L'utilisateur est de retour sur `https://a.example` mais cette fois l'image est hébergée dans une iframe de `https://c.example` .

Dans ce cas, l'image est téléchargée depuis le réseau car aucune ressource dans le cache ne correspond à la clé composée de `https://a.example` , `https://c.example` et `https://x.example/doge.png` .

<figure class="float-left">
  {% Img src="image/T4FyVKpzu4WKF1kBNvXepbi08t52/Gg99hTwbcxc3DdUtgdnM.png", alt="Cache Key { https://a.example, https://a.example, https://x.example/doge.png}", width="570", height="433" %}
  <figcaption>
    <b>Cache Key</b>: { <code>https://a.example</code>, <code>https://c.example</code>, <code>https://x.example/doge.png</code> }
  </figcaption>
</figure>

Que faire si le domaine contient un sous-domaine ou un numéro de port ? L'utilisateur visite `https://subdomain.a.example` , qui intègre un iframe ( `https://c.example:8080` ), qui demande l'image.

Étant donné que la clé est créée sur la base de "scheme://eTLD+1", les sous-domaines et les numéros de port sont ignorés. Par conséquent, un accès au cache se produit.

<figure class="float-left">
  {% Img src="image/T4FyVKpzu4WKF1kBNvXepbi08t52/47mGHE7I9qlFpPER12CL.png", alt="Cache Key { https://a.example, https://a.example, https://x.example/doge.png}", width="570", height="433" %}
  <figcaption>
    <b>Cache Key</b>: { <code>https://a.example</code>, <code>https://c.example</code>, <code>https://x.example/doge.png</code> }
  </figcaption>
</figure>

Que se passe-t-il si l'iframe est imbriqué plusieurs fois ? L'utilisateur visite `https://a.example` , qui intègre un iframe ( `https://b.example` ), qui intègre encore un autre iframe ( `https://c.example` ), qui demande enfin l'image.

Étant donné que la clé est extraite du cadre supérieur ( `https://a.example` ) et du cadre immédiat qui charge la ressource ( `https://c.example` ), un accès au cache se produit.

## FAQ

### Est-il déjà activé sur mon Chrome ? Comment puis-je vérifier ?

La fonctionnalité sera déployée jusqu'à la fin de l'année 2020. Pour vérifier si votre instance Chrome la prend déjà en charge :

1. Ouvrez `chrome://net-export/` et appuyez sur **Démarrer la journalisation sur le disque** .
2. Spécifiez où enregistrer le fichier journal sur votre ordinateur.
3. Naviguez sur le Web sur Chrome pendant une minute.
4. Revenez à `chrome://net-export/` et appuyez sur **Stop Logging** .
5. Accédez à `https://netlog-viewer.appspot.com/#import` .
6. Appuyez sur **Choisir un fichier** et transmettez le fichier journal que vous avez enregistré.

Vous verrez la sortie du fichier journal.

Sur la même page, recherchez `SplitCacheByNetworkIsolationKey` . S'il est suivi de `Experiment_[****]` , le partitionnement du cache HTTP est activé sur votre Chrome. S'il est suivi de `Control_[****]` ou `Default_[****]` , il n'est pas activé.

### Comment puis-je tester le partitionnement du cache HTTP sur mon Chrome ?

Pour tester le partitionnement du cache HTTP sur votre Chrome, vous devez lancer Chrome avec un indicateur de ligne de commande : `--enable-features=SplitCacheByNetworkIsolationKey` . Suivez les instructions sur [Exécuter Chromium avec des indicateurs](https://www.chromium.org/developers/how-tos/run-chromium-with-flags) pour savoir comment lancer Chrome avec un indicateur de ligne de commande sur votre plate-forme.

### En tant que développeur Web, y a-t-il une action que je devrais prendre en réponse à ce changement ?

Il ne s'agit pas d'un changement radical, mais cela peut imposer des considérations de performances pour certains services Web.

Par exemple, ceux qui desservent de gros volumes de ressources pouvant être mises en cache sur de nombreux sites (comme les polices et les scripts populaires) peuvent voir leur trafic augmenter. De plus, ceux qui consomment ces services peuvent avoir une dépendance accrue à leur égard.

(Il existe une proposition pour activer les bibliothèques partagées d'une manière préservant la confidentialité appelée [Web Shared Libraries](https://docs.google.com/document/d/1lQykm9HgzkPlaKXwpQ9vNc3m2Eq2hF4TY-Vup5wg4qg/edit#) ( [vidéo de présentation](https://www.youtube.com/watch?v=cBY3ZcHifXw) ), mais elle est toujours à l'étude.)

### Quel est l'impact de ce changement de comportement ?

Le taux global d'échec du cache augmente d'environ 3,6 %, les modifications apportées au FCP (First Contentful Paint) sont modestes (~ 0,3 %) et la fraction globale d'octets chargés depuis le réseau augmente d'environ 4 %. Vous pouvez en savoir plus sur l'impact sur les performances dans [l'explication du partitionnement du cache HTTP](https://github.com/shivanigithub/http-cache-partitioning#impact-on-metrics) .

### Est-ce standardisé ? Les autres navigateurs se comportent-ils différemment ?

Les "partitions de cache HTTP" sont [standardisées dans la spécification de récupération](https://fetch.spec.whatwg.org/#http-cache-partitions) bien que les navigateurs se comportent différemment :

- **Chrome** : utilise le schéma de niveau supérieur://eTLD+1 et le schéma de cadre://eTLD+1
- **Safari** : Utilise [eTLD+1 de premier niveau](https://webkit.org/blog/8613/intelligent-tracking-prevention-2-1/)
- **Firefox** : [Prévoyez de mettre en œuvre](https://bugzilla.mozilla.org/show_bug.cgi?id=1536058) avec le schéma de niveau supérieur://eTLD+1 et envisagez d'inclure une deuxième clé comme Chrome

### Comment traite-t-on la récupération auprès des travailleurs ?

Les travailleurs dédiés utilisent la même clé que leur cadre actuel. Les agents de service et les agents partagés sont plus compliqués car ils peuvent être partagés entre plusieurs sites de niveau supérieur. La solution pour eux est actuellement en discussion.

## Ressources

- [Projet d'isolation de stockage](https://docs.google.com/document/d/1V8sFDCEYTXZmwKa_qWUfTVNAuBcPsu6FC0PhqMD6KKQ/edit#heading=h.oixrt0wpp8h5)
- [Explication - Partitionner le cache HTTP](https://github.com/shivanigithub/http-cache-partitioning)

*******
express
*******

Node vient souvent avec express (http://expressjs.com), un framework permettant d'en tirer le meilleur parti.

.. note:: on va copier notre site actuel pour le modifier avec express

gestion de package
==================

Node vient avec son célèbre installeur npm : https://www.npmjs.com 
Il existe des alternatives, comme  :code:`yarn` par exemple
 
:code:`npm` va installer les package pour nous : :code:`npm install express` 

Il a rajouté un répertoire :code:`node_module` plein d'autres répertoires à l'intérieur. Ce sont les dépendances du framework express. Tout installer à la main aurait été non faisable. 

Il manque cependant un fichier : :code:`package.json` : il gère les dépendances de notre projet pour que l'on ait pas à emmener avec nous le répertoire :code:`node_module`.

Donc commençons par reprendre de 0 : :code:`npm uninstall express` (:code:`node_module` est à nouveau vide).

Puis : 
    #. :code:`npm init` (on crée le fichier de configuration)
    #. :code:`npm install express --save` (ajout de la dépendance express dans :code:`package.json`)
    

On pourra maintenant supprimer le répertoire :code:`node_modules` et le recréer avec :code:`npm install` 


.. note:: 
    Il y a de tout sur :code:`npm`. Quasi tout le monde peut poster des package. Donc Le meilleur y cotoie le pire, comme https://github.com/kevva/is-negative . Je vous rassure, il y a aussi https://github.com/kevva/is-positive mais bizarrement pas https://github.com/kevva/is-zero
    
    

Route express
=============

on a juste remplacé les routes basiques par celles offertes par express dans :code:`app.js` 

On utilise les méthodes *get* de http. Les autres existent également bien sur : http://expressjs.com/fr/starter/basic-routing.html


app.js
^^^^^^ 

.. code-block:: javascript 

    const http = require('http') 

    var fs = require('fs')

    var express = require('express')
    var app = express()

    app.get('/', (request, response) => {
	    response.statusCode = 200;
	    response.setHeader('Content-Type', 'text/html');
        fs.createReadStream(__dirname + "/html/index.html", "utf8").pipe(response)    
    })

    app.get('/contact', (request, response) => {
    	response.statusCode = 200;
		response.setHeader('Content-Type', 'text/html');
		
		fs.createReadStream(__dirname + "/html/contact.html", "utf8").pipe(response)    
    })

    //manque 404

    app.listen(3000);
    console.log("c'est parti")


La lecture des fichiers est également simplifiée en utilisant la méthode :code:`sendFile` du paramètre :code:`response` :


.. code-block:: javascript 


    var http = require('http') 

    var fs = require('fs')

    var express = require('express')
    var app = express()

    app.get('/', (request, response) => {
            response.sendFile(__dirname + "/html/index.html")
    })

    app.get('/contact', (request, response) => {
        response.sendFile(__dirname + "/html/contact.html")
    })

    //manque 404

    app.listen(8080);
    console.log("c'est parti")


middleware et 404
=================

Le middleware se trouve entre la réception de la requête par node et le rendu donné par :code:`app.METHOD`. Plus d'informations ici : http://expressjs.com/fr/guide/using-middleware.html 

Les appels aux middleware se font dans l'ordre. Le paramètre next permettant d'aller à l'élément suivant de la route.


.. code-block:: javascript 

    var express = require('express')
    var app = express()

    app.use(function (req, res, next) {
        console.log('Time:', Date.now());
        next(); // sans cette ligne on ne pourra pas poursuivre.
    })

    app.use(function (req, res, next) {
        console.log("ensuite");
        next(); // sans cette ligne on ne pourra pas poursuivre.
    })


    app.get('/', (request, response) => {
            response.sendFile(__dirname + "/html/index.html")
    })

    app.get('/contact', (request, response) => {
        response.sendFile(__dirname + "/html/contact.html")
    })

    app.use(function (req, res, next) {
        console.log('la fin');
    })

    app.listen(8080);
    console.log("c'est parti")


Pour toute requête, on affiche la date, puis ensuite. Si la requête est un get que l'on réceptionne, on effectue la méthode puis on s'arrête puisqu'il n'y a pas de :code:`next()`. On écrit donc "la fin" que si aucune requête get n'est interceptée : c'est notre 404 !

On peut donc finalement écrire : 

.. code-block:: javascript

    var http = require('http') 

    var express = require('express')
    var app = express()

    app.get('/', (request, response) => {
            response.sendFile(__dirname + "/html/index.html")
    })

    app.get('/contact', (request, response) => {
        response.sendFile(__dirname + "/html/contact.html")
    })


    // 404 aucune interception
    app.use(function (req, res, next) {
          res.status(404).sendFile(__dirname + "/html/404.html")
    })
 
    app.listen(8080);
    console.log("c'est parti")


fichiers statiques
==================

Remplaçons le lien vers l'image de contact en un lien local. On va placer tous ces fichiers dans un répertoire :code:`assets`, puis puisque c'est une image dans le répertoire :code:`img`.

Et ça ne marche pas... On a un 404. C'est parce que notre serveur ne répond qu'à nos requêtes, pas aux fichiers réels. Il faut trouver un moyen que notre serveur puisse à la fois servir nos requêtes et les fichiers css, images, javascript front et autres inclus dans les fichiers html.

En développement, on pourra utiliser un middleware qui servira en tant que fichier toutes les demandes commançant par :code:`/static/`, mais c'est une mauvaise idée en production où l'on perd inutilement de la performance (voir https://blog.xervo.io/supercharge-your-nodejs-applications-with-nginx ou encore https://thefullsnack.com/don-t-serve-static-files-with-nodejs-31666462f79c#.sdjaerga1 ). 


On utilisera ainsi un autre serveur, :code:`nginx`, dont la spécialité est de servir les fichiers statiques, les autres routes étant dirigées vers express et node. Vous verrez ça plus tard lorsque l'on mettra le site en production. Une configuration production possible est décrite ici : http://blog.danyll.com/setting-up-express-with-nginx-and-pm2/ 

Pour l'instant, utilisons un petit middleware : 


.. code-block:: javascript

    var http = require('http') 

    var express = require('express')
    var app = express()


    app.use("/static", express.static(__dirname + '/static'))

    app.get('/', (request, response) => {
            response.sendFile(__dirname + "/html/index.html")
    })

    app.get('/contact', (request, response) => {
        response.sendFile(__dirname + "/html/contact.html")
    })


    // 404 aucune interception
    app.use(function (req, res, next) {
          res.status(404).sendFile(__dirname + "/html/404.html")
    })

    app.listen(8080);
    console.log("c'est parti");


templates
=========

Générer des fichiers html spécifiques pour chaque requête. Pour cela on du choix : http://expressjs.com/en/guide/using-template-engines.html et on utilisera http://ejs.co :

Il faut commencer par l'installer et le mettre en dépendance : :code:`npm install ejs --save` 
.. code-block:: javascript

    app.set('view engine', 'ejs')


Commençons par transformer nos fichiers html en templates :
    * les templates se trouvent par défaut dans le répertoire :code:`views`
    * on renomme nos fichiers html en ejs
    * on utilise la méthode de rendu plutôt que de charger directement les fichiers : https://www.npmjs.com/package/ejs

.. code-block:: javascript

    var http = require('http') 

    var express = require('express')
    var app = express()
	
	app.set('view engine', 'ejs')

    app.use("/static", express.static(__dirname + '/static'))

	app.get('/', (request, response) => {
	        response.render("index")
	})

	app.get('/contact', (request, response) => {
	    response.render("contact")
	})

    app.use(function (req, res, next) {
        res.status(404).render("404")
    })

    app.listen(3000);
    console.log("c'est parti")



Puis ajoutons un élément qui va être sur toutes les pages :
    * on crée une navbar toute simple, que l'on place dans un sous-répertoire de :code:`views`,  :code:`partials`
    * on l'inclut dans nos templates en ajoutant dans notre fichier ejs la ligne :code:`<% include partials/navbar.ejs %>` Ici, cela pourra être la première ligne du body. 


navbar.ejs
^^^^^^^^^^ 

.. code-block:: html

	<style>
	    nav > ul {
	        font-size: .5em;
	        text-align: left;
	    }
		nav > ul > li {
			display: inline;
		
		}
	</style>

	<nav>
	  <ul>
	  	<li><a href="/">Maison</a></li>
	    <li><a href="/contact">contact</a></li>
	  </ul>
	</nav>

Passage de paramètres
=====================
 
 .. todo:: cours prochain.
 

 
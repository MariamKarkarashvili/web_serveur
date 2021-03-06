*******************************
Outils de développement continu
*******************************

Logueur et tests unitaire/fonctionels

loggeur
=======

On commence à avoir pas mal de trafic sur notre site, et on est pas à l'abri de bug. La première chose à faire est donc de stocker tout ce qui se passe niveau serveur. En cas d'erreurs, on pourra remonter la trace de ce qui a conduit à l'erreur ou pire au plantage.

De nombreuses applications de log existent pour node, nous utiliserons `<https://github.com/winstonjs/winston>`_.

.. note:: voir par exemple `<https://blog.risingstack.com/node-js-logging-tutorial/>`_ ou encore `<http://thisdavej.com/using-winston-a-versatile-logging-library-for-node-js/>`_

Pour la version 3, celle que nous utiliserons, in faut installer la version de dév de winston : :command:`npm install winston@next --save`. 


.. note ::  Le code ci-après ne fonctionnera pas avec la version 2.4, qui est  la version stable à l'heure où j'écris ces ligne.


app.js
^^^^^^ 


En lisant un peu la doc et les tutos, on ajoute son propre logger dans :file:`app.ejs` :

.. code-block:: javascript

  const winston = require('winston');
  const logger = winston.createLogger({
      format: winston.format.combine(winston.format.timestamp(),
      winston.format.simple())
  });

Puis on ajoute la visibilité des logs. Ci après on ajoute un retour console qui va tout afficher jusqu'à un level silly : 

.. code-block:: javascript

  logger.add(new winston.transports.Console({
      level: 'silly'    
  }));  


Par défaut, le level d'affichage est de :code:`info` (voir `<https://github.com/winstonjs/winston#logging-levels>`_). 

On ajoute les deux dernières lignes à :file:`app.js` :

.. code-block:: javascript

  logger.info("c'est parti")
  logger.silly("mon kiki")

L'affichage console est pratique en développement. En production, on préfère peut de log et dans des fichiers. Par exemple : 

.. code-block:: javascript

  logger.add(new winston.transports.File({ filename: 'error.log', level: 'error' }))
  logger.add(new winston.transports.File({ filename: 'combined.log' }))
 

On va ajouter un log systématique à chaque appel grace à un petit middle-ware : 

.. code-block:: javascript

  app.use(function(req, res, next) {
      logger.debug("url: " + req.originalUrl);
      next();
  })

Et on log également les 404 :

.. code-block:: javascript

  // 404 aucune interception
  app.use(function (req, res, next) {
      logger.info("404 for: " + req.originalUrl);

      res.status(404).render("404")
  })


En changeant le level de notre logger à *debug* ont devrait voir tous les appels.

.. note :: les différents attributs de req sont décrites ici : `<http://expressjs.com/fr/api.html#req>`_


logger.js
^^^^^^^^^

De part la nature des import js, on peut passer des paramètres à la création d'un module. Utilisons ça pour séparer logger et app.

fichier :file:`logger.js` :

.. code-block:: js

    const winston = require('winston');
    const logger = winston.createLogger({
        format: winston.format.combine(winston.format.timestamp(),
        winston.format.simple())
    });

    // dev mode logger
    logger.add(new winston.transports.Console({
        level: 'silly'    
    }));  

    //file
    //logger.add(new winston.transports.File({ filename: 'error.log', level: 'error' }))
    //logger.add(new winston.transports.File({ filename: 'combined.log' }))

    module.exports = logger


fichier :file:`app.js` :

.. code-block:: js 

    var express = require('express')
    var app = express()

        module.exports = (logger) => {

            app.set('view engine', 'ejs')

            app.use(function(req, res, next) {
                logger.debug("url: " + req.originalUrl);
                next();
            })

            app.use("/static", express.static(__dirname + '/assets'))


            app.get('/', (request, response) => {
                response.render("home")
            })

            app.get('/commentaires', (request, response) => {
                response.render("commentaires")
            })


            // 404 aucune interception
            app.use(function (req, res, next) {
                res.status(404).render("404")
                logger.info("404 for: " + req.originalUrl)
            })

            return app
        }


fichier : file:`server.js` :

.. code-block:: js 

    logger = require('./logger.js')
    app = require('./app.js')(logger)

    port = 8080
    app.listen(port);

    logger.info("c'est parti: http://localhost:" + port.toString())
    logger.silly("mon kiki")


tests
=====

.. note :: 

    `<https://www.slideshare.net/robertgreiner/test-driven-development-at-10000-feet>`_
    regardez en particulier la courbe décroissante.

côté client
^^^^^^^^^^^  

On peut tester le rendu client en simulant un navigateur.

Pour cela on utilise selenium `<http://www.seleniumhq.org>`_ et ses webdriver qui simulent un browser. Tout ceci fonctionne en java, donc assurez vous d'avoir un java qui va.

#. installation de java (si nécessaire. Tapez java dans un terminal/powershell et soi ça rate, c'est qu'il faut l'installer) : `<https://www.java.com/fr/download/faq/develop.xml>`_ et suivez le lien pour télécharger le jdk.
#. le jar de selenium standalone server : `<http://www.seleniumhq.org/download/>`_
#. un driver. Nous utiliserons celui de chrome : `<https://sites.google.com/a/chromium.org/chromedriver/>`_.Il y en a d'autres possibles (par exemple pour firefox : `<https://github.com/mozilla/geckodriver/releases>`_)

Une fois selenium et le driver pacé dans un dossier selenium. Je l'ai placé dans le dossier parent de l'application. On peut tester pour voir si ça marche. En utilisant ce que j'ai téléchargé et mis dans le même dossier : :command:`java -Dwebdriver.chrome.driver=./chromedriver -jar selenium-server-standalone-3.8.1.jar` 

Un serveur web selenium est lancé. Il est sur le port 4444 par défaut (lisez les logs).

.. note :: java est toujours verbeux dans ses log. Apprenez à les lire. 

Et maintenant, il nous reste à installer `<http://webdriver.io>`_ pour utiliser selenium avec node : :command:`npm install --save-dev webdriverio`

.. note :: on a installé webdriver.io uniquement pour le developpement. Il n'est pas nécessaire de l'emmener avec nous en production.

Et on fait un premier essai avec le tout : :file:`selenium.essai.js` :

.. code-block:: js 
 
    var webdriverio = require('webdriverio');

    var options = {
        desiredCapabilities: {
            browserName: 'chrome'
        }
    }

    webdriverio
    .remote(options)
    .init()
    .url('https://www.google.fr')
    .saveScreenshot("snapshot.png")
    .catch(function(err) {
        console.log(err);}) 
    .end();




Avant d'executer le fichier avec :command:`node selenium.essai.js` On s'assure que le serveur selenium tourne toujours sur le port 4444.

.. note :: assurez vous de ne part avoir de serveur qui tourne sur le port par défaut. Sinon, changez de port par défaut.

On peut maintenant faire des vrai tests pour notre application : 

* vérifier que par défaut on est sur la page d'accueil,
* en cliquant sur commantaires on arrive bien sur la page de commantaires
* en recliquant sur le nom de la page, on retourne à l'accueil.

.. code-block:: js 

    var webdriverio = require('webdriverio');


    var options = {
        desiredCapabilities: {
            browserName: 'chrome'
        }
    }

    browser = webdriverio
    .remote(options)
    .init()


    browser.url('http://localhost:8080')
    .getTitle().then( (title) => {
        console.log("titre : " + title)
    })
    .click("a[href='commentaires']")
    .getTitle().then( (title) => {
        console.log("titre : " + title)
    })
    .click("a*=Da")
    .getTitle().then( (title) => {
        console.log("titre : " + title)
    })
    .catch(function(err) {
        console.log(err);
        }) 
    .end()


.. note :: attention au .end(). Tout est asynchrone donc si ajoute une ligne avec le .end() il risque d'être exécuté avant la fin de la requête.

On peut attraper plein de choses avec selenium et webdriver.io en utilisant les selecteurs : `<http://webdriver.io/guide/usage/selectors.html>`_


On peut finalement rajouter tout nos tests à la batterie de tests de node en faisant finir notre fichier par le nom "test.js". Voir partie suivante pour dréer une batterie de tests avec jest.js.


côté serveur
^^^^^^^^^^^^ 

`<https://facebook.github.io/jest/>`_

Tester le js et les routes.


Pour l'instant on a pas de fonction js, donc on va faire comme si et reprendre le tuto.

:file:`sum.js` :

.. code-block:: js

  function sum(a, b) {
      return a + b;
    }
    module.exports = sum;


:file:`sum.test.js` :

.. code-block:: js

  const sum = require('./sum');

  describe('test sum', () => {
      test('adds 0 + 0 to equal 0', () => {
          return expect(sum(0, 0)).toBe(0)
        });    
      test('adds 1 + 2 to equal 3', () => {
          return expect(sum(1, 2)).toBe(3)
        });
  })


On installe jest pour le developpement, :command:`npm install --save-dev jest`, puis on mets jest comme commande de test dans :file:`package.json`. Par exmeple le mien ressemble à : 

.. code-block:: json

  {
    "name": "donnees",
    "version": "1.0.0",
    "description": "cours sur les données",
    "main": "server.js",
    "scripts": {
      "test": "jest"
    },
    "author": "",
    "license": "ISC",
    "dependencies": {
      "ejs": "^2.5.7",
      "express": "^4.16.2",
      "winston": "^3.0.0-rc1"
    },
    "devDependencies": {
      "jest": "^21.2.1"
    }
  }


On peut ensuite utiliser la commande :command:`npm test`  pour exécuter tous les fichiers qui finissent par `test.js` 


.. note :: on pet aussi utiliser jest en ligne de commande en l'installant de façon globale. Voir `<https://facebook.github.io/jest/docs/en/getting-started.html#running-from-command-line>`_


Test des routes. On utilise en plus supertest `<https://github.com/visionmedia/supertest>`_ 

:command:`npm install --save-dev supertest`

.. note :: voir `<http://www.albertgao.xyz/2017/05/24/how-to-test-expressjs-with-jest-and-supertest/>`_

:file:`routes.test.js` :

.. code-block:: js

  const request = require('supertest');
  logger = require('./logger.js')
  app = require('./app.js')(logger)

  describe('routes ok', () => {
      test('It should response the GET method', (done) => {
          request(app).get("/").then((response) => {
              expect(response.statusCode).toBe(200)            
              done()
          })
      });
      test('It should response the GET method', (done) => {
          request(app).get("/commentaires").then((response) => {
              expect(response.statusCode).toBe(200)  
              done()          
          })
      });
  })

  describe('routes 404', () => {
      test('It should response 404', (done) => {
          request(app).get("/troululu").then((response) => {
              expect(response.statusCode).toBe(404)            
              done()
          })
      });
  })




test utilisateur (UI)
^^^^^^^^^^^^^^^^^^^^^ 

`<https://www.invisionapp.com/blog/ux-usability-research-testing/>`_

`<https://blogs.adobe.com/creativecloud/best-practices-for-usability-testing-in-ux-design/>`_


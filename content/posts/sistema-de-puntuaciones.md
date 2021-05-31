+++
title = "Sistema de puntuaciones"
date = "2021-05-31T19:04:17+02:00"
author = "Marcos Gómez"
authorTwitter = "usualmarcos" #do not include @
cover = ""
tags = ["js", "html", "router", "routes", "sequelize", "models", "controllers", "views"]
keywords = ["", ""]
description = "En este post se detalla como añadir la posibilidad de almacenar puntuaciones por cada grupo, y crear una página de puntuación para cada grupo"
showFullContent = false
+++

Cuando nos pidan implementar una funcionalidad nueva que se tenga que ver en una nueva "página" tendremos que tocar las rutas (`routes/index.js`), las vistas (`views/*`) e implementar una nueva funcionalidad en el controlador que toque (`controllers/*`) y en este caso los modelos (`models/*`).

A mí personalmente me gusta empezar por lo fácil, implementar la ruta o rutas.

## Implementación de la ruta

Para implementar esta funcionalidad vamos a hacer que el usuario pueda visualizar la puntuación de los grupos en una nueva página. Para ello implementaremos una primitiva HTTP `GET /scores/` en la que se visualizará una tabla con todas las puntuaciones de todos los grupos. Para gestionar el tema de las puntuaciones crearemos un controlador que se encarge de gestionar todas las primitivas relacionadas con las puntuaciones.

De momento supondremos que este controller lo llamaremos `scoreController`. Vamos a editar el documento de rutas para incluir la nueva ruta.

{{< code language="js" title="routes/index.js" expand="Show" collapse="Hide" isCollapsed="false" >}}
var express = require("express");
var router = express.Router();

const quizController = require("../controllers/quiz");
const userController = require("../controllers/user");
const sessionController = require("../controllers/session");
const groupController = require("../controllers/group");
// añadimos nuestro controlador que luego crearemos
const scoreController = require("../controllers/score");

[...]

// añadimos una nueva ruta al final del archivo para satisfacer nuestro objetivo
router.get("/scores", scoreController.index);
{{</code>}}

De esta manera, cuando el usuario haga un GET a esta ruta se pasará el control al middleware `index` de `scoreController` que ahora crearemos.

# Modificación del modelo

Para acomodar las puntuaciones por grupo tenemos que modificar el modelo de `Group` añadiendo un campo de puntuación, eso lo haremos en el archivo del modelo correspondiente:

{{< code language="js" title="routes/index.js" expand="Show" collapse="Hide" isCollapsed="false" >}}
const { Model } = require("sequelize");

// Definition of the Quiz model:

module.exports = (sequelize, DataTypes) => {
  class Group extends Model {}

  Group.init(
    {
      name: {
        type: DataTypes.STRING,
        unique: true,
        validate: { notEmpty: { msg: "Group name must not be empty" } },
      },
      // añadimos el nuevo campo
      score: {
        type: DataTypes.INTEGER,
        defaultValue: 0
      }
    },
    {
      sequelize,
    }
  );

  return Group;
};
{{</code>}}

Ahora podemos almacenar en la BBDD la puntuación de cada grupo. Para ello modificaremos `groupPlay` para ello.

## Creación del controllador

Crearemos un nuevo archivo `controllers/score.js` para este fin e implementaremos el método index.

{{< code language="js" title="controllers/score.js" expand="Show" collapse="Hide" isCollapsed="false" >}}
const Sequelize = require("sequelize");
const Op = Sequelize.Op;
const { models } = require("../models")

[...]

// GET /index
exports.index = async (req, res, next) => {
  try {
    const groups = await models.Group.findAll();
    res.render("scores/index.ejs", {
      groups,
    });
  } catch (error) {
    next(error);
  }
};
{{</code>}}

Se puede ver cómo el código es casi igual al index de groups, y esto es pq al final estamos obteniendo en un array los grupos que hay y sus atributos, entre ellos nombre y puntuación. La diferencia es en el `res.render(...)` que ahora llamamos a la nueva vista que crearemos.

## Implementación de la vista

Crearemos una nueva vista para ello: `views/scores/index.ejs` basándonos en el intex de la vista de groups ya hecha.

{{< code language="html" title="views/scores/index.ejs" expand="Show" collapse="Hide" isCollapsed="false" >}}
<h1>Scores:</h1>

<table>
  <% for (var i in groups) { %> <% var group = groups[i]; %>
  <tr>
    <td>
      <a href="/groups/<%= group.id %>/randomplay"><%= group.name %></a>
    </td>
    <td>
      <p><%= group.score %></p>
    </td>
  </tr>
  <% } %>
</table>
{{</code>}}

## Modificación de `groupCheck`

Para ir sumando las puntuaciones, modificaremos el controlador de groupCheck. En mi caso lo he llamado randomCheck pero es distinto al de los quizzes individuales porque está en el archivo `controllers/group.js`.

{{< code language="js" title="controllers/group.js" expand="Show" collapse="Hide" isCollapsed="false" >}}
[...]
exports.randomCheck = async (req, res, next) => {
  const curGroup = req.load.group;
  try {
    req.session.groupsRandomPlay = req.session.groupsRandomPlay || {};
    req.session.groupsRandomPlay[curGroup.id] = req.session.groupsRandomPlay[
      curGroup.id
    ] || {
      resolved: [],
      lastQuizId: 0,
    };

    const answer = req.query.answer || "";
    const result =
      answer.toLowerCase().trim() === req.load.quiz.answer.toLowerCase().trim();

    if (result) {
      req.session.groupsRandomPlay[curGroup.id].lastQuizId = 0;
      if (
        req.session.groupsRandomPlay[curGroup.id].resolved.indexOf(
          req.load.quiz.id
        ) === -1
      ) {
        req.session.groupsRandomPlay[curGroup.id].resolved.push(
          req.load.quiz.id
        );
      }

      const score = req.session.groupsRandomPlay[curGroup.id].resolved.length;

      // actualizamos el score del group en cuestión
      curGroup = await curGroup.save({fields: ["score"]});

      res.render("groups/random_result", {
        group: curGroup,
        result,
        answer,
        score,
      });
    } else {
      const score = req.session.groupsRandomPlay[curGroup.id].resolved.length;
      delete req.session.groupsRandomPlay[curGroup.id];
      res.render("groups/random_result", {
        group: curGroup,
        result,
        answer,
        score,
      });
    }
  } catch (error) {
    next(error);
  }
};
[...]
{{</code>}}

## Adición del enlace

Ahora sólo nos falta añadir el enlace en el menú de navegación.

{{< code language="html" title="views/scores/index.ejs" expand="Show" collapse="Hide" isCollapsed="false" >}}
<nav class="main" id="mainNav" role="navigation">
  <a href="/">Home</a>
  <a href="/quizzes">Quizzes</a>
  <a href="/author">Author</a>
  <a href="/groups">Groups</a>
  <a href="/scores">Groups Scores</a>
  <% if (locals.loginUser) { %>
      <a href="/users">Users</a>
  <% } %>
</nav>
{{</code>}}

Y ya estaría!!!! :)

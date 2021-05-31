+++
title = "Control de acceso"
date = "2021-05-31T20:04:17+02:00"
author = "Marcos Gómez"
authorTwitter = "usualmarcos" #do not include @
cover = ""
tags = ["js", "html", "router", "routes", "sequelize", "models", "controllers", "views"]
description = "En este post se detalla como hacer que ciertas páginas solo se puedan ver un determinado número de veces o durante unas horas del día"
showFullContent = false
+++

En el examen nos pueden pedir restringir el acceso a ciertas páginas, pudiendo éstas ser visualizadas un número determinado de veces o sólo por algunos usuarios durante ciertas horas del día.

Iremos paso a paso viendo que modificaciones se tienen que hacer en cada caso. Todos estos controles se implementan como middlewares en el controlador `controllers/session.js`

## Acceso a cada página permitido sólo un número X de veces

Este problema lo podemos resolver de varias maneras dependiendo de lo que queramos conseguir. Si queremos que el número de accesos realizados se resetee al volver a entrar a la página, haremos el control de acceso a través de `session`. Si queremos que el cambio sea persistente y para todos los usuarios, modificaremos los modelos.

### Control no persistente (mediante `session`)

Para hacer esto posible crearemos un nuevo middleware que va a ir generando y almacenando el número de veces que entramos en una página y el límite que establezcamos lo fijaremos en una variable que podemos tocar. Haremos dos versiones de esto: que el contador sea dependiente de cada página, o que el contador se aplique y tenga en cuenta para todas las páginas.

#### Un contador para cada página

Crearemos un controlador en `controllers/session.js` que se encargue de verificar el acceso y contabilizar los accesos nuevos.

{{< code language="js" title="controllers/session.js" expand="Show" collapse="Hide" isCollapsed="false" >}}
exports.visitLimitPerPage = (req, res, next) => {
    // establecemos el límite de visitas para cada página
    const VISIT_LIMIT = 10;

    // almacenamos en una variable la ruta a la que accede el usuario
    const url = req.url;

    // creamos el objeto req.session.visits donde almacenaremos un objeto con las visitas a cada página o cogemos el objeto ya presente
    req.session.visits = req.session.visits || {};

    // almacenamos un nuevo valor de visita para cada ruta o creamos la primera visita a esa página
    req.session.visits[url] = req.session.visits[url] || 1;

    // comprobamos que no se excede el número de visitas permitidas para esa página, en ese caso continuamos al siguiente mw, si no, mostramos mensaje y volvemos al home
    if (req.session.visits[url] <= VISIT_LIMIT) {
        next();
    } else {
        req.flash("error", "You have depleted the amount of visits permitted to " + url);
        res.redirect("/");
    }
}
{{</code>}}

Ahora sólo tenemos que incluir este middleware **antes** del resto de middlewares para las rutas en las que las queramos aplicar, por ejemplo:

{{< code language="js" title="routes/index.js" expand="Show" collapse="Hide" isCollapsed="false" >}}
router.get("/groups", sessionController.visitLimitPerPage, groupController.index);
{{</code>}}

#### Un contador para todas las páginas

Modificando un poco el middleware anterior se puede conseguir lo mismo pero con un contador común en `session`:

{{< code language="js" title="controllers/session.js" expand="Show" collapse="Hide" isCollapsed="false" >}}
exports.visitLimitGeneral = (req, res, next) => {
    // establecemos el límite de visitas general
    const VISIT_LIMIT = 10;

    // creamos el objeto req.session.visits donde almacenaremos un objeto con las visitas totales o cogemos el objeto ya presente
    req.session.visits = req.session.visits || 1;

    // comprobamos que no se excede el número de visitas permitidas para esa página, en ese caso continuamos al siguiente mw, si no, mostramos mensaje y volvemos al home
    if (req.session.visits <= VISIT_LIMIT) {
        next();
    } else {
        req.flash("error", "You have depleted the amount of visits permitted to " + url);
        res.redirect("/");
    }
}
{{</code>}}

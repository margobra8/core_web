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

Este problema lo podemos resolver de varias maneras dependiendo de lo que queramos conseguir. Si queremos que el número de accesos realizados se resetee al volver a entrar a la página, haremos el control de acceso a través de `session`. Si queremos que el cambio sea persistente y para todos los usuarios, modificaremos los modelos o usaremos una variable global.

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
    req.session.visits[url] = req.session.visits[url] || 0;
    req.session.visits[url]++;

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
    req.session.visits = req.session.visits || 0;
    req.session.visits++;

    // comprobamos que no se excede el número de visitas global permitido, en ese caso continuamos al siguiente mw, si no, mostramos mensaje y volvemos al home
    if (req.session.visits <= VISIT_LIMIT) {
        next();
    } else {
        req.flash("error", "You have depleted the amount of visits permitted to " + url);
        res.redirect("/");
    }
}
{{</code>}}

Añadiríamos este middleware a todas las rutas que lo necesiten como en el apartado anterior.

### Contador persistente

Para que la cuenta se mantenga durante sesiones es necesario que se almacene de manera persistente. La manera idónea sería crear un modelo `Page` que tuviese los campos de `url` y `visits`, creáramos todas las páginas actuales con un seeder y las migraciones correspondientes. Sin embargo, esto es muy tedioso y largo. Optaremos por otro método similar al anterior pero empleando una variable global nutriéndonos del objeto `global` que proporciona node.

En este caso sólo implementaré la situación de cuentas distintas por página porque es el caso más particular, se deja la implementación del caso del contador general persistente para el lector.

{{< code language="js" title="controllers/session.js" expand="Show" collapse="Hide" isCollapsed="false" >}}
exports.visitLimitPerPageGlobal = (req, res, next) => {
    // establecemos el límite de visitas para cada página
    const VISIT_LIMIT = 10;

    // almacenamos en una variable la ruta a la que accede el usuario
    const url = req.url;

    // creamos el objeto global.visits donde almacenaremos un objeto con las visitas a cada página o cogemos el objeto ya presente
    global.visits = global.visits || {};

    // almacenamos un nuevo valor de visita para cada ruta o creamos la primera visita a esa página
    global.visits[url] = global.visits[url] || 0;
    global.visits[url]++;

    // comprobamos que no se excede el número de visitas permitidas para esa página, en ese caso continuamos al siguiente mw, si no, mostramos mensaje y volvemos al home
    if (global.visits[url] <= VISIT_LIMIT) {
        next();
    } else {
        req.flash("error", "You have depleted the amount of visits permitted to " + url);
        res.redirect("/");
    }
}
{{</code>}}

Como siempre añadiríamos este middleware a las rutas que lo precisaran.

## Acceso a las páginas permitido a ciertos usuarios durante ciertas horas

Ahora trataremos el caso en el que queremos que los accesos estén restringidos a ciertos usuarios durante ciertas horas del día. Aquí expongo 3 middlewares que realizan funcionalidades distintas según lo que queremos conseguir:

- acceso restringido a un rango de horas para cualquier usuario
- acceso restringido un rango de horas para admins y el resto de tiempo para todos los usuarios
- acceso restringido sólo a admins durante una franja horaria determinada y cerrado para todos el resto del tiempo (con cascada de middlewares).

### Acceso restringido durante un rango de horas/fecha para cualquier usuario

Vamos a suponer que queremos que cualquier usuario sólo pueda entrar entre las 17:00 h y las 18:59 h a las páginas que decidamos. Para eso necesitamos crear un nuevo middleware en `session`.

{{< code language="js" title="controllers/session.js" expand="Show" collapse="Hide" isCollapsed="false" >}}
exports.hourRestrictionAnyUser = (req, res, next) => {
    // establecemos la hora inicio y final de la franja donde admitiremos visitas
    const LOW_HOUR = 17;
    const HIGH_HOUR = 19;

    // obtenemos la hora actual en el momento que se hace la request y sacamos la hora (sólo número de hora en formato 24h)
    const now = new Date().getHours();

    // comprobamos si estamos en esa franja, si es así continuamos al next mw, si no mostramos error y vamos a home.
    if (now >= LOW_HOUR && now < HIGH_HOUR) {
        next();
    } else {
        req.flash("error", "No visits allowed for this page at " + new Date());
        res.redirect("/");
    }
}
{{</code>}}

### Acceso restringido mixto basado en tiempo

Vamos a explorar ahora la situación en la que dejemos pasar a todos los usuarios a una determinada página, pero que durante un rango de horas sólo sea accesible para admins.

{{< code language="js" title="controllers/session.js" expand="Show" collapse="Hide" isCollapsed="false" >}}
exports.hourRestrictionMixed = (req, res, next) => {
    // establecemos la hora inicio y final de la franja donde admitiremos visitas solo de admins
    const LOW_HOUR = 17;
    const HIGH_HOUR = 19;

    // vemos si el usuario logueado es admin
    const isAdmin = !!req.loginUser.isAdmin;

    // obtenemos la hora actual en el momento que se hace la request y sacamos la hora (sólo número de hora en formato 24h)
    const now = new Date().getHours();

    // comprobamos si estamos en la *franja prohibida*, si es así vemos si es admin y si ok continuamos al next mw, si no mostramos error y vamos a home.
    // si no estamos en la franja pasamos ok
    if (now >= LOW_HOUR && now < HIGH_HOUR) {
        if (isAdmin) {
            next();
        } else {
            req.flash("error", "No visits allowed from normal users for this page at " + new Date());
            res.redirect("/");
        }
    } else {
        next();
    }
}
{{</code>}}

Aplicaríamos este middleware antes como cualquier otro.

### Acceso restringido para admins durante una franja de tiempo y para nadie fuera de ella

Ahora veremos el caso en el que emplearemos cascadas de middlewares para implementar las funcionalidades que queramos. Esto es por si en el examen nos dan middlewares ya hechos y disponibles y nos piden usarlos. Para este ejemplo consideraremos que tenemos un middleware ya proporcionado que es `sessionController.isAdmin` que nos deja ontinuar si somos admins y hemos implementado el primer middleware por horas de este post: `sessionController.hourRestrictionAnyUser`.

El truco está en usar estos dos middlewares para comprobar que somos admins primero, y luego comprobar si estamos en las horas permitidas, aunque hayamos diseñado el middleware para "todos los usuarios". Esto es posible gracias a la independencia de los middlewares: uno se centra en comprobar el acceso basado en tiempo y el otro basado en roles. Si ambos los usamos conjuntamente crearemos una condición no-excluyente. Claro está que si necesitamos comportamientos más avanzados podemos implementar soluciones más avanzadas como `sessionController.hourRestrictionMixed`.

Para hacer este encadenado de middlewares procedemos así:

{{< code language="js" title="routes/index.js" expand="Show" collapse="Hide" isCollapsed="false" >}}
routes.get("/scores", sessionController.isAdmin, sessionController.hourRestrictionAnyUser, scoresController.index);
{{</code>}}

Esto es todo por este post :)
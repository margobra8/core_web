+++
title = "Cambiar el aspecto"
date = "2021-05-31T18:04:17+02:00"
author = "Marcos Gómez"
authorTwitter = "usualmarcos" #do not include @
cover = ""
tags = ["css", "aspecto", "html", "img"]
keywords = ["", ""]
description = "En este post se detalla como cambiar el aspecto del proyecto Quiz"
showFullContent = false
+++

## Aspecto visual

En el examen nos pueden pedir que cambiemos el aspecto de la interfaz del proyecto. Por lo general modificaremos el aspecto empleando CSS mediante la modificación de los archivos `public/stylesheets/*.css` que vienen en el proyecto.

Un ejemplo sería cambiar la fuente de la interfaz eso lo podemos hacer modificando el siguiente archivo:

{{< code language="css" title="public/stylesheets/layout.css" id="1" expand="Show" collapse="Hide" isCollapsed="false" >}}
body {
  font: "Open Sans";
}
{{</code>}}

Otro ejemplo lo tenemos si queremos cambiar el típico fondo amarillo que hay en los cuadros de respuestas por otro color. Para cambiarlo a verde podemos, por ejemplo, hacer lo siguiente:

{{< code language="css" title="public/stylesheets/layout.css" id="1" expand="Show" collapse="Hide" isCollapsed="false" >}}
input[type=text]:focus, input[type=password]:focus {
  background-color: green;
  height: 24px;
}
{{</code>}}

Por último a continuación se muestra si queremos poner el color del texto de la página web de diferente color (ej. morado):

{{< code language="css" title="public/stylesheets/layout.css" id="1" expand="Show" collapse="Hide" isCollapsed="false" >}}
body {
  color: purple;
}
{{</code>}}

## Adición de imagen en la cabecera

Nos pueden pedir añadir una imagen en la cabecera de la página web donde ahora mismo se muestra el texto "The Quiz Site".

{{< image src="/img/header.png" alt="header" position="center" >}}

Para eso tenemos que editar el HTML que ahora mismo se renderiza como vistas a través de el motor de renderizado EJS. Para editar el marcado HTML de la página web, lo debemos hacer en `views/*`. En este caso como queremos editar el documento base (layout) editaremos `views/layout.ejs`.

{{< code language="html" title="views/layout.ejs" id="1" expand="Show" collapse="Hide" isCollapsed="false" >}}
[...línea 22]
<header class="main" id="mainHeader">

    <div class="right">
        <% if (!locals.loginUser) { %>
            <a href="/login">Login</a>
        <% } else { %>
            <a href="/users/<%= loginUser.id %>"><%= loginUser.displayName %></a>
            <a href="/login?_method=DELETE">Logout</a>
        <% } %>
    </div>
    <h1><span class="no-narrow">The</span> Quiz <span class="no-narrow">Site</span></h1>
    <!-- aquí insertamos la imagen que nos piden -->
    <img src="http://example.com/headerlogo.png" />
</header>
[...]
{{</code>}}

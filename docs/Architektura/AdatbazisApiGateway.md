# Adattárolás és Api Gateway

## Előadás

[Adattárolás mikroszolgáltatásokban](https://edu.vik.bme.hu/mod/resource/view.php?id=22404)

[Api Gateway](https://edu.vik.bme.hu/mod/resource/view.php?id=22402)

## Cél

A labor célja egy mikroszolgáltatás architektúrájú alkalmazás felépítésének megismerése, modern adattárolás módszerek kipróbálása, valamint az Api Gateway használatának megértése.

## Előkövetelmény

- Docker Desktop
- Visual Studio 2019 legalább v16.6-ra felfrissítve az alábbi worklodokkal
    - Web development
    - .NET Core cross-platform development
- Visual Studio Code
- Postman
- Kiinduló projekt: <https://github.com/bmeviauav42/todoapp>

## Feladat

1. Checkoutoljuk a minta alkalmazás kódját és ismerjük meg a rendszer felépítését.

2. Nyissuk meg Visual Studion-ban a solution-t és ismerjük meg a forráskód felépítését.

3. Ismerjük meg a `users` mikroszolgáltatás funkcióit:

    - Indítsunk el egy mongodb-t a _compose_ fájlban.

    - Nézzük meg a `/api/users` címeket kiszolgáló kérések Python kódját.

    - Próbáljuk ki a lekérdezéseket Postmanból (a szolgáltatás a <http://localhost:5083/api/users> címen érhető el).

4. Ismerjük meg a`todos` mikroszolgáltatás funkcióit:

    - Nézzük meg a `/api/todos` címeket kiszolgáló kérések C# kódját és a mögöttes repository-t, valamint a Redis-alapú cache-elést.

    - Próbáljuk ki a lekérdezéseket Postmanból (a szolgáltatás a <http://localhost:5081/api/todos> címen érhető el.).

5. Ismertjük meg a user törlés folyamatát. A törlési kérés a `users` mikroszolgáltatáshoz érkezzen, de a törölt userhez tartozó note-okat is töröljük.

    - A `users` szolgáltatásban törlés esetén értesíti a `todos`-nak Redis-en keresztül.

    - A `todos` szolgáltatás háttérben futó pollozással dolgozza fel a törlés feladatokat.

6. Ismerjük meg a Traefik API Gateway-t.

    - Nézzük meg a Traefik konténer elindítását és a többi konténeren a label-öket.

    - A gatway-en keresztül egy porton érhető el a teljes alkalmazás a <http://localhost:5080> címen.

7. Konfiguráljuk be a _forward authentication_-t.

    - Próbáljuk ki, hogyan működik a REST API, ha a `users` mikroszolgáltatás kódjában a `/api/auth` válaszát lecseréljük.

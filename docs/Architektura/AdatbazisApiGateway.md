---
title: Adattárolás és Api Gateway
---

## Előadás

[Adattárolás mikroszolgáltatásokban](https://www.aut.bme.hu/Upload/Course/VIAUAV42/hallgatoi_jegyzetek/04-Adatbazisok.pdf)

[Api Gateway](https://www.aut.bme.hu/Upload/Course/VIAUAV42/hallgatoi_jegyzetek/05-API%20Gateway.pdf)

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

3. Implementáljuk a `users` mikroszolgáltatásban a `GET /api/users` és `GET /api/users/<id>` címeket kiszolgáló kéréseket.

   - Indítsunk el egy mongodb-t a _compose_ fájlban.

   - Próbáljuk ki a lekérdezéseket Postmanból (a szolgáltatás a <http://localhost:5083/api/users> címen érhető el.).

4. Implementáljuk a `todos` mikroszolgáltatásban a `GET /api/todos` és `GET /api/todos/<id>` címeket kiszolgáló kéréseket.

   - A repository hiányzó kódját is írjuk meg.

   - Használjuk ki a Redis cache-t az utóbbi lekérdezéshez.

   - Próbáljuk ki a lekérdezéseket Postmanból (a szolgáltatás a <http://localhost:5081/api/todos> címen érhető el.).

5. Implementáljuk user törlést. A törlési kérés a `users` mikroszolgáltatáshoz érkezzen, de a törölt userhez tartozó note-okat is töröljük.

   - Írjuk meg a `users` szolgáltatásban a törlést.

   - Szóljunk a `todos`-nak Redis-en keresztül.

   - A `todos` szolgáltatás háttérben futó pollozással dolgozza fel a törlés feladatokat.

6. Tegyünk egy Traefik API Gateway-t az összes mikroszolgáltatás elé.

   - Most már használható lesz a web frontend a <http://localhost:5080> címen.

7. Konfiguráljuk be a _forward authentication_-t.

   - Próbáljuk ki, hogyan működik a REST API, ha a `users` mikroszolgáltatás kódjában a `/api/auth` válaszát lecseréljük.

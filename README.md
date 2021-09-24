# BME Mikroszolgáltatások és konténeralapú szoftverfejlesztés

![Build docs](https://github.com/bmeviauav42/jegyzetek/workflows/Build%20docs/badge.svg?branch=master)

Jegyzetek, feladatok és példa kódok [BMEVIAUAV42 Mikroszolgáltatások és konténeralapú szoftverfejlesztés](https://www.aut.bme.hu/Course/VIAUAV42/) tárgyhoz.

A jegyzetek MkDocs segítségével készülnek és GitHub Pages-en kerülnek publikálásra: <https://bmeviauav42.github.io/jegyzetek>.

### Helyi gépen történő renderelés

- Visual Studio Code és [Remote Containers](https://aka.ms/vscode-remote/download/extension)
  1. VS Code-ban `Remote-Containers: Open Folder in Container...` paranccsal kell megnyitni a könyvtárat
  1. VS Code-on belül egy új terminálban: `mkdocs serve`

- Konzolból
  1. Powershell konzol nyitása a repository gyökerébe
  1. `docker run -it --rm -p 8000:8000 -v ${PWD}:/docs squidfunk/mkdocs-material:7.3.0`

A helyi verzió <http://localhost:8000> címen érhető el böngészőben. A Markdown forrás szerkesztése és mentése után automatikusan frissül a weboldal.

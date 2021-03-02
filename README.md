# BME Mikroszolgáltatások és konténeralapú szoftverfejlesztés

![Build docs](https://github.com/bmeviauav42/jegyzetek/workflows/Build%20docs/badge.svg?branch=master)

Jegyzetek, feladatok és példa kódok [BMEVIAUAV42 Mikroszolgáltatások és konténeralapú szoftverfejlesztés](https://www.aut.bme.hu/Course/VIAUAV42/) tárgyhoz.

A jegyzetek MkDocs segítségével készülnek és GitHub Pages-en kerülnek publikálásra: <https://bmeviauav42.github.io/jegyzetek>.

#### Helyi gépen történő renderelés

1. Powershell konzol nyitása a repository gyökerébe

1. `docker run -it --rm -p 8000:8000 -v ${PWD}:/docs squidfunk/mkdocs-material:7.0.3`

1. <http://localhost:8000> megnyitása böngészőből.

1. Markdown szerkesztése és mentése után automatikusan frissül a weboldal

---
title: Calculer la couverture de code d'un projet .Net
date: '2018-07-24 23:59:00 Z'
layout: post
author: Yvan Razafindramanana
description: Couverture de code .Net
excerpt_separator: "<!--more-->"
---

Avec [AltCover](https://github.com/SteveGilham/altcover)
et [ReportGenerator](https://danielpalme.github.io/ReportGenerator/)
il est relativement facile de générer des statistiques au format
HTML sur la couverture de
votre code .Net par les tests unitaires.

<!--more-->

```powershell
dotnet test /p:AltCover=true
ReportGenerator.exe -reports:coverage.xml -targetdir:./coverage
```
Article inspiré par l'excellent 
[Scott Hanselman](https://www.hanselman.com/blog/AltCoverAndReportGeneratorGiveAmazingCodeCoverageOnNETCore.aspx).

## Principe

La couverture de code est une mesure intéressante (mais pas infaillible) pour analyser comment
votre code est effectivement testé unitairement. Elle indique
quelles lignes de code sont exécutées par les tests unitaires
et dans quelle mesure les différents branchements
dans le code sont testés : par exemple si toutes les valeurs possibles d'un ```if``` sont testées,
si toutes les clauses ```catch``` sont testées, etc.

Et pourtant cette métrique est difficile à obtenir dans Visual Studio (ou VS Code) pour le commun des mortels,
sauf si vous avez une licence Entreprise 💶 ou si vous utilisez une plateforme d'intégration continue comme 
VSTS avec des builds bien configurés 😇.

Heureusement la communauté fournit des initiatives ouvertes permettant d'obtenir ces informations à moindre
frais. Voici une solution, parmi les multiples options disponibles, utilisant une combinaison de deux outils
open source.

## AltCover

[AltCover](https://github.com/SteveGilham/altcover) va d'abord 
lancer vos tests unitaires, les analyser et produire un ficher
de couverture de code ```coverage.xml```.

On peut installer cet outil dans son projet de tests avec NuGet :

```powershell
cd ...MonProjet.Tests

dotnet add package AltCover
```

Puis lancer les tests unitaires et produire le ficher ```coverage.xml``` :

```powershell
dotnet test /p:AltCover=true
```

Le framework de tests unitaires utilisé importe peu, tant qu'il
peut être lancé avec ```dotnet test```.

## ReportGenerator

Le fichier généré par AltCover est un fichier XML au format OpenCover.
Diffilement lisible par un être humain, il peut être transformé en pages HTML
par [ReportGenerator](https://danielpalme.github.io/ReportGenerator/).

On peut installer cet outil, également avec NuGet :

```powershell
dotnet add package ReportGenerator
```

Petite astuce pour continuer, il faut retrouver le chemin vers l'exécutable
ReportGenerator.exe, caché quelque part dans votre répertoires de packages NuGet.

```powershell
Get-ChildItem $env:USERPROFILE\.nuget\packages\reportgenerator -Recurse -File `
	| Where-Object { $_.Name -eq "ReportGenerator.exe" } `
	| ForEach-Object { $_.FullName }
```

Enfin lancer la génération du rapport HTML :

```powershell
c:\...ReportGenerator.exe -reports:coverage.xml -targetdir:./coverage
```

Le rapport est alors accessible dans ```./coverage/index.htm```.

## En résumé

Voici un script Powershell pour automatiser ces étapes
(sauf l'installation des packages NuGet).

```powershell
$reportgenerator = Get-ChildItem $env:USERPROFILE\.nuget\packages\reportgenerator -Recurse -File `
	| Where-Object { $_.Name -eq "ReportGenerator.exe" } `
	| ForEach-Object { $_.FullName }

dotnet test /p:AltCover=true /p:AltCoverXmlreport="./coverage/coverage.xml"

& $reportgenerator -reports:./coverage/coverage.xml -targetdir:./coverage/report

Start-Process ".\coverage\report\index.htm"
```

Autre astuce, générez le rapport dans le répertoire ```./coverage```
pour ne pas polluer la racine de votre projet avec les fichiers générés.

La dernière commande ```Start-Process``` est optionelle, elle ouvre
le rapport dans votre navigateur préféré.

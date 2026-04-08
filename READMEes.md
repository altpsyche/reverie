<h1 align="center">Reverie</h1>

Reverie es un entorno de desarrollo enfocado en el análisis y la modificación de memoria de juegos y aplicaciones para uso personal. Es el sucesor de Cheat Engine, refactorizado para eliminar monetización, telemetría y soporte para sistemas operativos obsoletos, y rebautizado como una cadena de herramientas limpia, profesional y moderna.


## Instrucciones de compilación

Reverie se compila con Lazarus 4.x / FPC 3.2.2 (o posterior) en Windows 10+.

  1. Instale Lazarus 4.x con FPC 3.2.2 (compilador cruzado de 32 y 64 bits).
  2. Abra `Cheat Engine/cheatengine.lpi` en el IDE de Lazarus, o ejecute:

         lazbuild --build-mode="Release 64-Bit" "Cheat Engine/cheatengine.lpi"

  3. Opcionalmente, compile los proyectos secundarios que desee utilizar (consulte `README.md` para la lista completa).

## Contribuir al proyecto

  1. Haga un fork del repositorio en GitHub.
  2. Cree una rama para sus cambios.
  3. Envíe (push) su rama a su fork personal.
  4. Abra un Pull Request contra el repositorio principal de Reverie.

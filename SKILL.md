---
name: breaklab
description: Busca fallos reales en código ya escrito con técnicas que el usuario elige (revisores rivales, propiedades, fuzzing, diferencial, mutación, metamórficas) y los demuestra con tests.
argument-hint: "[archivos, endpoint o módulo] [objetivo opcional]"
disable-model-invocation: true
---

# Breaklab

Objetivo: encontrar fallos reales en un código concreto y dejarlos demostrados con tests. **No arreglas el código.** Un hallazgo solo cuenta si un test lo demuestra al ejecutarse.

Entrada: `$ARGUMENTS` (archivos, endpoint o módulo a revisar y, opcionalmente, el objetivo). Si falta, pregúntalo.

## Fase 1. Preparar el terreno

1. **Alcance:** identifica los archivos de código y sus tests. Lista rutas concretas.
2. **Stack:** detecta herramientas en composer.json (Pest/PHPUnit, Infection, pest-plugin-mutate), package.json (Vitest/Jest, fast-check, Stryker) o pyproject (pytest, Hypothesis, mutmut). Anota lo que falta; no instales nada sin preguntar.
3. **Línea base:** corre la suite actual del alcance. Si ya falla, detente y avísalo: no se verifica sobre una base rota.
4. **Criterios de aceptación:** busca un archivo de criterios (p. ej. `docs/criterios/<tema>.md`) o pídelos al usuario. Conviértelos en reglas verificables y muéstraselas. Marca como **ambigüedad** todo lo que ni el código ni el usuario resuelven (¿18 incluido?, ¿decimales?, ¿máximo?). No las resuelvas tú: pregúntalas ahora si bloquean, si no van al informe.
   Guarda los criterios en `.breaklab/<fecha>-<slug>/criterios.md`.

## Fase 2. Elegir técnicas

Usa AskUserQuestion (si no está disponible, pregunta en texto con lista numerada). Haz dos preguntas, ambas con selección múltiple. Pon "(Recomendado)" al final de las opciones que mejor encajen con el código según la guía de abajo y colócalas primero.

**Pregunta 1**, header "Buscar fallos": "¿Qué formas de buscar fallos en el código quieres usar? Puedes elegir varias."
- Revisores rivales: "Dos revisores que no vieron cómo se escribió el código intentan romperlo."
- Reglas que nunca fallan: "Defines reglas que siempre deben cumplirse y se prueban con miles de datos al azar."
- Datos basura: "Se envían miles de entradas raras o corruptas para ver si algo se cae."
- Antes vs. después: "Se compara la versión anterior con la nueva para detectar cambios no deseados."

**Pregunta 2**, header "Tus tests": "¿Quieres además poner a prueba tus tests o la coherencia de los resultados?"
- Sabotaje controlado: "Se meten errores a propósito en el código para ver si tus tests los notan."
- Coherencia de resultados: "Se comprueba que cambiar la entrada cambie la salida de forma lógica (p. ej., filtrar más nunca da más resultados)."
- Ninguna de estas

Guía para recomendar:

| Tipo de código | Recomienda |
|---|---|
| Validación de entradas, endpoints, formularios | Revisores rivales, Datos basura, Sabotaje |
| Cálculos, reglas de negocio, algoritmos, conversión de formatos | Reglas que nunca fallan, Sabotaje |
| Búsquedas, filtros, reportes | Coherencia de resultados |
| Migración, refactor, reescritura | Antes vs. después |
| Parsers, lectura de archivos o datos externos | Datos basura, Reglas que nunca fallan |

Antes de lanzar, di en una línea cuántos subagentes vas a usar y qué hace cada uno.

## Fase 3. Ejecutar

Lanza un subagente por técnica con la herramienta Agent (tipo general-purpose), en paralelo cuando sean independientes. Incluye en el prompt de cada uno estas reglas comunes:

- No reciben esta conversación. Solo: rutas del alcance, `criterios.md`, el comando para correr tests y su técnica.
- Su trabajo es encontrar fallos reales. "No encontré nada relevante" es una respuesta válida; no se inventan hallazgos para llenar.
- Cada hallazgo exige evidencia ejecutada: el comando o test que corrieron y su salida. Sin evidencia, no se reporta.
- No modifican código de producción. Solo escriben archivos nuevos de test marcados con el grupo `breaklab` (Pest: `->group('breaklab')`; Vitest/Jest: `describe('breaklab', ...)`; pytest: `@pytest.mark.breaklab`).
- Si un hallazgo depende de algo que los criterios no definen, se reporta como ambigüedad, no como fallo.
- Formato de cada hallazgo:

```
- id: <tecnica>-<n>
  archivo: ruta:linea
  descripcion: una frase
  reproduccion: comando o entrada exacta
  evidencia: salida relevante (recortada)
  gravedad: alta | media | baja
  confianza: 0-100
  tipo: fallo | ambiguedad | test-debil
```

### Instrucciones por técnica

**Revisores rivales.** Dos subagentes con enfoques distintos, nunca el mismo prompt:
- Revisor A (entradas): límites, tipos incorrectos, vacíos, nulos, decimales, notación científica, arreglos donde se espera un valor simple, tamaños extremos, caracteres especiales, autorización.
- Revisor B (tests y reglas): tests que pasan sin afirmar nada útil, aserciones débiles, mocks que ocultan el comportamiento real, reglas de `criterios.md` sin test, caminos de error no cubiertos.

**Reglas que nunca fallan.** Deduce de `criterios.md` y del código entre 3 y 8 propiedades (invariantes, ida y vuelta, idempotencia, equivalencia con una versión simple). Escríbelas con la librería del stack: fast-check, Hypothesis; en PHP, Eris si está disponible, o si no un test con cientos de casos aleatorios generados con semilla fija e impresa. Reporta el contraejemplo más pequeño.

**Datos basura.** Genera cientos o miles de entradas malformadas (tipos cruzados, cadenas enormes, unicode, JSON roto, campos duplicados o faltantes) contra la función o el endpoint. Es fallo una excepción no controlada, un error 500, un cuelgue o un estado corrupto; un 422 bien formado no lo es. Guarda las entradas que rompen. En el informe, indica que esto es fuzzing ligero, no guiado por cobertura.

**Antes vs. después.** Requiere una referencia (rama base, commit o sistema anterior); si no la hay, pídela. Ejecuta ambas versiones con las mismas entradas (aleatorias o muestreadas), normaliza lo irrelevante (espacios, orden, marcas de tiempo) y reporta diferencias. Las diferencias intencionales van a una lista de cambios aprobados que confirma el usuario.

**Sabotaje controlado.** Usa la herramienta del stack: `./vendor/bin/pest --mutate --parallel` (los tests necesitan `covers()` o `mutates()`), Infection, Stryker o mutmut. Si no está instalada, pregunta antes. Por cada mutante sobreviviente relevante, escribe un test que **pase** con el código actual y **falle** con el mutante. Reporta el puntaje antes y después. Tipo: `test-debil`.

**Coherencia de resultados.** Define entre 3 y 6 relaciones (agregar un filtro nunca aumenta resultados; reordenar la entrada no cambia un total; duplicar un dato no altera un conteo de únicos) y pruébalas con varias entradas.

## Fase 4. Juez

Lanza un subagente juez nuevo. No juzgues tú: esta sesión pudo haber escrito el código. El juez recibe los hallazgos, `criterios.md` y las rutas, y para cada hallazgo:

1. Lo reproduce. Si no se reproduce, lo rechaza con el motivo.
2. Si se reproduce y viola un criterio, escribe (o valida el que dejó el revisor) un test del grupo `breaklab` que falle hoy por esa razón exacta, lo ejecuta y adjunta la salida. Solo entonces queda **confirmado**.
3. Si depende de algo no definido, lo marca como ambigüedad.
4. Funde duplicados entre técnicas y anota cuáles lo encontraron.
5. Para `test-debil`, confirma que el test nuevo pasa con el código actual.

El juez no arregla código.

## Fase 5. Entregar

Escribe `.breaklab/<fecha>-<slug>/informe.md` y `hallazgos.json` (los mismos campos, para automatizar después). El informe lleva:

1. **Veredicto** en dos líneas y contadores: fallos confirmados por gravedad, tests reforzados, rechazados, ambigüedades.
2. **Fallos confirmados**, de mayor a menor gravedad: archivo:línea, qué falla, cómo reproducirlo, ruta del test que lo demuestra y técnica(s) que lo encontraron.
3. **Preguntas para ti**: cada ambigüedad como una decisión concreta con sus opciones.
4. **Tests reforzados y métricas** por técnica: puntaje de mutación antes y después, propiedades probadas, entradas generadas, semillas.
5. **Rechazados**, con el motivo en una línea.
6. **Lo que no se cubrió** y los límites de las técnicas usadas.
7. **Reglas candidatas** para CLAUDE.md o REVIEW.md cuando un patrón aparezca dos o más veces. Proponlas, no las apliques.

En el chat responde con el veredicto, los fallos de gravedad alta, las preguntas pendientes y la ruta del informe. Recuerda que los tests del grupo `breaklab` que demuestran fallos fallan a propósito: se excluyen de la suite normal (p. ej. `--exclude-group=breaklab`) hasta corregir el código, y el siguiente paso es arreglarlo hasta que pasen.

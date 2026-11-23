# Breaklab

Skill de Claude Code que busca fallos reales en tu código y los demuestra con tests que fallan.

*A Claude Code skill that hunts for real bugs in your code and proves each one with a failing test.*

---

## Qué hace

Breaklab toma un código ya escrito (archivos, un endpoint o un módulo) e intenta romperlo con las técnicas que tú eliges. Un hallazgo solo cuenta si un test lo demuestra al ejecutarse; lo que no se reproduce se descarta.

**No arregla el código.** Entrega los fallos demostrados, las dudas que solo tú puedes resolver y los tests que los prueban, para que el arreglo sea el siguiente paso.

## Instalación

Copia la carpeta en el directorio de skills de Claude Code:

```bash
git clone https://github.com/Fsjaimes/breakerlab-skill.git ~/.claude/skills/breaklab
```

Para un solo proyecto, cópiala en `.claude/skills/breaklab/` dentro de ese proyecto.

## Uso

```
/breaklab app/Http/Controllers/UserController.php validar el registro de usuarios
```

El argumento es lo que se va a revisar y, opcionalmente, el objetivo. La skill solo se activa cuando la llamas (`disable-model-invocation: true`).

## Cómo funciona

| Fase | Qué pasa |
|---|---|
| 1. Preparar | Identifica el alcance y el stack, corre la suite actual (si ya falla, se detiene) y fija los criterios de aceptación. |
| 2. Elegir | Te pregunta qué técnicas usar y recomienda las que mejor encajan con el tipo de código. |
| 3. Ejecutar | Lanza un subagente por técnica, en paralelo. Cada hallazgo debe traer evidencia ejecutada. |
| 4. Juzgar | Un subagente juez nuevo reproduce cada hallazgo y solo confirma los que quedan demostrados con un test. |
| 5. Entregar | Escribe el informe y un JSON con los hallazgos. |

## Técnicas

| Técnica | En qué consiste | Útil para |
|---|---|---|
| Revisores rivales | Dos revisores que no vieron cómo se escribió el código intentan romperlo. | Validaciones, endpoints, formularios |
| Reglas que nunca fallan | Propiedades que siempre deben cumplirse, probadas con miles de datos al azar. | Cálculos, reglas de negocio, algoritmos |
| Datos basura | Miles de entradas raras o corruptas para ver si algo se cae (fuzzing ligero). | Parsers, entradas externas |
| Antes vs. después | Compara la versión anterior con la nueva para detectar cambios no deseados. | Migraciones, refactors |
| Sabotaje controlado | Mete errores a propósito para ver si tus tests los notan (pruebas de mutación). | Medir la calidad de los tests |
| Coherencia de resultados | Comprueba que cambiar la entrada cambie la salida de forma lógica. | Búsquedas, filtros, reportes |

## Qué entrega

Todo queda en `.breaklab/<fecha>-<slug>/` dentro del proyecto revisado:

- `criterios.md`: las reglas verificables que se usaron.
- `informe.md`: veredicto, fallos confirmados por gravedad, preguntas para ti, métricas, rechazados y lo que no se cubrió.
- `hallazgos.json`: los mismos hallazgos en formato estructurado.

Los tests que demuestran fallos se marcan con el grupo `breaklab` y **fallan a propósito**. Exclúyelos de la suite normal hasta corregir el código:

```bash
./vendor/bin/pest --exclude-group=breaklab
```

## Stacks compatibles

| Stack | Tests | Propiedades | Mutación |
|---|---|---|---|
| PHP | Pest, PHPUnit | Eris o generación con semilla fija | pest-plugin-mutate, Infection |
| JavaScript / TypeScript | Vitest, Jest | fast-check | Stryker |
| Python | pytest | Hypothesis | mutmut |

Breaklab no instala nada sin preguntar.

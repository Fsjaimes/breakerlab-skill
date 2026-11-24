# Breaklab Skill

> Busca fallos reales en tu código y los demuestra con tests que fallan.
>
> *Hunts for real bugs in your code and proves each one with a failing test.*

Una skill para [Claude Code](https://claude.com/claude-code) que toma un código ya escrito (archivos, un endpoint o un módulo) e intenta romperlo con las técnicas que tú eliges: revisores rivales, propiedades, fuzzing, pruebas diferenciales, mutación y pruebas metamórficas. Un hallazgo solo cuenta si un test lo demuestra al ejecutarse; lo que no se reproduce se descarta.

**No arregla el código.** Entrega los fallos demostrados, las dudas que solo tú puedes resolver y los tests que los prueban, para que el arreglo sea el siguiente paso.

## Qué genera

Todo queda en `.breaklab/<fecha>-<slug>/` dentro del proyecto revisado:

| Archivo | Qué incluye |
|---|---|
| `criterios.md` | Las reglas verificables que se usaron y las ambigüedades detectadas |
| `informe.md` | Veredicto, fallos confirmados por gravedad (cada uno explicado en lenguaje sencillo y en detalle técnico, con el comando para reproducirlo listo para copiar), preguntas para ti, tests reforzados y métricas, rechazados, lo que no se cubrió y reglas candidatas para tu `CLAUDE.md` |
| `hallazgos.json` | Los mismos hallazgos en formato estructurado, para automatizar después |
| Tests del grupo `breaklab` | Un test por fallo confirmado, que falla hoy por esa razón exacta |

## Requisitos

| Herramienta | Para qué | Nota |
|---|---|---|
| [Claude Code](https://claude.com/claude-code) | Ejecutar la skill | Usa subagentes (`Agent`) y preguntas (`AskUserQuestion`) |
| Node.js 18+ | Instalar con `npx skills` | Solo para la instalación |
| Suite de tests del proyecto | Línea base y demostración de fallos | Ver [Stacks compatibles](#stacks-compatibles) |

Las herramientas opcionales (fast-check, Hypothesis, Infection, Stryker, mutmut…) dependen de las técnicas que elijas. Breaklab detecta cuáles tienes y **no instala nada sin preguntar**.

## Instalación

### Global, con `-g` (recomendado)

```bash
npx skills add Fsjaimes/breakerlab-skill -g
```

Con `-g` la skill se instala **a nivel de usuario** (en `~/.agents/skills/`, enlazada a `~/.claude/skills/`) y Claude Code la tiene disponible en **todos tus proyectos**. Es lo recomendado: Breaklab sirve para cualquier código que quieras poner a prueba.

### Local al proyecto, sin `-g`

```bash
cd ruta/de/tu/proyecto
npx skills add Fsjaimes/breakerlab-skill
```

Sin `-g` la skill se instala **solo en el proyecto donde ejecutas el comando** (en su carpeta `.claude/skills/`, y registrada en su `skills-lock.json`). Claude Code solo la ve cuando trabaja dentro de esa carpeta. Sirve para fijar la skill a un repositorio concreto o para probar una versión sin afectar al resto de proyectos.

### Actualizar

```bash
npx skills update breaklab        # o: npx skills update -g para las globales
```

### Manual (sin `npx skills`)

```bash
git clone https://github.com/Fsjaimes/breakerlab-skill.git ~/.claude/skills/breaklab
```

## Uso

La skill solo se activa cuando la invocas (`disable-model-invocation: true`), para que nunca se lance sola sobre tu código:

```
/breaklab app/Http/Controllers/UserController.php validar el registro de usuarios
/breaklab src/utils/price.ts el cálculo de descuentos e impuestos
/breaklab el endpoint POST /api/orders
```

El argumento es lo que se va a revisar y, opcionalmente, el objetivo. Si falta, la skill lo pregunta.

## Cómo funciona

| Fase | Qué pasa |
|---|---|
| **1. Preparar** | Identifica el alcance y el stack, corre la suite actual (si ya falla, se detiene) y fija los criterios de aceptación |
| **2. Elegir** | Te pregunta qué técnicas usar y recomienda las que mejor encajan con el tipo de código |
| **3. Ejecutar** | Lanza un subagente por técnica, en paralelo. Cada hallazgo debe traer evidencia ejecutada |
| **4. Juzgar** | Un subagente juez nuevo reproduce cada hallazgo y solo confirma los que quedan demostrados con un test |
| **5. Entregar** | Escribe el informe y el JSON, y te resume el veredicto en el chat |

## Técnicas

| Técnica | En qué consiste | Útil para |
|---|---|---|
| Revisores rivales | Dos revisores que no vieron cómo se escribió el código intentan romperlo | Validaciones, endpoints, formularios |
| Reglas que nunca fallan | Propiedades que siempre deben cumplirse, probadas con miles de datos al azar | Cálculos, reglas de negocio, algoritmos |
| Datos basura | Miles de entradas raras o corruptas para ver si algo se cae (fuzzing ligero) | Parsers, entradas externas |
| Antes vs. después | Compara la versión anterior con la nueva para detectar cambios no deseados | Migraciones, refactors |
| Sabotaje controlado | Mete errores a propósito para ver si tus tests los notan (pruebas de mutación) | Medir la calidad de los tests |
| Coherencia de resultados | Comprueba que cambiar la entrada cambie la salida de forma lógica | Búsquedas, filtros, reportes |

## Stacks compatibles

| Stack | Tests | Propiedades | Mutación |
|---|---|---|---|
| PHP | Pest, PHPUnit | Eris o generación con semilla fija | pest-plugin-mutate, Infection |
| JavaScript / TypeScript | Vitest, Jest | fast-check | Stryker |
| Python | pytest | Hypothesis | mutmut |

## Los tests de Breaklab fallan a propósito

Los tests que demuestran fallos se marcan con el grupo `breaklab` y **fallan hasta que corrijas el código**. Exclúyelos de la suite normal mientras tanto:

```bash
./vendor/bin/pest --exclude-group=breaklab     # Pest / PHPUnit
pytest -m "not breaklab"                       # pytest
```

Cuando el código esté arreglado, esos tests pasan y quedan como pruebas de regresión.

## Corregir los fallos

La carpeta de cada revisión tiene todo lo que necesita quien vaya a corregir, sea una persona o un agente. No hace falta `AGENTS.md` ni ningún otro archivo:

- `hallazgos.json` da, por cada fallo, el test que lo demuestra, el comando para correrlo y su estado (`pendiente`, `esperando-decision` o `corregido`).
- `criterios.md` guarda tus respuestas a las preguntas del informe, en el campo **Decisión** de cada ambigüedad.
- La sección **Para corregir** de `informe.md` fija las reglas: corregir el código y no los tests, no tocar lo que espera una decisión, ir de mayor a menor gravedad y correr la suite completa al final.

Basta con pedirlo:

```
corrige los fallos de .breaklab/2026-11-24-registro-usuarios/
```

## Estructura

```
breakerlab-skill/
├── SKILL.md      # punto de entrada: las 5 fases, las técnicas y el formato de los hallazgos
└── README.md
```

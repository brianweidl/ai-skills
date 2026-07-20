# ai-skills

Colección de **skills de IA** (*agentic skills*): ejemplos de cómo se estructura una capacidad reutilizable para un agente de IA como Claude. Pensado como material de estudio para entender el diseño de skills en ingeniería de software asistida por IA.

## ¿Qué es una "skill"?

Una skill es un conjunto de **instrucciones + recursos** que le enseñan a un agente de IA a ejecutar una tarea concreta de forma consistente y segura. En vez de explicarle todo cada vez, la skill encapsula:

- **El "cómo"** — el procedimiento paso a paso (`SKILL.md`).
- **El contexto** — datos de referencia que el agente consulta (`references/`).
- **Las herramientas** — scripts que el agente ejecuta (`scripts/`).
- **Las plantillas** — archivos base para configurar el entorno (`templates/`).

La clave de una buena skill agéntica: **los guardrails son duros, no dependen de la disciplina del operador.** El diseño asume que del otro lado hay un agente, y las barreras de seguridad se aplican solas.

## Skills en este repo

### `database-example-skill`

Ejemplo de una skill de infraestructura: correr consultas SQL contra las bases de datos de tus apps (local / staging / prod), con barreras de seguridad pensadas para un agente:

- **Lectura** en todos los entornos.
- **Escritura** solo en staging (con confirmación explícita por sentencia).
- **Prod es solo lectura** (doble candado: el agente + un wrapper).
- Excluye bases con PII por defecto.

> ⚠️ Es un **ejemplo genérico**: todos los hosts, puertos, schemas y nombres de apps son placeholders (`example.com`, `ORDERSSTG`, `orders-service`…). Hay que adaptarlos a tu propia infraestructura antes de usarlo. Existe para mostrar *cómo se diseña* una skill, no para correr tal cual.

**Estructura:**

```
database-example-skill/
├── SKILL.md                  # El procedimiento: cómo el agente resuelve una consulta
├── references/
│   ├── app-registry.md       # Mapa app → entorno → {host, schema, PII, segmentación}
│   ├── db-access.md          # Cómo se conecta (cliente, keychain, wrapper, proxy)
│   └── example-queries.md    # Consultas listas para el caso base
├── scripts/
│   └── db-connect            # Wrapper de conexión con gate de solo-lectura en prod
└── templates/
    └── my.cnf.example        # Plantilla de config del cliente MySQL (sin password)
```

## Por dónde empezar

1. Leé **`database-example-skill/SKILL.md`** — es el corazón de la skill y explica todo el flujo de una consulta.
2. Seguí con los archivos de **`references/`** para entender cómo resuelve conexiones y datos.
3. Mirá **`scripts/db-connect`** para ver cómo se implementan los guardrails en código (por ejemplo, el bloqueo de escrituras en prod como defensa en profundidad).

## Concepto para llevarse

Una skill bien hecha no es "un prompt largo": es un **sistema** con procedimiento, contexto, herramientas y guardrails. Diseñarla es un ejercicio de ingeniería — separar el "qué" del "cómo", hacer las barreras verificables, y dejar que el contexto (no la memoria del agente) sea la fuente de verdad.

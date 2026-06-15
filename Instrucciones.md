# Cómo utilizar las skills de SkillForge

> ⚠️ Este documento fue generado con ayuda de inteligencia artificial. Las instrucciones aquí descritas pueden cambiar con el tiempo o no reflejar con exactitud las capacidades más recientes de cada herramienta. Si encuentras información incorrecta o desactualizada, por favor abre un issue o envía una contribución.

Las skills de SkillForge están diseñadas para proporcionar contexto especializado a asistentes y agentes de IA. Cada skill está compuesta por múltiples documentos organizados en una estructura estándar.

## Estructura de una skill

Cada skill se encuentra dentro de una carpeta propia:

```text
skills/
└── wompi-payments/
    ├── SKILL.md
    └── references/
        ├── payments-backend.md
        ├── purchase-component.md
        ├── webhook-handler.md
        └── widget-helper.md
```

### ¿Qué contiene cada archivo?

#### SKILL.md

Es el punto de entrada principal de la skill.

Contiene:

- Objetivo de la skill.
- Instrucciones de uso.
- Flujo recomendado.
- Referencias a documentación complementaria.

Este es el archivo que normalmente debe leer primero la IA.

#### references/

Contiene documentación de apoyo utilizada por la skill.

Puede incluir:

- Implementaciones de referencia.
- Ejemplos de código.
- Arquitecturas.
- Guías de integración.
- Casos de uso específicos.
- Contexto técnico adicional.

La IA debe consultar estos archivos cuando necesite profundizar en una parte concreta de la implementación.

---

## Recomendación general

Siempre que sea posible, proporciona acceso a la carpeta completa de la skill y no únicamente al archivo `SKILL.md`.

De esta forma el modelo podrá utilizar tanto las instrucciones principales como la documentación de referencia.

---

## Uso genérico

Si tu herramienta no tiene soporte específico para skills:

1. Comparte el contenido de `SKILL.md`.
2. Comparte la carpeta `references/`.
3. Indica explícitamente qué skill deseas utilizar.

Ejemplo:

```text
Utiliza la skill wompi-payments ubicada en skills/wompi-payments.
Consulta también los documentos dentro de references cuando necesites detalles de implementación.
```

---

## Cursor

### Opción 1: Mantener las skills dentro del proyecto

```text
skills/
└── wompi-payments/
```

Luego solicita:

```text
Lee la skill ubicada en skills/wompi-payments.
Utiliza SKILL.md como guía principal y consulta references cuando necesites detalles técnicos.
```

### Opción 2: Reglas de Cursor

Puedes crear una regla que indique:

```text
Para cualquier tarea relacionada con pagos Wompi,
consulta primero skills/wompi-payments/SKILL.md
y utiliza los documentos ubicados en references como documentación complementaria.
```

---

## Claude Code

Claude Code funciona especialmente bien cuando las skills están disponibles dentro del repositorio.

### Estructura recomendada

```text
skills/
└── wompi-payments/
```

Luego puedes indicarle:

```text
Lee la skill ubicada en skills/wompi-payments.

Utiliza SKILL.md como fuente principal de instrucciones.

Cuando necesites información específica sobre backend, checkout, widgets o webhooks, consulta los documentos dentro de references.
```

### Integración con CLAUDE.md

```markdown
Para cualquier tarea relacionada con pagos:

1. Leer skills/wompi-payments/SKILL.md
2. Consultar la documentación ubicada en skills/wompi-payments/references
3. Seguir las instrucciones de la skill antes de generar código
```

---

## Kiro

Kiro incluye soporte nativo para skills mediante:

```text
.kiro/skills/
```

### Estructura recomendada

```text
.kiro/
└── skills/
    └── wompi-payments/
        ├── SKILL.md
        └── references/
```

Kiro podrá detectar la skill y utilizar su contenido como contexto del proyecto.

También puedes indicarle explícitamente:

```text
Usa la skill wompi-payments.

Toma SKILL.md como instrucción principal y utiliza la documentación de references cuando sea necesario.
```

---

## TRAE

Puedes almacenar las skills dentro del proyecto:

```text
skills/
```

o

```text
.ai/skills/
```

Y solicitar:

```text
Consulta la skill wompi-payments.

Utiliza SKILL.md como guía principal y los documentos de references como documentación técnica.
```

---

## VS Code + GitHub Copilot

GitHub Copilot no tiene un sistema nativo de skills.

### Recomendación

Mantén la estructura completa:

```text
skills/
└── wompi-payments/
```

Cuando utilices Copilot Chat:

```text
Usa la skill ubicada en skills/wompi-payments.

Lee primero SKILL.md y utiliza los documentos dentro de references para obtener detalles de implementación.
```

---

## Antigravity

Antigravity permite trabajar con contexto persistente.

La recomendación es registrar la ubicación de las skills en las instrucciones del proyecto.

Ejemplo:

```text
Antes de generar código relacionado con pagos:

1. Leer skills/wompi-payments/SKILL.md
2. Consultar los documentos de skills/wompi-payments/references
3. Seguir las prácticas descritas por la skill
```

---

## Buenas prácticas

### Utiliza SKILL.md como punto de entrada

Evita enviar directamente un documento de `references` sin haber proporcionado primero el contexto de `SKILL.md`.

La jerarquía recomendada es:

```text
SKILL.md
    ↓
references/*
```

### Mantén las referencias separadas

Una skill debe describir el proceso general.

La documentación técnica detallada debe mantenerse dentro de `references`.

### Mantén las skills pequeñas y enfocadas

Es mejor tener:

```text
wompi-payments
convex-auth
whatsapp-bot
stripe-subscriptions
```

que una única skill enorme que intente cubrir múltiples responsabilidades.

### Referencia explícitamente la skill

Siempre obtendrás mejores resultados indicando qué skill debe utilizar la IA.

Por ejemplo:

```text
Utiliza la skill wompi-payments.
```

En lugar de:

```text
Implementa un sistema de pagos.
```

---

## Estructura recomendada para SkillForge

```text
SkillForge/
├── skills/
│   ├── wompi-payments/
│   │   ├── SKILL.md
│   │   └── references/
│   │       ├── payments-backend.md
│   │       ├── purchase-component.md
│   │       ├── webhook-handler.md
│   │       └── widget-helper.md
│   │
│   ├── another-skill/
│   │   ├── SKILL.md
│   │   └── references/
│   │
│   └── ...
│
├── README.md
├── USAGE.md
└── LICENSE
```

La regla general es simple:

> Lee primero `SKILL.md`. Utiliza los archivos dentro de `references/` como documentación complementaria cuando necesites información más específica.
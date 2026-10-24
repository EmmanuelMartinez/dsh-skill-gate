# DSH Skill Gate

**Candado skill-first para DeepSeek Harness.** Un tool call de escritura sobre una
ruta de dominio se **deniega** hasta que el agente haya leído, en esa misma
sesión, las skills que el proyecto declara para esa ruta. No es un recordatorio:
la escritura falla con un `deny` accionable.

Es el equivalente en DSH del hook de Antigravity que ya vive en `agent-standards`,
pero como plugin Cordis: sin transcript, sin shell, sin UI.

## Comportamiento

```
write/edit sobre app/Domains/**
  → ¿se leyeron ddd-domain-mapper y laravel-ddd-architecture en esta sesión?
     ├── sí → allow
     └── no → deny con la ruta exacta de la skill que falta
```

## Instalación

En Plugins → **Add plugin**:

```
github:EmmanuelMartinez/dsh-skill-gate#v0.1.0
```

Reinicia la aplicación. El plugin no renderiza nada; su única superficie es el
resultado del tool call.

## Configuración

La config la genera `agent-standards`; el plugin no define su propio mapeo. Vive
en el workspace:

```
.agents/skill-gate.json
```

```json
{
  "failOpen": true,
  "rules": [
    { "prefix": "app/Domains", "skills": ["ddd-domain-mapper", "laravel-ddd-architecture"] },
    { "prefix": "database/migrations/Tolten", "skills": ["database-schema-modeling"] },
    { "prefix": "resources/ts/domains", "skills": ["vue-clean-architecture"] },
    { "prefix": "routes/domains", "skills": ["ddd-domain-mapper"] }
  ]
}
```

- **Orden**: se respeta el orden del archivo; gana la primera regla que hace match.
- **prefix**: relativo al workspace. Match si la ruta es igual al prefijo, empieza
  por `prefix/` o contiene `/prefix/`.
- **failOpen** (default `true`): si no se puede verificar el estado de la sesión,
  permite. `false` deniega.
- **Sin archivo**: no-op total. Instalarlo en cualquier proyecto no rompe nada.
- **Skill declarada que no existe** en `.agents/skills/<skill>/SKILL.md`: se ignora
  (evita deadlocks).

## Qué cuenta como lectura

Por sesión, y solo para esa sesión:

- la herramienta **`skill`** con `{ name: "<skill>" }` (la forma nativa de DSH), o
- una lectura (`read`, `view_file`, `read_file`) de
  `.agents/skills/<skill>/SKILL.md`.

Las rutas se normalizan (`\\` → `/`, relativas contra el workspace, sin `./`).
El estado se libera cuando la sesión se destruye.

## Herramientas y argumentos

| Acción | Herramientas | Argumento de ruta |
| --- | --- | --- |
| Escritura | `write`, `edit`, `apply_patch`, `multi_edit` | `file_path`, `path`, `TargetFile` |
| Lectura | `read`, `view_file`, `read_file` | `file_path`, `path`, `AbsolutePath` |
| Skill | `skill` | `name` |

Cualquier otra herramienta pasa sin evaluación.

## Mensaje de denegación

```json
{
  "kind": "deny",
  "reason": "skill-gate: antes de modificar app/Domains/User/Actions/X.php lee .agents/skills/ddd-domain-mapper/SKILL.md y .agents/skills/laravel-ddd-architecture/SKILL.md."
}
```

Una sola línea, en español, con la ruta relativa del archivo y **solo** las skills
que faltan.

## Desactivarlo

- En Plugins, desactiva **DSH Skill Gate** (su fila en el Loader), o
- borra `.agents/skill-gate.json` del workspace (no-op inmediato).

## Límites conocidos

- Solo cubre las herramientas de escritura listadas arriba; una escritura hecha
  por otra herramienta (p. ej. una terminal) no pasa por el candado.
- El estado de lecturas es por sesión y vive en memoria del host; reiniciar la
  sesión vuelve a exigir las skills (es el comportamiento buscado).
- La sesión debe tener `cwd` para resolver el workspace y la config.
- Sin UI, sin red, sin llamadas al modelo.

## Desarrollo

En el checkout de DeepSeek Harness:

```
node_modules/.bin/tsc -b packages/experimental/skill-gate/tsconfig.json
node_modules/.bin/vitest run packages/experimental/skill-gate/tests/skill-gate.spec.ts
```

El paquete publicado trae el `lib/` ya construido; el repo solo contiene el bundle,
el patch, el README y la licencia.

## Licencia

MIT.

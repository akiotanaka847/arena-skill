# Changelog

## [1.0.0] - 2026-10-07

Primera versión.

### Incluye
- `/arena`: torneo de N subagentes (100 por defecto, 16 con `--quick`) con la misma tarea y una carta de estrategia distinta cada uno.
- Bracket de eliminación directa: ataque, defensa con revisión y juez con rúbrica escrita, hasta que sobrevive una solución.
- `bracket.py`: máquina de estados del torneo en un único `arena.json`, reanudable tras una compactación. Solo biblioteca estándar.
- Repartidor de cartas sin repeticiones y equilibrado: 15 modos de razonar, 12 flujos y 12 estrategias.
- Comparación final a ciegas contra la respuesta rechazada, cuando la hay.
- Los competidores responden en el idioma de la tarea; el orquestador, en el del usuario.
- `SKILL_DIR` resuelto desde la ruta del propio `SKILL.md`, sin depender de variables del harness.

### Notas
- Layout multi-plugin (`plugins/<nombre>/`) para alojar más skills.
- Tests herméticos con `unittest`: un torneo completo de 100 agentes por la línea de comandos, sin red.

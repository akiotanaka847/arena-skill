# arena-skill

**Skills para Claude Code, empezando por un torneo: `/arena`.**

Cuando Claude te da una respuesta que no te convence, en vez de pedirle otra y
otra vez, `/arena` lanza 100 subagentes con la misma tarea, palabra por palabra,
y los enfrenta entre sí hasta que queda una sola solución. Gratis, MIT, sin API
key y sin nada que conectar.

**Nunca toca tus ficheros.** Todo lo que escriben los subagentes queda dentro de
`.arena/`. La respuesta ganadora vuelve a ti, y aplicarla es decisión tuya.

## Instalación

```bash
/plugin marketplace add akiotanaka847/arena-skill
/plugin install arena@arena-skill
```

Como plugin, Claude Code la muestra con espacio de nombres: `/arena:arena`. Si
quieres `/arena` a secas, copia la carpeta de la skill:

```bash
git clone https://github.com/akiotanaka847/arena-skill.git
cp -r arena-skill/plugins/arena/skills/arena ~/.claude/skills/
```

Para un solo proyecto, copia la misma carpeta en `.claude/skills/` del repo.
Requiere Claude Code (lanza subagentes con la herramienta Agent) y Python 3.8 o
superior. No hay nada que instalar con pip.

## SKILLS

| Skill | Qué hace | Docs |
|---|---|---|
| [`/arena`](plugins/arena/skills/arena/SKILL.md) | Torneo de subagentes: misma tarea, cartas de estrategia distintas, bracket de ataque, defensa y juez hasta que sobrevive una solución | [diseño](implementation/arena-diseno.md) |

## Qué resuelve

Pedir "inténtalo otra vez" produce variaciones de la misma idea: el modelo parte
del mismo sitio y llega a sitios parecidos. `/arena` fuerza la diversidad desde
el principio y deja que la selección haga el resto:

1. **Diversidad real.** Cada agente recibe una carta distinta: un modo de
   razonar, un flujo de trabajo y una estrategia. 15 x 12 x 12 = 2.160 cartas, sin
   repetir ninguna.
2. **Presión adversarial.** Cada solución pasa por ataques concretos de un rival
   que piensa distinto, y tiene que defenderse y corregirse.
3. **Juicio con reglas escritas.** Un juez independiente puntúa con una
   [rúbrica](plugins/arena/skills/arena/rubric.md) que puedes leer y cambiar, y
   la aritmética la hace `bracket.py`, no el juez.
4. **Honestidad al final.** Si había una respuesta rechazada, se compara a ciegas
   con la ganadora y se te dice el resultado, aunque gane la vieja.
5. **En tu idioma.** Los competidores responden en el idioma de la tarea: una
   petición en español sale en español.

## Uso rápido

```bash
/arena
```

```bash
/arena --quick escribe el titular de nuestra página de precios
```

```bash
/arena --agents 32 arregla el test intermitente de tests/test_api.py
```

Sin texto, `/arena` toma tu última petición como tarea y la respuesta que no te
gustó como la que hay que superar. Claude también puede recurrir a ella sin el
comando cuando dices algo como "mala respuesta, ponlos a competir"; en ese caso
pregunta antes de gastar nada.

| Flag | Qué hace |
|---|---|
| `--agents N` | N competidores. Por defecto 100. |
| `--quick` | 16 competidores. El ajuste para el día a día. |
| `--seed S` | Misma semilla, mismas cartas y mismo bracket. Por defecto aleatoria, y queda registrada. |
| `--wave W` | Subagentes por oleada. Por defecto 10. Súbelo solo si subiste el límite de Claude Code. |

## Cómo funciona

1. **Spawn.** N subagentes, una llamada a Agent cada uno. Todos reciben el mismo
   texto de la tarea, byte a byte (hay un test que lo comprueba), más su carta.
2. **Ataque.** Las soluciones se emparejan evitando que dos agentes con el mismo
   modo de razonar se enfrenten. Cada lado ataca la solución del otro: qué está
   mal, qué requisito falta, la entrada exacta que la rompe. Hasta 7 ataques,
   etiquetados FATAL, MAJOR o MINOR.
3. **Defensa.** Cada lado responde a cada ataque, concediendo o rebatiendo con
   evidencia, y reescribe su solución corrigiendo todo lo que concedió.
4. **Juicio.** Un juez aparte lee las dos soluciones revisadas, comprueba cada
   ataque por sí mismo y puntúa: corrección 30, completitud 25, robustez 20,
   especificidad 15, claridad 10. El juez nunca ve las cartas. Una solución con un
   fallo fatal verificado no puede ganar a una sin él. El perdedor queda fuera.
5. **Repetir.** Los supervivientes llevan su solución revisada a la ronda
   siguiente. Con un número impar hay un bye, nunca dos veces al mismo agente
   mientras otro espera el suyo.
6. **Resultado.** Queda una solución. Recibes la solución, los ataques que
   superó, su carta y el número de rondas.

El bracket de 100 agentes, según `bracket.py plan`:

```
  round  alive  matches  bye  sub-agent calls  waves
  spawn    100        -    -              100     10
      1    100       50    -              250     25
      2     50       25    -              125     13
      3     25       12  yes               60      8
      4     13        6  yes               30      5
      5      7        3  yes               15      3
      6      4        2    -               10      3
      7      2        1    -                5      3
  total                                 595     70

  alive per round: 100 -> 50 -> 25 -> 13 -> 7 -> 4 -> 2 -> 1
```

Todo el torneo vive en un único `arena.json`. La sesión principal solo ejecuta
el bucle y nunca lee los cientos de ficheros de solución: cada subagente escribe
su trabajo en disco y responde con una línea. Si la conversación se compacta a
mitad de la carrera, `bracket.py next` la retoma desde el fichero.

## Coste

La skill es gratis; los tokens son tuyos, y 100 agentes son muchos.

| Agentes | Rondas | Llamadas a subagentes | Oleadas de 10 |
|---|---|---|---|
| 100, por defecto | 7 | 595 | 70 |
| 64 | 6 | 379 | 49 |
| 32 | 5 | 187 | 28 |
| 16, `--quick` | 4 | 91 | 16 |
| 8 | 3 | 43 | 10 |

Suma una llamada si hay una respuesta rechazada que superar. Usa `--quick` para
lo cotidiano y guarda los 100 para la respuesta que de verdad importa. La skill
imprime estas cifras antes de empezar.

## La letra pequeña

- **"100 versiones de Claude" son 100 subagentes del modelo que estás usando**,
  no 100 modelos distintos. Lo que los diferencia es la carta.
- **Un competidor es su carta más su fichero de solución.** Los subagentes no
  recuerdan nada entre llamadas: cuando a017 ataca en la ronda 3, es un subagente
  nuevo que recibe la carta y la última solución de a017.
- **"La mejor respuesta" es la que sobrevivió a todos los combates**, no una
  prueba de que sea correcta. Por eso ves los ataques que superó.
- **No corren los 100 a la vez.** Claude Code ejecuta como mucho 10 llamadas en
  paralelo por defecto (`CLAUDE_CODE_MAX_TOOL_USE_CONCURRENCY`), así que va por
  oleadas. Si subes ese límite, pasa `--wave` a juego.
- **Espera prompts de permisos** salvo que actives accept-edits (Shift+Tab): cada
  subagente escribe un fichero en `.arena/`.
- **Los subagentes no ven tu chat.** Solo conocen `.arena/<run>/task.md`. Si un
  requisito no llegó a ese fichero, los 100 lo pasan por alto.
- **Misma semilla, mismas cartas y mismo bracket**, no las mismas respuestas.
- **No arregla una tarea mal planteada.** Tarea vaga, 100 sabores de vaguedad.

## Documentación

| Doc | Contenido |
|---|---|
| [implementation/arena-diseno.md](implementation/arena-diseno.md) | Decisiones de diseño y por qué |
| [best-practice/orquestacion-en-disco.md](best-practice/orquestacion-en-disco.md) | Cómo orquestar cientos de subagentes sin llenar el contexto |
| [plugins/arena/skills/arena/rubric.md](plugins/arena/skills/arena/rubric.md) | Los cinco criterios con los que puntúa cada juez |
| [CHANGELOG.md](CHANGELOG.md) | Historial de versiones |

## Ficheros

```
plugins/arena/skills/arena/SKILL.md          orquestación y el brief exacto de cada subagente
plugins/arena/skills/arena/bracket.py        máquina de estados del torneo, solo biblioteca estándar
plugins/arena/skills/arena/strategies.json   15 modos de razonar, 12 flujos, 12 estrategias. Edítalo
plugins/arena/skills/arena/rubric.md         los cinco criterios del juez
tests/test_bracket.py                        los tests
```

## Desarrollo

```bash
python3 -m unittest discover -s tests -v
```

Sin dependencias. El test principal juega un torneo completo de 100 agentes por
la línea de comandos con ganadores aleatorios y comprueba que termina con un
único superviviente; el resto cubre 16, 7 y 1 agentes, las garantías del
repartidor de cartas y cada fase, del spawn a la comprobación final, con
subagentes simulados.

## Licencia

MIT. Ver [LICENSE](LICENSE).

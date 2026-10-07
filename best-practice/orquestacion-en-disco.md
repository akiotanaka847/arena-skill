[← Volver al índice](../README.md)

# Orquestar cientos de subagentes sin llenar el contexto

Una skill que lanza 5 subagentes puede permitirse leer lo que devuelven. Una que
lanza 595, no. El patrón que usa `/arena` sirve para cualquier skill que reparta
trabajo en muchos subagentes.

## El problema

El orquestador es una sesión con un contexto finito. Si cada subagente devuelve
su trabajo en la respuesta, el contexto se llena de soluciones que el
orquestador no necesita leer. Y cuando el harness compacta la conversación, se
pierde lo único que sí necesitaba: quién sigue vivo y qué toca ahora.

## El patrón

Tres reglas, cada una con su razón:

| Regla | Qué evita |
|---|---|
| El estado vive en un fichero, gestionado por un script | Que una compactación borre el progreso |
| Cada subagente escribe en disco y responde con una línea | Que las salidas llenen el contexto |
| El script dice el siguiente paso y el comando exacto | Que el orquestador tenga que recordar en qué fase está |

### 1. Un script dueño del estado

Todo cambio de estado pasa por el script: nunca a mano, nunca de memoria.

```bash
python3 "<SKILL_DIR>/bracket.py" next
```

`next` lee el fichero y contesta qué hacer y con qué comando. Tras una
compactación, el orquestador ejecuta `status` y `next` y sigue donde estaba. La
escritura es atómica (fichero temporal más `os.replace`), así que un corte a
mitad de guardado no deja un JSON roto.

### 2. Briefs en disco, recibos en el chat

El script escribe un brief por trabajo. La llamada al subagente es siempre la
misma frase:

```text
Read <ruta del brief> and follow it exactly. It is your whole brief.
```

Y el subagente contesta con una línea de formato fijo:

```text
DONE a017 412
```

El orquestador acumula recibos, no trabajo. El único fichero de solución que lee
en toda la carrera es el del ganador.

### 3. Comprobar por salidas, no por respuestas

Que un subagente diga `DONE` no prueba que escribiera el fichero. El script
comprueba qué salidas existen en disco (`check`) y `prompts` vuelve a listar
solo las que faltan. Un trabajo que falla dos veces se marca con `NO OUTPUT` y
la regla de la fase decide qué significa (un ataque ausente cuenta como cero
ataques; una solución ausente pierde su combate).

## Cuándo no merece la pena

Por debajo de una decena de subagentes, leer sus respuestas es más simple que
mantener un script de estado. El patrón compensa cuando el número de llamadas
supera lo que cabe cómodamente en un contexto, o cuando la tarea dura lo
bastante como para sobrevivir a una compactación.

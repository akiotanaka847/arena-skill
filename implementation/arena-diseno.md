[← Volver al índice](../README.md)

# `/arena`: decisiones de diseño

Por qué la skill está construida como está. Cada sección documenta una decisión
que no es obvia y el fallo concreto que evita.

## 1. Una sola tarea, byte a byte

Los subagentes no ven la conversación. Lo único que saben es el fichero
`task.md`, y `bracket.py` lo inyecta idéntico en cada brief.

**Fallo que evita.** Si el orquestador redacta la tarea para cada agente, la
parafrasea: un agente recibe un matiz que otro no, y el torneo deja de comparar
estrategias para comparar enunciados. Y si el orquestador mete su propia opinión
de la respuesta correcta, empuja a los 100 en la misma dirección, que es justo lo
contrario de lo que se busca. Un test extrae el bloque de la tarea de cada brief
y comprueba que todos son el mismo texto.

## 2. Las cartas se reparten, no se sortean

Una carta es un triple: modo de razonar (15), flujo de trabajo (12) y
estrategia (12). Son 2.160 combinaciones.

**Fallo que evita.** Sacar 100 cartas al azar de 2.160 da al menos un duplicado
con un 90 % de probabilidad (paradoja del cumpleaños), y deja modos sin usar
mientras otros se repiten. Por eso `deal()` no sortea, construye:

| Garantía | Cómo |
|---|---|
| Ninguna carta se repite | El agente `i` recibe razonamiento `i mod R` y flujo `(i + i // mcm(R, W)) mod W`, que nunca repite flujo dentro de un modo |
| Cada parte se usa de forma pareja | Los recuentos de dos modos cualesquiera difieren como mucho en 1 |
| Dos agentes difieren en al menos dos de las tres partes | Las estrategias son una coloración propia y equilibrada de las aristas de la rejilla razonamiento x flujo (König para existir, intercambios de De Werra para equilibrar) |

La tercera garantía se cumple mientras ningún modo ni flujo tenga más agentes
que estrategias: hasta 144 agentes con las cartas incluidas. La semilla solo
baraja etiquetas y asignación, así que misma semilla, mismo reparto.

## 3. Se emparejan ángulos distintos

`pair_round()` busca para cada agente un rival con otro modo de razonar.

**Fallo que evita.** Dos agentes con el mismo modo tienden a ver los mismos
huecos y a pasar por alto los mismos. Un ataque útil viene de quien mira el
problema desde otro sitio.

## 4. El juez puntúa; la aritmética decide

El juez devuelve cinco notas de 0 a 10 por solución y una marca `fatal`. Quien
gana lo calcula `decide()`:

1. Una solución fatal no puede ganar a una que no lo es.
2. Si no, gana el mayor total ponderado (los pesos de la rúbrica, que un test
   compara con las constantes del código).
3. En empate exacto: menos ataques en pie, luego más corrección, y solo entonces
   la elección del juez.

**Fallo que evita.** Un juez LLM que elige ganador "a ojo" a veces contradice
sus propias notas. Aquí, si su elección no cuadra con su puntuación, ganan las
notas y queda anotado. El juez además nunca ve las cartas: puntúa el trabajo, no
el método.

## 5. Byes justos

Con un número impar de supervivientes, uno pasa sin combate. Se lo lleva el
agente con menos byes acumulados.

**Fallo que evita.** Que el mismo agente llegue a la final con dos pases gratis
mientras otros se han batido en todas las rondas.

## 6. Estado en disco, recibos de una línea

El torneo vive en `arena.json`, escrito de forma atómica (fichero temporal y
`os.replace`). Cada subagente escribe su trabajo en disco y responde con una sola
línea (`DONE a017 412`).

**Fallo que evita.** 595 llamadas con sus soluciones devueltas al chat
desbordan cualquier contexto, y una compactación a mitad de carrera borraría
quién sigue vivo. Con el estado fuera del contexto, `bracket.py next` siempre
sabe el paso siguiente. El patrón general está en
[orquestacion-en-disco.md](../best-practice/orquestacion-en-disco.md).

## 7. Oleadas de 10

Claude Code ejecuta como mucho 10 llamadas a herramientas a la vez por defecto.
`prompts` agrupa los trabajos pendientes en oleadas de ese tamaño y el
orquestador espera a que vuelva una entera antes de lanzar la siguiente.

**Fallo que evita.** Un mensaje con 100 llamadas no corre más rápido si solo 10
se ejecutan a la vez, y cuando algo falla no queda claro en qué punto. Con
oleadas, `check` dice exactamente qué salidas faltan y solo se repiten esas.

## 8. La comparación final es ciega

Si había una respuesta rechazada, un último juez la compara con la ganadora. La
semilla decide cuál es X y cuál es Y, y el juez no sabe cuál es cuál.

**Fallo que evita.** Un juez que sabe cuál viene del torneo tiende a premiarla.
La skill informa el resultado tal cual, incluso cuando gana la respuesta vieja.

## 9. El idioma de la tarea

Los briefs están en inglés, pero los competidores escriben en el idioma de
`task.md`, y el orquestador habla con el usuario en el suyo.

**Fallo que evita.** Unas instrucciones en inglés arrastran la respuesta al
inglés aunque la tarea esté en español: el usuario pidió un titular para su web
y recibe uno que no puede publicar. Código, identificadores y citas se quedan
como están.

## 10. `SKILL_DIR` sin variables del harness

El `SKILL.md` pide al agente la ruta absoluta del directorio del propio
`SKILL.md`, que su harness le dio al cargar la skill, en vez de depender de una
variable de entorno concreta. Y como las variables de shell no sobreviven entre
comandos, la ruta se escribe literal en cada llamada.

**Fallo que evita.** Una variable sin expandir rompe el primer comando y, con
él, todo el torneo. Además, cada pista que imprime `bracket.py` ya lleva su ruta
absoluta, así que el orquestador puede copiarla tal cual.

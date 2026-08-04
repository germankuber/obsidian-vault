---
title: Consistent Hashing
source: https://www.hellointerview.com/learn/system-design/core-concepts/consistent-hashing
author: Hello Interview · Design Gurus
created: 2026-06-08
tags:
  - system-design/storage
  - system-design/patterns
  - type/concept
  - status/permanent
aliases:
  - Consistent Hashing
  - Hashing Consistente
  - Hash Ring
reading:
  total_words: 2630
  read_words: 295
  pct: 11
  last_read: ""
updated: 2026-06-16
---

# Consistent Hashing

> [!note] Definición
> Método de hashing donde agregar o quitar un nodo solo obliga a remapear una **fracción** de las claves, en vez de reshufflear casi todo. Servidores y claves se ubican en un mismo **anillo virtual** y cada clave pertenece al primer nodo que encuentra caminando clockwise.

> [!note] Tesis operativa
> El hashing consistente resuelve **un** problema del [[Sharding]]: cuando cambia N (la cantidad de nodos), mover la **mínima** cantidad de datos. El módulo falla porque ata cada key a N (cambiar N reubica casi todo); el ring **desacopla** la posición de cada key de N (se calcula una sola vez), así cambiar N solo reasigna el arco vecino. Todo lo demás de la nota son las grietas de esa idea: el balance (vnodes), el tráfico (hot spots), y el hecho de que el algoritmo dice *dónde* viven los datos pero no los mueve.

## Marco mental (leé esto primero)

- **De dónde viene el problema**: una sola DB no aguanta el crecimiento (pensá en TicketMaster sumando eventos), así que distribuís los datos en varias instancias — eso es [[Sharding]]. La pregunta madre es: *¿qué evento vive en qué DB?*
- **La hash function** convierte un `event_id` en un número. Ese número es la base TANTO del módulo COMO del ring; la única diferencia entre ambos enfoques es **qué hacés con ese número** después.
- **El objetivo real no es "distribuir"** — distribuir es fácil. El objetivo es **redistribuir poco** cuando el cluster cambia de tamaño. Esa es la métrica con la que se juzga todo lo que sigue.

## La cadena causal

La nota es una sola historia encadenada: el módulo falla → el ring lo arregla → el ring desbalancea → los vnodes lo arreglan → quedan los hot spots de tráfico → y al final el algoritmo igual no mueve datos solo.

1. **Módulo**: atar la key a `% N` hace que cambiar N reubique casi todas las keys.
2. **Hash ring**: poner keys y nodos en un círculo desacopla la posición de N → cambiar N solo toca el arco vecino.
3. **Virtual nodes**: el ring básico manda toda la carga de un nodo caído a UN vecino; los vnodes la reparten.
4. **Hot spots**: los vnodes balancean *keys*, no *tráfico*; una key muy caliente satura su nodo igual.
5. **Data movement**: el ring dice *dónde* deben vivir los datos, no los teletransporta; los sistemas reales usan replicación preexistente.

## Modulo Hashing — por qué falla (el punto de partida)

La mecánica es la intuición obvia: (1) hasheás el `event_id` a un número; (2) hacés módulo con la cantidad de DBs; (3) el resto te dice la DB.

```python
database_id = hash(event_id) % number_of_databases
```

Con 3 DBs reparte parejo:

```text
Event #1234 → hash(1234) % 3 = 1 → Database 1
Event #5678 → hash(5678) % 3 = 0 → Database 0
Event #9012 → hash(9012) % 3 = 2 → Database 2
```

![[modulo-hashing.svg]]
*Modulo Hashing — el flujo `hash(event_id) % N → database_id`*

El **WHY** de la falla está en que el resultado queda **atado al divisor N**: la fórmula de pertenencia de CADA key incluye a N, así que cambiar N cambia el resto para casi cualquier entrada *al mismo tiempo*. El evento #1234 no es mala suerte: `hash(1234) % 3 = 1` (DB1) pero `hash(1234) % 4 = 0` (DB0). Le pasa a la mayoría de las keys porque el residuo respecto de 4 **no tiene relación** con el residuo respecto de 3 — son cuentas distintas.

Lo importante es que hay **dos disparadores y un solo colapso**: agregar capacidad (`%3` → `%4`) y perder un nodo por falla (`%3` → `%2`) son el mismo problema, porque ambos cambian el N que vive *dentro* de la fórmula.

> [!warning] La cadena del desastre
> Cambia N → casi todas las keys cambian de dueño → migración masiva de datos → picos enormes de carga en todo el cluster → usuarios sin acceso o con latencia altísima. Y lo peor: **el movimiento es innecesario** — los datos no cambiaron, solo cambió la fórmula que decide dónde buscarlos.

![[modulo-add-node.svg]]
*Issue adding a Node — pasar de `%3` a `%4` reubica casi todos los eventos, no solo los de la DB nueva*

## El hash ring — la idea que desacopla

El insight es poner los **datos Y los nodos** en un mismo espacio circular. Es un círculo (no una línea) por el **wraparound**: cuando pasás el último nodo volvés al primero, así que ninguna key se queda sin dueño.

Construcción (usando un espacio chico 0–100 para que las cuentas se vean; en la práctica es 0 a 2³²−1, mismo concepto):

1. Un ring con N puntos fijos, p. ej. 100.
2. 4 DBs ubicadas en las posiciones **0, 25, 50, 75**.
3. **Lookup**: hasheás el `event_id` y, en vez de hacer módulo, ubicás ese valor en el ring y caminás **clockwise** hasta la primera DB que aparezca.

El **WHY** del desacople, comparado lado a lado con el módulo:

- En el **módulo**, N está *dentro* del cálculo de cada key (`% N`), así que cambiar N recalcula todas.
- En el **ring**, la posición de la key se calcula **una sola vez** (`hash(key)` → un punto fijo) y esa cuenta **no menciona N**. Al cambiar los nodos, la key no se mueve; lo único que cambia es *quién es el primer nodo clockwise*. Por eso solo se reasigna el arco afectado.

### Agregar un nodo (DB5) — la derivación del ~15%

La regla de propiedad: cada nodo es dueño del arco que va **desde el nodo anterior hasta él** (clockwise).

- Con 4 nodos equiespaciados, cada uno posee ~1/4 del ring (~25%).
- DB1 está en 0; su nodo anterior es DB4 en 75 → **DB1 es dueña del arco 75 → 100/0**, un span de **25 unidades**.
- Metemos **DB5 en la posición 90**. DB5 se queda con el arco **75 → 90 = 15 unidades**; DB1 retiene solo **90 → 0 = 10 unidades**.
- De DB1 se movió **15/25 = 60%** de *sus* keys.
- En proporción al total: DB1 era ~25% del ring, y se movió el 60% de eso → **60% × 25% ≈ 15% de todos los eventos**.
- **Todo lo demás queda exactamente donde estaba**, porque sus arcos no tocan la posición 90.

El contraste con el módulo es el corazón de la nota: antes se movía *casi todo*; ahora se mueve **~15% del total** (solo una parte de un único nodo).

![[hash-ring-db5-added.svg]]
*Anillo con DB5 agregada en la posición 90 — solo migra el rango 75–90*

### Quitar un nodo (DB2)

Quitar **DB2 (pos 25)**: solo migran *sus* eventos, y van al siguiente nodo clockwise = **DB3 (pos 50)**, que hereda el arco huérfano entero. Todo el resto del ring queda intacto.

![[hash-ring-db2-removed.svg]]
*Anillo con DB2 quitada — sus claves pasan a DB3 (pos 50), el resto no se toca*

> [!question] 🎯 ¿De dónde sale el ~15% al agregar DB5 en la posición 90?
> DB1 era dueña del arco 75 → 0 = 25 unidades. DB5 en 90 toma 15 de esas unidades y DB1 retiene 10, así que se movió 15/25 = **60% de las keys de DB1**. Como DB1 era ~25% del ring, eso es 0,60 × 0,25 ≈ **15% del total de eventos**. Lo demás no se toca.

## Virtual nodes — el balance

El problema del ring básico aparece justo al quitar un nodo: quitar DB2 manda **toda** su carga a DB3, que ahora tiene **2x** la carga de DB1 y DB4. El **WHY** es que el arco de DB2 es **contiguo** (25 → 50) y el único nodo siguiente clockwise lo hereda entero. En espejo, al *agregar* un nodo, este le roba carga solo a su único vecino clockwise → balance desparejo.

La solución son los **virtual nodes**: cada DB se coloca en **varios puntos** del ring, hasheando variaciones de su nombre:

```text
hash("DB1-vn1") → posición 20
hash("DB1-vn2") → posición 35
hash("DB1-vn3") → posición 65
```

El **WHY** de por qué funciona: la hash function **desparrama**, así que los vnodes de una misma DB caen en posiciones **dispersas y no contiguas**, entremezcladas con los vnodes de las otras DBs. Entonces, cuando una DB falla, sus arcos son **muchos arquitos chicos esparcidos** por todo el ring, y cada uno es heredado por un vecino **distinto**: `DB2-vn1 → DB1`, `DB2-vn2 → DB3`, `DB2-vn3 → DB4`… → la carga se reparte de forma uniforme en lugar de caer toda sobre un solo nodo.

- **Trade-off cuantitativo**: más vnodes = distribución más uniforme. Más arquitos chicos = menos varianza en cuánto hereda cada vecino.
- **Simetría al agregar**: con vnodes, un nodo nuevo absorbe pedacitos de *varios* nodos a la vez → el cluster queda balanceado desde el arranque, no solo tras una falla.

![[hash-ring-virtual-nodes.svg]]
*Anillo con virtual nodes — cada DB física aparece en varios puntos entremezclados*

> [!question] 🎯 ¿Por qué sin vnodes la caída de un nodo duplica la carga de UN vecino y con vnodes no?
> Sin vnodes, el nodo caído posee **un arco contiguo** que el único vecino clockwise hereda completo (de ahí el 2x). Con vnodes, ese nodo posee **muchos arcos chicos dispersos**, y cada uno cae en un vecino distinto → la carga se reparte parejo en vez de concentrarse.

## Hot spots — balance estructural vs. balance de workload

Acá está la trampa: **los vnodes no alcanzan**. Los vnodes balancean la distribución de *keys*, pero un hot spot es un desbalance de **tráfico**. Una sola key con 100x reads (la página del recital de **Taylor Swift**) vive en **un** nodo y lo satura, aunque ese nodo tenga pocas keys. El hashing consistente distribuye *keys*, no *tráfico* → ningún rebalanceo de keys arregla esto. Son problemas **ortogonales**.

Tres estrategias para el tráfico:

- **Read replicas**: replicás la key caliente en varios nodos y load-balanceás las lecturas (ver [[Load Balancing]]). Es la estrategia **más común**.
- **Key-space salting**: le agregás un sufijo random a la hot key (`taylor-swift-{0..9}`), convirtiendo 1 key en 10 que hashean a nodos distintos. **Costo**: las lecturas se vuelven *scatter-gather* (dispersás la consulta a las 10 y agregás los resultados).
- **Adaptive rebalancing**: monitoreás el tráfico en vivo y movés rangos de keys fuera de los nodos sobrecargados. Es lo más complejo; **DynamoDB** lo automatiza.

> [!tip] El cierre para la entrevista
> Los **vnodes previenen el structural imbalance** (distribución de keys); la **replicación + el salting previenen el workload imbalance** (tráfico). Son dos ejes distintos y no se sustituyen entre sí.

## El movimiento de datos en la práctica — el algoritmo dice DÓNDE, no MUEVE

El malentendido más común: el hashing consistente dice *dónde deben vivir* los datos, pero **no teletransporta** terabytes cuando un nodo se cae. Los sistemas reales **no mueven datos al fallar** — usan **replicación preexistente** montada sobre el ring.

- **DynamoDB**: replica cada partición en **3 availability zones**. Cuando cae el primario, **promueve una réplica** vía un consenso tipo **Raft** → **0 movimiento** de datos, porque la copia ya estaba ahí; solo cambia el líder. (Ver [[Quorum]] y [[Primary-Replica]] para el modelo de replicación detrás.)
- **Cassandra**: replica cada key en **N nodos consecutivos** del ring → las lecturas se sirven desde las réplicas sobrevivientes.

**¿Cuándo SÍ se mueven datos?** Solo en cambios de membresía **planificados** (agregar capacidad, reemplazar permanentemente un nodo para restaurar el replication factor), y aun ahí solo se re-replica una **fracción acotada** de las keys — nunca "casi todo" como en el módulo.

## Redis Cluster — fixed hash slots (el contraejemplo)

No todo usa hashing consistente. **Redis Cluster** usa **16.384 slots fijos** con el mapeo `CRC16(key) mod 16384`, y asigna *rangos de slots* a cada nodo.

- **Ventaja**: es más simple de razonar — el mapeo key→slot es fijo y conocido, no depende de posiciones de nodos hasheadas que se mueven.
- **Costo**: más coordinación al rebalancear — mover slots entre nodos es un paso explícito y orquestado, no algo que "sale solo" del ring.

Es un trade-off de diseño perfectamente discutible en una entrevista, no una decisión obviamente peor.

## Cuándo usarlo / cuándo solo mencionarlo

El hashing consistente aplica a **cualquier cluster**: DBs, caches (ver [[Cache-Aside]]), message brokers ([[Message Queue]]) y application servers ([[Load Balancing]]).

Usos reales:

- **Apache Cassandra** — distribuye los datos sobre el ring.
- **Amazon DynamoDB** — partition placement por debajo ("under the hood").
- **CDNs** ([[Content Delivery Network]]) — decidir qué edge server cachea qué contenido.

La **meta-habilidad** de entrevista es reconocer cuándo profundizar y cuándo solo mencionar — y la mayoría de las entrevistas caen en lo segundo:

- **Sistema gestionado** (usás DynamoDB / Cassandra / un CDN): solo **mencionás** que usan hashing consistente por debajo y seguís.
- **Infra desde cero** (diseñar una DB / cache / message broker distribuido): hacés el *deep dive* y explicás los cinco puntos: (1) por qué CH le gana al módulo; (2) cómo los vnodes mejoran el balance; (3) las estrategias ante fallas y agregados; (4) cómo surgen los hot spots y sus mitigaciones; (5) la relación CH ↔ replicación para tolerancia a fallos.

> [!note] Nota de rigor (Kleppmann / DDIA)
> El término "consistent hashing" se usa de forma floja. Martin Kleppmann (*Designing Data-Intensive Applications*) señala que varios sistemas que dicen usarlo en realidad usan variantes como *hash-based partitioning* con rangos de slots fijos (es decir, el modelo de **Redis**, no el ring clockwise). Lo que importa es el **principio** — minimizar el movimiento de datos en el rebalanceo —; mecánicas distintas (ring clockwise vs. slots fijos) pueden cumplirlo igual.

## ¿Qué aplica a mi caso? (árbol de decisión)

- **¿Estoy distribuyendo datos en un cluster** (DB / cache / broker / app servers)**?**
  - **No** → no aplica.
  - **Sí, y es gestionado** (DynamoDB / Cassandra / CDN) → solo **mencioná** que usa CH por debajo.
  - **Sí, e infra desde cero** → deep dive:
    - *"Cambiar N reubica casi todo"* → **hash ring** + lookup clockwise.
    - *Una falla sobrecarga un solo vecino* → **virtual nodes**.
    - *UNA key con 100x tráfico* → **NO** es problema de vnodes, es **workload imbalance**:
      - caso común → **read replicas**;
      - key muy caliente → **key-space salting** `{0..9}` (scatter-gather);
      - querés que se ajuste solo → **adaptive rebalancing** (lo automatiza DynamoDB).
    - *Tolerancia a fallos sin mover TB* → **replicación preexistente** (DynamoDB 3 AZ + Raft; Cassandra N nodos consecutivos); mover datos solo en cambios **planificados** y solo una fracción acotada.

> [!tip] Mnemónica
> *Arrange everything in a circle and walk clockwise.* Vnodes = balance **estructural**; replicación + salting = balance de **workload**; el ring dice **dónde** viven los datos, pero **no los mueve solo**.

## References

- Fuente original (definición + caso de uso): [50 System Design Patterns Every Engineer Should Know](https://designgurus.substack.com/p/50-system-design-patterns-every-engineer) — Design Gurus, 2026-05-11.
- Profundización técnica (mecánica del ring, derivaciones numéricas, vnodes, hot spots, data movement, Redis slots, nota Kleppmann): deep dive de **Hello Interview** — https://www.hellointerview.com/learn/system-design/core-concepts/consistent-hashing
- Nota de procedencia: la nota original era fina y su guía de trade-offs era en parte conocimiento general; el contenido profundo de esta versión proviene del deep dive de Hello Interview. Las figuras son SVGs de `files.hellointerview.com`.

## Related

- [[_System Design|System Design]]
- [[Sharding]] — el caso de uso principal del hashing consistente
- [[Primary-Replica]] — replicación montada sobre los nodos del ring
- [[Quorum]] — consenso para promover réplicas (DynamoDB / Raft)
- [[Content Delivery Network]] — un CDN usa CH para decidir qué edge cachea qué
- [[Load Balancing]] — repartir lecturas sobre réplicas de una hot key
- [[Message Queue]] — los message brokers distribuidos también particionan con CH
- [[Cache-Aside]] — los caches distribuidos son un caso clásico de CH

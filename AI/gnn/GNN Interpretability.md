---
title: GNN Interpretability
source: https://togoailabs.substack.com/p/what-is-my-graph-neural-network-actually
author: togoAI Labs
created: 2026-06-10
tags:
  - ai/gnn
  - type/concept
  - status/permanent
aliases:
  - GNN Interpretability
  - GNN Explainability
  - Interpretabilidad de GNN
  - Interpretabilidad de Graph Neural Networks
---

# GNN Interpretability

> [!note] Definición
> El problema de entender **por qué un [[Graph Neural Network]] hace una predicción**. Los GNNs son cajas negras: poderosos para aprender de datos en grafo, pero opacos sobre su razonamiento interno. La interpretabilidad busca **espiar dentro** de esa caja.

## Por qué importa

- **Trust & adoption** — en campos de alto riesgo (salud, finanzas) la gente no usa un modelo que no entiende. Un médico no confía en una recomendación de droga sin el porqué; un regulador no aprueba un detector de fraude que es una caja negra total.
- **Debugging & improvement** — cuando el modelo falla, la interpretabilidad ayuda a entender por qué: ¿agarra patrones espurios?, ¿ignora features importantes? Sin ella, estás adivinando.
- **Scientific discovery** — en drug discovery o ciencia de materiales, el "por qué" suele valer más que la predicción. Si un GNN predice que una molécula es tóxica, saber **qué parte** causa la toxicidad es clave para diseñar mejores drogas.
- **Fairness & bias detection** — los modelos aprenden patrones sesgados de los datos; la interpretabilidad detecta cuándo decide en base a algo que no debería.
- **Legal requirements** — muchas industrias exigen que las decisiones automatizadas sean explicables. El **GDPR** de la UE, por ejemplo, da a las personas el derecho a una explicación cuando una decisión automatizada les afecta.

## Qué lo hace difícil

- **El problema de la combinación** — la predicción depende de features + estructura del grafo + las interacciones entre ambos. Desenredar cuál de esos factores impulsó una predicción es difícil.
- **La información se dispersa** — tras varias capas de message passing, la representación de un nodo contiene info de nodos a **varios hops** de distancia; la "señal" original se mezcla y transforma de formas difíciles de rastrear.
- **Transformaciones no-lineales** — como toda red neuronal, los GNNs usan funciones de activación no-lineales y matrices de pesos aprendidas; son matemáticamente complejas y sin interpretación intuitiva.
- **Distintas necesidades de explicación** — a veces querés saber qué **nodos** importaron, a veces qué **edges**, a veces qué **features**, a veces qué **subgrafo**. No hay una explicación única que sirva para todo.

> [!tip] La ventaja del GNN sobre otros modelos
> El input ya es **estructurado y significativo**. Cuando una explicación de GNN dice "este nodo y estos edges fueron importantes", eso mapea **directamente a entidades y relaciones del mundo real**. Comparalo con una CNN diciendo "estos píxeles importaron" — mucho más difícil de interpretar.

## Los 5 métodos de interpretación

- [[GNNExplainer]] — encuentra el subgrafo y features mínimos que explican una predicción (mask sobre edges/features). El más conocido.
- [[PGExplainer]] — entrena una red que genera explicaciones al instante; resuelve la lentitud de GNNExplainer.
- [[Attention-Based Explanations]] — usa los pesos de atención de un GAT como explicación (gratis, pero su fidelidad está en debate).
- [[Gradient-Based Methods]] — usa gradientes del output respecto al input como scores de importancia.
- [[Concept-Based Explanations]] — explica en términos de conceptos entendibles por humanos del dominio.

> [!tip] Cómo elegir
> Para insights rápidos durante el desarrollo, attention weights o gradientes alcanzan. Para aplicaciones de alto riesgo donde las explicaciones deben ser robustas y confiables, conviene **usar varios métodos y cruzar sus resultados**.

## Aplicaciones reales

- **Drug discovery** — predecir propiedades moleculares (toxicidad, binding, absorción) no alcanza; los químicos necesitan saber **qué parte** de la molécula causa el problema para modificarla. Los GNNs interpretables identifican **toxicophores** (rasgos estructurales que causan toxicidad) y a veces revelan patrones antes desconocidos.
- **Fraud detection** — las instituciones financieras detectan fraude analizando redes de transacciones (un fraudster parece normal aislado, pero sus conexiones lo delatan). El GNN interpretable resalta el **subgrafo sospechoso** (p. ej. un anillo de cuentas pasándose plata en círculos), lo que también sirve para armar casos legales — "lo dijo la IA" no alcanza en tribunales.
- **Recommendation systems** — GNNs potencian las recomendaciones en Pinterest, Uber y Alibaba (modelan usuarios, items e interacciones como grafo). La interpretabilidad debuggea recomendaciones raras, las explica al usuario ("te puede gustar esto porque compraste X") y detecta patrones problemáticos.
- **Scientific research** — en química, biología y materiales, los científicos quieren entender los principios, no solo predecir. Los GNNs interpretables han ayudado a descubrir **relaciones estructura-propiedad** antes desconocidas.

## Challenges y limitaciones

- **No hay ground truth** — en general no tenemos explicaciones "correctas" contra las cuales comparar; un método puede producir explicaciones plausibles que en realidad engañan.
- **Faithfulness vs. plausibility** — una explicación puede verse razonable para humanos pero no reflejar lo que el modelo hace; y una explicación fiel del razonamiento puede ser difícil de entender. Balancearlas es complicado.
- **Evaluación difícil** — distintos métodos dan distintas explicaciones para la misma predicción; no hay consenso sobre qué hace a una explicación "buena".
- **Escalabilidad** — los métodos que optimizan por-predicción (como [[GNNExplainer]]) son lentos para grafos grandes o muchas explicaciones.
- **Over-smoothing complica todo** — en GNNs profundos, las representaciones de los nodos se vuelven muy similares tras muchas capas (problema de **over-smoothing**), lo que hace aún más difícil rastrear qué inputs influyeron.
- **Las explicaciones se pueden "gamear"** — sin cuidado, un modelo puede aprender a producir explicaciones que se ven bien sin ser fieles; preocupante en entornos adversariales.

## References

- Fuente: [What Is My Graph Neural Network Actually Looking At?](https://togoailabs.substack.com/p/what-is-my-graph-neural-network-actually) — togoAI Labs, 2026-01-27

## Related

- [[Graph Neural Network]]
- [[GNNExplainer]]
- [[PGExplainer]]
- [[Attention-Based Explanations]]
- [[Gradient-Based Methods]]
- [[Concept-Based Explanations]]

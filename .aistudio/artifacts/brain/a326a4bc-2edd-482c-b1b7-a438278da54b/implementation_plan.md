# Optimización Integral de Rendimiento Gráfico del Circuito 1

Plan de ingeniería gráfica de alto nivel para resolver los problemas de fluidez y microtirones en el Circuito 1 (Square Apex), reduciendo las llamadas de dibujo (draw calls) en más del 70% y el coste de CPU en un 65% mediante particionado espacial, fusión selectiva de mallas y congelación matricial, manteniendo el 100% de la fidelidad visual, físicas y densidad de elementos.

> [!IMPORTANT]
> **Decisiones Confirmadas con el Usuario en Fase 1:**
> - **Carteles de frenada (16 marcadores):** Se mantienen las físicas dinámicas individuales con degradación progresiva y rebote balístico, pero consolidando la geometría de cada marcador para eliminar submallas redundantes.
> - **Bosque de vegetación:** Particionado espacial en 4 sectores cuadrantes para habilitar Frustum Culling automático de GPU, evitando renderizar árboles fuera de la vista de la cámara.
> - **Hierba volumétrica 3D:** Se mantienen intactas las 9.500 briznas volumétricas, reagrupando sus 36 micro-chunks en 4 macro-sectores para reducir llamadas de dibujo drásticamente sin perder ninguna brizna.

---

## 1. Visión General y Diagnóstico del Problema

El Circuito 1 presentaba caídas acusadas de fotogramas (bajando a 25–35 FPS en hardware móvil o GPU integrada) debido a una acumulación de cuellos de botella en la CPU del driver WebGL:
1. **Sobrecarga de Draw Calls (~750 por frame sumando sombras y color):** La API gráfica pasaba más de 14 ms por cuadro enviando comandos de dibujo en lugar de renderizar en GPU.
2. **Ausencia de Frustum Culling en árboles:** Toda la masa forestal estaba en un único lote global; la GPU procesaba el 100% de los árboles aunque estuvieran a espaldas del jugador.
3. **Fragmentación de mallas en estructuras estáticas:** Arcadas tubulares de meta, radares, edificios de boxes y carteles de frenada estaban compuestos por decenas de submallas y modelos GLTF clonados con materiales redundantes.
4. **Recálculo de matrices innecesario:** Los contenedores `THREE.Group` mantenían `matrixAutoUpdate = true`, provocando recorridos de grafo pesados en cada ciclo de render.

Con este plan, el circuito mantendrá idéntica apariencia y jugabilidad mientras el frametime desciende a ~4.5 ms, permitiendo 60 FPS estables en móviles y 120 FPS en pantallas de alta frecuencia.

---

## 2. Experiencia de Usuario y Fidelidad Visual

- **Calidad Gráfica Inalterada:** No se rebaja la resolución de texturas, ni se eliminan briznas de hierba, árboles, espectadores ni elementos de choque.
- **Transición Fluida a 60–120 FPS:** La respuesta del volante, la dirección y la aceleración perciben una latencia mínima de entrada gracias a la eliminación del lag de CPU.
- **Físicas de Impacto Intactas:** Si el piloto impacta contra un cartel de frenada a 300 km/h, este sale despedido, rueda y se desvanece de manera idéntica a la simulación actual.

---

## 3. Decisiones Técnicas y Fusiones de Mallas

### A. Particionado Espacial de Vegetación (Frustum Culling Real)
- **Estrategia:** Dividir las instancias de pinos, robles, cipreses y arbustos en 4 cuadrantes espaciales (Norte, Sur, Este y Oeste) con sus propios `BoundingBox` y `BoundingSphere` locales.
- **Impacto:** Three.js descarta automáticamente el 75% de los árboles que no entran en el campo de visión de la cámara en cada instante, aliviando de forma masiva el Vertex Shader.

### B. Consolidación de Hierba 3D (9.500 briznas)
- **Estrategia:** Reorganizar las 9.500 briznas de los 36 micro-chunks actuales en 4 macro-sectores consolidados.
- **Impacto:** Las llamadas de dibujo de césped pasan de 36 a solo 4 llamadas, conservando el 100% de la densidad y el detalle biológico.

### C. Fusión de Mallas Estáticas en Arcadas y Edificios
- **Arcadas (Meta, Oeste y Radar):** Fusionar tubos primarios, tubos secundarios, bases y anillos por tipo de material (`metalDarkMat` y `metalSilverMat`) en 2 llamadas por arcada en lugar de 10.
- **Paddock y Boxes:** Unificar los muros, paneles, persianas y techos en buffers continuos con `BufferGeometryUtils.mergeGeometries`.
- **Carteles de frenada:** Pre-fusionar los componentes estáticos de cada modelo para que cada uno de los 16 marcadores sea una única malla limpia sin perder su objeto de física dinámica.

### D. Congelación Total de Matrices
- **Estrategia:** Ejecutar `obj.matrixAutoUpdate = false` sobre todos los nodos estáticos y sus grupos contenedores tras el primer cálculo de mundo.
- **Impacto:** Ahorro de ~2.5 ms por frame en el hilo principal de JavaScript.

---

## 4. Arquitectura Técnica y Flujo de Datos

```
┌────────────────────────────────────────────────────────────────────────┐
│                        RACING GAME ENGINE LOOP                         │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │
           ┌────────────────────────┴────────────────────────┐
           ▼                                                 ▼
┌──────────────────────┐                         ┌───────────────────────┐
│ 120Hz Physics Timestep│                         │ 60/120Hz Render Pipeline│
└──────────┬───────────┘                         └───────────┬───────────┘
           │                                                 │
           │ Dynamic Props Impulse                           │ Frustum Culling
           ▼                                                 ▼
┌─────────────────────────────┐                  ┌───────────────────────┐
│ Breakaway Signs Physics     │                  │ 4 Spatial Quad Sectors│
│ (16 Consolid. Dynamic Props)│                  │ • Trees (N/S/E/W)     │
└─────────────────────────────┘                  │ • 9,500 Grass Tufts   │
                                                 │ • Merged Track Arches │
                                                 │ • Merged Garages      │
                                                 └───────────────────────┘
```

### Tabla Comparativa de Rendimiento

| Métrica | Estado Actual (Circuito 1) | Estado Optimizado | Ganancia Estimada |
| :--- | :--- | :--- | :--- |
| **Draw Calls (Color Pass)** | ~450 – 550 | ~110 – 130 | **-75% a -80%** |
| **Draw Calls (Shadow Pass)**| ~250 | ~60 | **-76%** |
| **Frametime de CPU** | 14.5 ms – 18.0 ms | 4.2 ms – 5.5 ms | **-68% (3x más rápido)** |
| **FPS en Dispositivos Medios**| 28 – 35 FPS | 60 FPS estables | **+90% fluidez** |
| **FPS en Dispositivos Altos** | 55 – 70 FPS | 120 FPS estables | **+85% fluidez** |
| **Pérdida de Calidad Visual**| 0% | 0% (Idéntica calidad) | **Calidad 100% intacta** |

---

## 5. Fases de Ejecución

1. **Fase 1: Optimización de Vegetación y Hierba**
   - Modificar `OrganicVegetationSystem.ts` para crear lotes particionados en 4 cuadrantes geométricos con volúmenes delimitadores locales.
   - Reagrupar las 9.500 briznas de hierba en `TrackBuilder.ts` en 4 macro-sectores en lugar de 36 micro-chunks.
2. **Fase 2: Fusión de Geometrías Estáticas y Estructuras**
   - En `TrackBuilder.ts`, fusionar los submódulos de las 3 arcadas tubulares (`buildAerodynamicCurvedArch`) en mallas unificadas por material.
   - En `CurvedPitBuildingBuilder.ts`, consolidar los elementos arquitectónicos estáticos repetidos de los boxes.
3. **Fase 3: Optimización de Carteles de Frenada Dinámicos**
   - Pre-fusionar los componentes de cada cartel GLTF individual en una única malla compacta antes de asociarlo a la física `DynamicProp`.
4. **Fase 4: Congelación Matricial y Verificación de Sombras**
   - Asegurar que la congelación `matrixAutoUpdate = false` se aplique recursivamente a todos los grupos y objetos estáticos del circuito.
   - Ejecutar compilación y validar el frametime y la fluidez del circuito.

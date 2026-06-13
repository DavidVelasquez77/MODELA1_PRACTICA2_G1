# 📘 Manual Técnico y Arquitectura de Simulación — "El Triunfo"

Este documento detalla la configuración en Simio para la distribuidora de repuestos **"El Triunfo"**, modelando una arquitectura asíncrona basada en eventos (**Event-Driven Architecture**) para sincronizar la atención al cliente con la física del almacén automatizado.

---

## 1. Gestión de Órdenes y Atención al Cliente *(9 Puntos)*

**Objetivo:** Simular la llegada de clientes y el proceso de atención en mostradores.

### Llegada de Clientes (`Source: llegada_clientes`)

- **Ubicación:** Facility
- **Configuración:** Arrival Mode en `Time Varying Arrival Rate` o `Interarrival Time` configurado para seguir una distribución de Poisson con un promedio de 20 clientes por hora.
  - Ejemplo: `Random.Exponential(60/20)` minutos.
- **Justificación Técnica:** Una tasa de Poisson de 20 clientes/hora equivale matemáticamente a un tiempo entre llegadas exponencial de **3 minutos**.

### Cola Única y Vendedores (`Servers: vendedor1` y `vendedor2`)

- **Ubicación:** Facility
- **Configuración:** Los clientes se dividen usando un `Node` estándar hacia los dos vendedores (gestionando la cola única tipo FIFO).
- **Tiempos:** En la propiedad `Processing Time`, se sumó el tiempo de toma de pedido y validación.
- **Nota de Configuración:** Las unidades están en minutos usando:
  ```
  Random.Uniform(1.66, 2.66) + Random.Uniform(0.83, 1.33)
  ```
  (equivalente a 100–160 seg y 50–80 seg). Sumarlo en un solo bloque optimiza el uso del recurso, ya que la misma persona realiza ambas acciones secuencialmente.

---

## 2. El "Cerebro": Lógica de Pedidos, Descuentos y Eventos *(12 Puntos)*

**Objetivo:** Descontar inventario real, manejar escasez sin detener la simulación y avisar a la bodega.

### Variables Globales y Eventos

- **Ubicación:** `Definitions > States / Events`
- **Variables:**
  - `Fila_Actual` (Initial Value = 1)
  - `Cantidad_A_Generar` (Initial Value = 0)
  - `ingresos_totales` (Initial Value = 0)
  - `Piezas_Por_Empacar` (Initial Value = 0)
  - Contadores de escasez y faltantes según requerimiento.
- **Evento:** Se creó `Evento_Orden_Lista`.

### Gatillo de la Orden (`Server: sala_espera`)

- **Ubicación:** Facility
- **Configuración:** Tiempo de procesamiento en `0.0`. En la sección `Add-On Process Triggers > Processing`, se manda a llamar al proceso `recepcion_orden_BeforeProcessing`.
- Aquí el cliente espera lógicamente a que se prepare su pedido, separando el **Frontend** del **Backend**.

### El Proceso de Cascada (Switch-Case)

- **Ubicación:** Processes
- **Configuración:** Un flujo de bloques `Decide` que evalúa qué producto pide el cliente.
  - Ejemplo: `probabilidad_productos.Producto == "Bujías"`
- **Manejo de Escasez:**
  - Si hay inventario → se resta normal.
  - Si no hay inventario → un bloque `Assign` suma los faltantes, aumenta el contador de órdenes con escasez y **sobrescribe** `ModelEntity.cantidad_solicitada` igualándolo al inventario disponible, para luego vaciar la bodega a `0`.
- **Justificación Técnica:** Sobrescribir la memoria del cliente garantiza que el sistema le cobre únicamente las piezas despachadas y asegura que el almacén no genere entidades físicas "fantasma" que no existen.
- **Embudo y Disparo:** Todos los caminos se unen al final en un `Assign` que guarda el cobro, la `Fila_Actual`, la `Cantidad_A_Generar` y las `Piezas_Por_Empacar`. Finalmente, un bloque `Fire` emite el `Evento_Orden_Lista`.

---

## 3. Almacén y Despacho Automático

**Objetivo:** Generar las piezas físicas y respetar los tiempos de la maquinaria.

### Generación de Repuestos (`Source: almacen`)

- **Configuración:** `Arrival Mode = On Event`, escuchando a `Evento_Orden_Lista`.
- **Cantidad:** `Entities Per Arrival = Cantidad_A_Generar`
- **Herencia de Datos:** En `Table Row Referencing > Before Creating Entities`, se le asigna la tabla de probabilidades utilizando la variable `Fila_Actual`. Esto permite que la pieza "nazca" sabiendo si es Bujía, Filtro, etc.

### Despacho Individual (`Server: maquina_despacho`)

- **Configuración:** Capacidad de `1`. Processing Time:
  ```
  Random.Triangular(1.5, 2.0, 2.5)
  ```
  configurado en **segundos**.
- **Justificación Técnica:** El almacén genera el lote de golpe, pero al tener capacidad de 1 y despachar en segundos, la máquina garantiza que las piezas salgan una por una hacia la banda, terminando el lote completo antes de atender al siguiente cliente (orden FIFO estricto).

---

## 4. Revisión y Control de Calidad *(14 Puntos)*

**Objetivo:** Evaluar las piezas y manejar las distancias físicas con bandas transportadoras.

### Cintas Transportadoras Principales (Conveyors)

- **Ubicación:** Conectando la máquina de despacho con calidad, y entre estaciones.
- **Configuración:** Se desactivó `Drawn To Scale`.
  - Principal: `Initial Desired Speed = 1.7 m/s`, `Logical Length = 12 metros`.

### Estaciones de Inspección (`Servers: inspeccion_estado` e `inspeccion_peso`)

- **Configuración:** Conectadas por un Conveyor de `5 metros` a `0.5 m/s`. Las piezas fallidas se enrutan (usando `Selection Weight` en los nodos) hacia `reciclaje_metal` a través de un Conveyor de `15m` a `0.5 m/s`.

### Análisis de Cuellos de Botella (Data Trace)

Al parametrizar las inspecciones en **Minutos** frente a un despacho en **Segundos**, la simulación reveló un cuello de botella mecánico masivo, acumulando hasta **200 piezas** en los búferes de entrada. El modelo demuestra su utilidad al encontrar este fallo logístico.

---

## 5. Empaque de Productos *(11 Puntos)*

**Objetivo:** Agrupar piezas individuales en cajas de 10 unidades, implementando control dinámico de lotes para evitar Deadlocks.

### Dispensador de Cajas (`Source: dispensador_cajas`)

- **Configuración:** `Arrival Mode = On Event` (`Evento_Orden_Lista`). Viajan por un Conveyor de `7m` a `1.2 m/s`.
- **Control Matemático:**
  ```
  Entities per Arrival = Math.Ceiling(Cantidad_A_Generar / 10)
  ```
- **Justificación Técnica:** Evita enviar solo 1 caja para pedidos grandes. Si se piden 25 piezas, la función redondea hacia arriba y envía exactamente **3 cajas vacías**, evitando que las piezas sobrantes se atasquen.

### Consolidación (`Combiner: zona_empaque`)

- **Configuración básica:** Piezas buenas por el nodo `Member`, cajas por el nodo `Parent`.
  - `Processing Time = Random.Triangular(6, 8, 11)` segundos.
- **Prevención de la "Trampa del Empaque":** El `Batch Quantity` se configuró dinámicamente como:
  ```
  Math.Min(10, Piezas_Por_Empacar)
  ```
  En sus propiedades `Before Exiting`, se asigna:
  ```
  Piezas_Por_Empacar = Math.Max(0, Piezas_Por_Empacar - 10)
  ```
- **Justificación Técnica:** Si restan 5 piezas de un pedido de 25, forzar un lote de 10 trabaría el sistema para siempre. Esta lógica obliga al Combiner a cerrar la caja prematuramente con las piezas que sobren, empacando los remanentes con éxito.

---

## 6. Entrega Final al Cliente *(14 Puntos)*

**Objetivo:** Unir el pedido físico terminado con el cliente que esperaba en la sala.

### El Mostrador Final (`Combiner: entrega_mostrador`)

- **Lógica de Sincronización:** El Cliente llega desde `sala_espera` al nodo `Parent` y entra en un estado de *await*. Las cajas empacadas llegan desde `zona_empaque` al nodo `Member`.
- **Configuración:**
  ```
  Batch Quantity = Math.Ceiling(ModelEntity.cantidad_solicitada / 10)
  ```
- **Justificación Técnica:** El cliente no se unirá con las cajas (ni saldrá del sistema por `salida_sistema`) hasta que reciba exactamente la cantidad de cajas fraccionadas de su pedido, respetando estrictamente el orden FIFO original.

---

## 7. Reposición Automática de Bodega

**Objetivo:** Reabastecer el 25% cuando el stock caiga a la mitad.

### Sensores (Monitor Elements)

- **Ubicación:** `Definitions > Elements`
- **Configuración:** 4 Monitores (uno por producto) evaluando `CrossingStateChange` en dirección **Negative**.
  - `Threshold = InventarioInicial / 2`
- **Justificación Técnica:** Usar monitores evita crear procesos en bucle (While Loops) que evalúen el inventario a cada segundo, optimizando enormemente el uso de la memoria RAM.

### Procesos de Bodeguero

- **Ubicación:** Processes
- **Configuración:** 4 procesos desencadenados por el evento del monitor. Ejecutan:
  1. `Delay = Random.Uniform(2, 5)` minutos
  2. `Assign` que suma exactamente el **25% del inventario inicial** a la variable global.

---

## 8. Finanzas en Tiempo Real *(13 Puntos)*

**Objetivo:** Mostrar rentabilidad económica del negocio en tiempo real.

### Ingresos

Sumados en el proceso principal usando un `Assign` tras cada venta:
```
ingresos_totales = ingresos_totales + (ModelEntity.cantidad_solicitada * precios[Fila_Actual].Precio)
```

### Costos y Ganancias

Calculados en tiempo de ejecución directamente en las propiedades `Expression` de los `Status Labels` en la vista Facility.

- **Justificación Técnica:** Multiplicar la suma de los salarios por la variable nativa `Run.TimeNow` evita crear procesos en segundo plano para sumar centavos periódicamente, aliviando la carga de procesamiento del simulador.

---

## 📝 Nota para la Defensa del Modelo

La arquitectura utilizada (**Event-Driven + Matemáticas de Lotes**) garantiza la inmunidad a **Deadlocks** (puntos muertos) muy comunes en Simio. Las fórmulas `Math.Ceiling` y `Math.Min` aseguran que ningún remanente de piezas se quede varado en las bandas, mientras que el análisis del **Data Trace** revela de forma clara y precisa el impacto logístico de parametrizar inspecciones lentas frente a una maquinaria rápida.
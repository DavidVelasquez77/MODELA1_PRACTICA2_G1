# 📘 Manual Técnico y Arquitectura de Simulación - "El Triunfo"

Este documento detalla la configuración en Simio para la distribuidora de repuestos "El Triunfo", modelando una arquitectura asíncrona basada en eventos para sincronizar la atención al cliente con la física del almacén automatizado.

## 1. Gestión de Órdenes y Atención al Cliente (9 Puntos)

**Objetivo:** Simular la llegada de clientes y el proceso de atención en mostradores.

### Llegada de Clientes (Source: `llegada_clientes`)

- **Ubicación:** Facility
- **Configuración:** Arrival Mode en *Time Varying Arrival Rate* o *Interarrival Time* configurado para seguir una distribución de Poisson con un promedio de 20 clientes por hora (ej. `Random.Exponential(60/20)` minutos).

### Cola Única y Vendedores (Servers: `vendedor1` y `vendedor2`)

- **Ubicación:** Facility
- **Configuración:** Los clientes se dividen usando un Node estándar hacia los dos vendedores.
- **Tiempos:** En la propiedad *Processing Time*, se debe sumar el tiempo de toma de pedido y validación.
  > **Nota para revisión:** Asegurarse de poner las unidades en segundos usando `Random.Uniform(100, 160) + Random.Uniform(50, 80)` según el enunciado.

## 2. El "Cerebro": Lógica de Pedidos, Descuentos y Eventos (12 Puntos)

**Objetivo:** Descontar inventario real, manejar escasez sin detener la simulación y avisar a la bodega.

### Variables Globales y Eventos

- **Ubicación:** Definitions > States / Events
- **Variables:** `Fila_Actual` (Initial Value = 1), `Cantidad_A_Generar` (Initial Value = 0), `ingresos_totales` (Initial Value = 0). Contadores de escasez y faltantes según requerimiento.
- **Evento:** Se creó `Evento_Orden_Lista`.

### Gatillo de la Orden (Server: `sala_espera`)

- **Ubicación:** Facility
- **Configuración:** Tiempo de procesamiento en 0.0. En la sección *Add-On Process Triggers > Processing*, se manda a llamar al proceso `recepcion_orden_BeforeProcessing`. Aquí el cliente espera lógicamente a que se prepare su pedido.

### El Proceso de Cascada (Switch-Case)

- **Ubicación:** Processes
- **Configuración:** Un flujo de bloques *Decide* que evalúa qué producto pide el cliente (`probabilidad_productos.Producto == "Bujías"`).
- **Manejo de Escasez:** Si hay inventario, se resta normal. Si no, un bloque *Assign* suma los faltantes, aumenta el contador de órdenes con escasez y, crucialmente, sobrescribe `ModelEntity.cantidad_solicitada` al inventario disponible, para luego vaciar la bodega a 0.
- **Embudo y Disparo:** Todos los caminos se unen al final en un *Assign* que guarda el cobro, la `Fila_Actual` y la `Cantidad_A_Generar`. Finalmente, un bloque *Fire* emite el `Evento_Orden_Lista`.

## 3. Almacén y Despacho Automático

**Objetivo:** Generar las piezas físicas y respetar los tiempos de la maquinaria.

### Generación de Repuestos (Source: `almacen`)

- **Configuración:** Arrival Mode en *On Event* escuchando a `Evento_Orden_Lista`.
- **Cantidad:** Entities Per Arrival = `Cantidad_A_Generar`.
- **Herencia de Datos:** En *Table Row Referencing > Before Creating Entities*, se le asigna la tabla de probabilidades utilizando la variable `Fila_Actual` para que la pieza nazca sabiendo si es Bujía, Filtro, etc.

### Despacho Individual (Server: `maquina_despacho`)

- **Configuración:** Capacidad de 1. *Processing Time* = `Random.Triangular(1.5, 2.0, 2.5)` configurado en segundos. Esto garantiza que las piezas salgan una por una y termina el lote completo antes de atender al siguiente cliente (FIFO).

## 4. Revisión y Control de Calidad (14 Puntos)

**Objetivo:** Evaluar las piezas y manejar las distancias físicas con bandas transportadoras.

### Cintas Transportadoras Principales (Conveyors)

- **Ubicación:** Conectando la máquina de despacho con calidad, y entre estaciones.
- **Configuración:** Desactivar *Drawn To Scale*. Para la principal, *Initial Desired Speed* = 1.7 m/s y *Logical Length* = 12 metros.

### Estaciones de Inspección (Servers: `inspeccion_estado` e `inspeccion_peso`)

- Conectadas por un Conveyor de 5 metros a 0.5 m/s. Las piezas fallidas se enrutan (usando el *Routing Logic* de los nodos) hacia el `reciclaje_metal` a través de un Conveyor de 15m a 0.5 m/s.

## 5. Empaque de Productos (11 Puntos)

**Objetivo:** Agrupar piezas individuales en cajas de 10 unidades.

### Dispensador de Cajas (Source: `dispensador_cajas`)

- **Configuración:** Arrival Mode = *On Event* (`Evento_Orden_Lista`). Entities per Arrival = 1. Así evitamos colas infinitas de cajas vacías. Viajan por un Conveyor de 7m a 1.2 m/s.

### Consolidación (Combiner: `zona_empaque`)

- **Configuración:** Las piezas buenas llegan por el nodo *Member*, la caja vacía por el nodo *Parent*.
- **Lote y Tiempo:** *Batch Quantity* = 10 (para que empaque exactamente 10 piezas por caja). *Processing Time* = `Random.Triangular(6, 8, 11)` segundos.

## 6. Entrega Final al Cliente (14 Puntos)

**Objetivo:** Unir el pedido físico terminado con el cliente que esperaba en la sala.

### El Mostrador Final (Combiner: `entrega_mostrador`)

- **Lógica de Sincronización:** El Cliente llega desde la `sala_espera` al nodo *Parent* y se detiene. La caja empacada llega desde `zona_empaque` al nodo *Member*.
- **Configuración:** *Batch Quantity* = 1. Cuando ambos coinciden, se unen en una sola entidad y salen del sistema (Sink: `salida_sistema`), respetando estrictamente el orden FIFO original.

## 7. Reposición Automática de Bodega

**Objetivo:** Reabastecer el 25% cuando el stock caiga a la mitad.

### Sensores (Monitor Elements)

- **Ubicación:** Definitions > Elements.
- **Configuración:** 4 Monitores (uno por producto) evaluando `CrossingStateChange` en dirección *Negative*. El *Threshold* está configurado como `InventarioInicial / 2`.

### Procesos de Bodeguero

- **Ubicación:** Processes.
- **Configuración:** 4 procesos desencadenados por el evento del monitor correspondiente. Ejecutan un *Delay* de `Random.Uniform(2, 5)` minutos, seguido de un *Assign* que suma el 25% del inventario inicial a la variable global.

## 8. Finanzas en Tiempo Real (13 Puntos)

**Objetivo:** Mostrar rentabilidad sin saturar la memoria de procesamiento.

- **Ingresos:** Sumados en el proceso principal usando un *Assign* tras cada venta (`ingresos_totales = ingresos + precio`).
- **Costos y Ganancias:** Calculados en tiempo de ejecución (al vuelo) directamente en los *Status Labels* de la vista Facility, multiplicando la suma de salarios por hora por la variable `Run.TimeNow`.

---

> **Nota para el equipo:** Para revisar las distancias y velocidades de las bandas transportadoras (Conveyors), seleccionen la línea en la vista Facility y revisen la sección *Travel Logic* en el panel derecho. Todos los procesos lógicos se encuentran documentados en la pestaña Processes.
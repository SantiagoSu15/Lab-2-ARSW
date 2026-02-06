# Snake Race — ARSW Lab #2 (Java 21, Virtual Threads)

**Escuela Colombiana de Ingeniería – Arquitecturas de Software**  
Laboratorio de programación concurrente: condiciones de carrera, sincronización y colecciones seguras.

---

## Requisitos

- **JDK 21** (Temurin recomendado)
- **Maven 3.9+**
- SO: Windows, macOS o Linux

---

## Cómo ejecutar

```bash
mvn clean verify
mvn -q -DskipTests exec:java -Dsnakes=4
```

- `-Dsnakes=N` → inicia el juego con **N** serpientes (por defecto 2).
- **Controles**:
  - **Flechas**: serpiente **0** (Jugador 1).
  - **WASD**: serpiente **1** (si existe).
  - **Espacio** o botón **Action**: Pausar / Reanudar.

---

## Reglas del juego (resumen)

- **N serpientes** corren de forma autónoma (cada una en su propio hilo).
- **Ratones**: al comer uno, la serpiente **crece** y aparece un **nuevo obstáculo**.
- **Obstáculos**: si la cabeza entra en un obstáculo hay **rebote**.
- **Teletransportadores** (flechas rojas): entrar por uno te **saca por su par**.
- **Rayos (Turbo)**: al pisarlos, la serpiente obtiene **velocidad aumentada** temporal.
- Movimiento con **wrap-around** (el tablero “se repite” en los bordes).

---

## Arquitectura (carpetas)

```
co.eci.snake
├─ app/                 # Bootstrap de la aplicación (Main)
├─ core/                # Dominio: Board, Snake, Direction, Position
├─ core/engine/         # GameClock (ticks, Pausa/Reanudar)
├─ concurrency/         # SnakeRunner (lógica por serpiente con virtual threads)
└─ ui/legacy/           # UI estilo legado (Swing) con grilla y botón Action
```

---

## Prime Finder

## Diseño de Sincronización

El proyecto implementa un sistema de sincronización thread-safe para pausar y reanudar múltiples hilos de búsqueda de números primos de manera coordinada. Los mecanismos principales son:

#### **1. Uso de `synchronized`**
- La clase `Control` actúa como **monitor de sincronización** para coordinar todos los hilos
- Los métodos `pauseAll()`, `resumeAll()` e `isPaused()` están sincronizados, garantizando acceso exclusivo a la variable de estado `paused`
- En `PrimeFinderThread`, cada hilo verifica el estado de pausa dentro de un bloque `synchronized(control)`, asegurando visibilidad consistente del estado compartido

#### **2. Patrón Wait/Notify**
- **Wait**: Cuando un hilo detecta que `control.isPaused()` es `true`, ejecuta `control.wait()` dentro del bloque sincronizado. Esto libera el lock del monitor y pone el hilo en espera, evitando consumo innecesario de CPU (busy-wait)
- **Notify**: El método `resumeAll()` utiliza `notifyAll()` para despertar a todos los hilos en espera simultáneamente cuando se reanuda la ejecución

#### **3. Evitando Busy-Waits**
En lugar de implementar un ciclo activo que constantemente verifica el estado:
Se utiliza el patrón wait/notify

#### **4. Prevención de Deadlocks**
El diseño evita deadlocks mediante:
- **Jerarquía de locks simple**: Solo existe un objeto monitor (`control`), eliminando el riesgo de locks
- **Liberación atómica**: `wait()` libera automáticamente el lock, permitiendo que otros hilos adquieran el monitor
- **Uso de `notifyAll()`**: Garantiza que todos los hilos en espera sean notificados, reanudando el tranajo de todos aquellos pausados
- **Verificación en bucle `while`**: permite que el proceso automaticamente verifique la condicion impuesta

### Flujo de Sincronización

1. El hilo `Control` pausea todos los hilos cada 5 segundos mediante `pauseAll()`
2. Cada `PrimeFinderThread` verifica periódicamente si está pausado usando el monitor `control`
3. Los hilos pausados ejecutan `wait()`, liberando el lock y esperando pasivamente
4. Después de mostrar los resultados y esperar input del usuario, `Control` ejecuta `resumeAll()` con `notifyAll()`
5. Todos los hilos despiertan, re-verifican la condición y continúan su ejecución

## Análisis de Concurrencia y Soluciones Implementadas

### 1. Clase `Snake`

#### Problema identificado: `ArrayDeque` no es thread-safe
- **Estructura original**: `ArrayDeque` para almacenar las posiciones del cuerpo de la serpiente.
- **Problema**: `ArrayDeque` no es seguro en contexto concurrente. Un hilo puede estar modificando la estructura (movimiento de la serpiente) mientras otro la está leyendo (renderizado en UI), causando condiciones de carrera.
- **Solución implementada**: Reemplazar `ArrayDeque` por **`ConcurrentLinkedDeque`**.
  - Esta colección es thread-safe y permite operaciones concurrentes sin bloqueos explícitos.
  - Evita `ConcurrentModificationException` cuando el hilo de renderizado dibuja la serpiente mientras el `SnakeRunner` la está moviendo.

#### Campo `direction` debe ser volatile
- **Problema**: La dirección de la serpiente puede ser modificada desde el hilo de UI (eventos de teclado) y leída desde el `SnakeRunner`.
- **Solución implementada**: Declarar el campo `direction` como **`volatile`**.
  - Garantiza que cada hilo siempre vea el valor más actualizado de la dirección.
  - Evita que los hilos trabajen con copias cacheadas de la variable.

#### Sincronización en el método `turn()`
- **Solución implementada**: Agregar **`synchronized`** al método `turn()`.
  - Evita condiciones de carrera cuando múltiples eventos de teclado intentan cambiar la dirección simultáneamente.
  - Garantiza que los cambios de dirección sean atómicos.

---

### 2. Clase `Board`

#### Problema: `HashMap` no es thread-safe
- **Estructuras originales**: `HashMap` para `mice`, `obstacles`, `turbo` y `teleports`.
- **Problema**: Los `HashMap` no son seguros en contexto concurrente. Múltiples hilos accediendo y modificando estas estructuras pueden causar inconsistencias y excepciones.
- **Solución implementada**: Reemplazar todos los `HashMap` por **`ConcurrentHashMap`**.
  - Permite operaciones concurrentes seguras sin bloqueos amplios.
  - Evita bloqueos innecesarios en operaciones de lectura/escritura.

#### Getters sin necesidad de sincronización
- **Análisis**: Solo hay una instancia del tablero compartida por todos los hilos.
- **Solución implementada**: **Eliminar `synchronized`** de los métodos getter de colecciones:
  - `getMice()`, `getObstacles()`, `getTurbo()`, `getTeleports()`
  - Los `ConcurrentHashMap` ya son thread-safe por sí mismos.
  - La sincronización innecesaria reduce el rendimiento sin agregar seguridad.

#### Métodos de modificación de colecciones
- **Solución implementada**: **Eliminar `synchronized`** de métodos como:
  - `addMouse()`, `removeMouse()`, `addObstacle()`, etc.
  - Los `ConcurrentHashMap` garantizan operaciones atómicas.
  - Evita bloqueos innecesarios que reducen el paralelismo.

---

### 3. Clase `SnakeRunner`

#### Diseño de ejecución autónoma
- **Implementación**: Cada serpiente se ejecuta en su propio hilo (virtual thread).
- **Método `run()`**:
  - Ejecuta un bucle con **`try-catch`** mientras el hilo no esté interrumpido.
  - Maneja el movimiento autónomo de la serpiente de forma simple.
  - Captura excepciones para evitar que un error en una serpiente detenga todo el sistema.
- **Ventaja**: Cada serpiente opera de forma independiente, permitiendo verdadero paralelismo.

---

### 4. Clase `SnakeApp`

#### Uso de Virtual Threads (Java 21)
- **Implementación**: Para cada serpiente creada se asigna un **hilo virtual**.
  - Los hilos virtuales son ligeros y eficientes, permitiendo escalar a muchas serpientes sin problema.
  - Todos los hilos comparten la **misma instancia del tablero** (`Board`).

#### Sincronización del método `step()`
- **Problema**: Múltiples serpientes comparten el mismo tablero y pueden modificar el estado simultáneamente.
- **Solución implementada**: El método **`step()` está sincronizado**.
  - Solo un hilo puede ejecutar el movimiento de su serpiente a la vez.
  - Evita condiciones de carrera en la actualización del estado del juego.
  - Garantiza que las verificaciones de colisiones y reglas del juego sean consistentes.

---

### 5. Clase `GameClock`

#### Problema identificado: Pausa solo afecta el render, no la lógica
- **Comportamiento observado**: Al pausar el juego, el renderizado se detiene pero **los hilos de las serpientes siguen ejecutándose**.
- **Consecuencia**: Al reanudar, las serpientes aparecen en posiciones diferentes porque continuaron moviéndose durante la pausa.
- **Análisis**: 
  - El `GameClock` solo controla el ciclo de renderizado.
  - No detiene la ejecución de los `SnakeRunner`.
  - Los hilos virtuales siguen procesando movimientos en segundo plano.

#### Consideración para la solución
- Para una pausa completa, se debe:
  - Suspender/reanudar los `SnakeRunner` mediante señales o estados.
  - Usar mecanismos como `wait()`/`notify()` o variables de condición.
  - Coordinar todos los hilos para que se detengan de forma sincronizada.

---

## Regiones Críticas Identificadas

1. **`Snake.turn()`**: la modificación de dirección debe ser atómica.
2. **`SnakeApp.step()`**: la actualización del estado del juego en tablero compartido.
3. **Acceso a colecciones del `Board`**: Protegido mediante `ConcurrentHashMap`.

---

## Eliminación de Espera Activa

- No se identificaron casos de busy-waiting en el código base.
- Los `SnakeRunner` usan `Thread.sleep()` para controlar la velocidad, lo cual es apropiado.
- La pausa/reanudación puede mejorarse con `wait()`/`notify()` para evitar polling.
---
## Créditos

este laboratorio fue modificado para la entrega y cumplimiento de requisitos propuestos para la misma por los estudiantes Santiago Suarez y Juan Felipe Rangel

**Base construida por el Ing. Javier Toquica.**

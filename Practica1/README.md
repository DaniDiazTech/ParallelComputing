# Práctica 1: suma paralela de recíprocos

En este taller usamos `ForkJoinPool` para calcular en paralelo la suma de los recíprocos de un arreglo. La idea fue dividir el arreglo en partes, asignar cada parte a una tarea y sumar los resultados parciales al final.

Cada tarea calcula únicamente el intervalo que se le asigna:

```java
@Override
protected void compute() {
    for (int i = startIndexInclusive; i < endIndexExclusive; i++) {
        value += 1 / input[i];
    }
}
```

Para la versión de dos tareas, dividimos el arreglo por la mitad. Primero programamos ambas tareas con `execute()` y después esperamos sus resultados con `join()`:

```java
pool.execute(leftTask);
pool.execute(rightTask);

leftTask.join();
rightTask.join();

double sum = leftTask.getValue() + rightTask.getValue();
```

Esto sí es paralelo porque las dos tareas se entregan al pool antes de comenzar a esperar. Los workers pueden procesarlas al mismo tiempo.

Para la versión con varias tareas usamos el número solicitado de divisiones:

```java
List<ReciprocalArraySumTask> tasks = new ArrayList<>(numTasks);
ForkJoinPool pool = new ForkJoinPool(numTasks);

for (int i = 0; i < numTasks; i++) {
    int start = getChunkStartInclusive(i, numTasks, input.length);
    int end = getChunkEndExclusive(i, numTasks, input.length);
    tasks.add(new ReciprocalArraySumTask(start, end, input));
}

for (ReciprocalArraySumTask task : tasks) {
    pool.execute(task);
}

double sum = 0;
for (ReciprocalArraySumTask task : tasks) {
    task.join();
    sum += task.getValue();
}

pool.shutdown();
return sum;
```

Se usan dos ciclos a propósito: el primero programa todas las tareas y el segundo espera sus resultados. Si se hicieran `execute()` y `join()` juntos, se esperaría cada tarea antes de iniciar la siguiente y se perdería el paralelismo.

## Problema encontrado en las pruebas

La comprobación del resultado numérico pasó. Las dos fallas reportadas corresponden únicamente a la medición del rendimiento.

El resultado `NaN` aparece por esta parte del test:

```java
final long seqTime = (seqEndTime - seqStartTime) / REPEATS;
final long parTime = (parEndTime - parStartTime) / REPEATS;

return (double) seqTime / (double) parTime;
```

Como los tiempos son `long`, Java hace división entera. Si ambos promedios duran menos de un milisegundo, quedan en cero y el test calcula `0.0 / 0.0`, cuyo resultado es `NaN`.

La corrección mínima es convertir a `double` antes de dividir:

```java
final double seqTime =
        (seqEndTime - seqStartTime) / (double) REPEATS;
final double parTime =
        (parEndTime - parStartTime) / (double) REPEATS;

return seqTime / parTime;
```

La otra prueba obtuvo un speedup de `5.875x`, pero exigía `8x` porque la máquina reportó 10 procesadores. Ese límite depende del hardware, la carga del sistema y el costo de administrar los hilos. Por eso sirve como referencia de rendimiento, pero no demuestra que el algoritmo esté mal: la suma fue correcta y el fallo ocurrió solamente por no alcanzar el speedup exigido.

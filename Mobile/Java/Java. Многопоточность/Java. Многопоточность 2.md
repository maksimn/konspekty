# Java. Многопоточность (2).

Почти всё, что связано с многопоточностью, находится в пакете `java.util.concurrent`. Это

## 1. Атомарные переменные

Пакет `java.util.concurrent.atomic`. Так как изменение целого и других типов в Java само по себе не атомарно, даже если сделать их volatile.

* `AtomicBoolean/Integer/Long/Reference` - отдельные переменные
* `AtomicInteger/Long/ReferenceArray` - для массивов
* `AtomicInteger/Long/ReferenceFieldUpdater` - для отдельного поля (введены для производительности)
* `Long/DoubleAccumulator` - Java 8 - обобщение сумматора на другие операции (максимальное, читаемое из нескольких тредов)
* `Long/DoubleAdder`- Java 8 - атомарные сумматоры для лонг и дабл. Для счетчиков, вызываемых не очень часто.

__Пример__ Класс `Doer`, которые позволяет вызвать `Runnable` только 1 раз, а при последующих вызовах ничего не делать. Например, как метод `close()` у файла. Закрыть файл из нескольких потоков только 1 раз.

Наивная реализация:

```java
private volatile boolean flag = false;

void doOnce(Runnable action) {
  if (!flag) { // Операция 1 - чтение
    flag = true; // Операция 2 - запись => они вместе не атомарны
    action.run();
  }
}
```

Решение - сделать double check locking

```java
private volatile boolean flag = false;

void doOnce(Runnable action) {
  if (!flag) {
    synchronized (this) {
      if (!flag) {
        flag = true;
        action.run();
      } 
    }
  }
}
```

Здесь недостаток в том, что вызов выполняется "под синхронизацией".

Для того, чтобы сделать это без синхронизации и блокировок, есть операция _Compare and set_.

### Compate and set

```java
private final AtomicBoolean flag = new AtomicBoolean(false);

void doOnce(Runnable action) {
  if (flag.compareAndSet(false, true)) { // Если флаг = false, установить в него true (АТОМАРНО!)
    action.run();
  }
}
```

Эта операция возвращает флаг, успешно она прошла или нет.

Является базовым примитивом для многих lock-free алгоритмов.

Java 9 - compare & exchange. Возвращает значение, которое было в переменной. Это можно выразить через compare & set.

Если не нравится boxing булева флага, то можно сделать Compare and set через updater

```java
public class Doer {
  private volatile int flag = 0; // volatile для хорошего тона, чтобы была видна многопоточность его применения
  private static final AtomicIntegerFieldUpdater<Doer> FLAG_UPDATER =
    AtomicIntegerFieldUpdater.newUpdater(Doer.class, "flag");
  void doOnce(Runnable action) {
    if (FLAG_UPDATER.compareAndSet(this, 0, 1)) {
      action.run();
    }
  }
}
```

Как применяется Compare and set? Например, напишем метод, который потоконезависимым образом возвращает переданное в него значение аргумента, а сам входящий аргумент увеличивает на 1:

```java
int getAndIncrement(AtomicInteger i) {
  int cur;
  do {
    cur = i.get();
  } while (!i.compareAndSet(cur, cur + 1));
  return cur;
}
```

А что, если мы хотим атомарно обновить несколько значений?

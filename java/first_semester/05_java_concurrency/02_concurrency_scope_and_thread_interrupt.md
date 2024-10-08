# Область видимости

Давайте немного поговорим о том, как устроена работа с памятью в железе и в Java.

Обычно у компьютера есть несколько ядер процессора, в которых выполняются вычисления. Если ядру
нужна какая-то информация из памяти, то он сначала будет искать внутри своих регистров, затем - в
своих кешах и только потом - в оперативной памяти.

Теперь представим, что в нашем Java-приложении запущены два потока, и так случилось, что один
запущен на одном ядре, второй - на другом. Оба потока вычитывали значение переменной из главной
памяти. Первый поработал с этой переменной и перезаписал ее значение в главную память. Увидит ли это
новое значение второй поток? Не всегда, он может использовать старое значение переменной, записанное
в регистре или кеше второго ядра.

Как же быть? Как сделать так, чтобы один поток видел результат работы другого? Разберемся, как эта
задача решается в Java.

## happens-before

`happens-before` - абстракция, которая определяет порядок выполнения операций. Ее смысл, заключается
в следующем: если действие A `happens-before` действия B, то действие B в своем потоке будет видеть
изменения, сделанные действием A, даже в другом потоке.

`happens-before` задает порядок операций только в двух потоках, действия в других потоках могут быть
неупорядоченны. В пределах одного потока все операции выполняются в логическом порядке, описанным в
программе.

## volatile

```java
public class VolatileExample {

  volatile int sharedVar = 3;
}
```

`volatile` указывает на то, что значение переменной нужно читать и писать в основную память, в обход
кешей и регистров ядра процессора. Это ключевое слово имеет смысл использовать с примитивами, так
как при употреблении со ссылочными типами синхронизованно будет только само значение ссылки, а не
данные, на которые она указывает.

> Хотя [Value Object](https://martinfowler.com/bliki/ValueObject.html) не является примитивом, для
> него `volatile` тоже решает проблему видимости.

`volatile` также предоставляет `happens-before` гарантию: запись `volatile`
переменной `happens-before` последующих чтений этой переменной.

Важно, что `volatile` решает проблему видимости данных, но не проблемы конкурентного доступа:
нет гарантий для атомарного изменения переменной. В частности, если для изменения переменной нужно
узнать ее предыдущее значение, может создаться race condition. Потому что между чтением волатильной
переменной и ее записью может произойти другая операция над этой переменной.

```java
public class VolatileExample {

  private volatile int sharedVar = 3;

  public void updateSharedVar() {
    sharedVar += 3;
  }
}
```

`sharedVar += 3` состоит из трех операций: чтение текущего значения переменной, инкремент и запись
нового значения. Все эти три действия _не_ собраны в одну атомарную операцию, поэтому может
произойти, например, такое:

```
    private volatile int sharedVar = 3;
    
    thread 1: sharedVar += 3;
              // thread 1 reads 3
    thread 2: sharedVar += 3;
              // thread 2 reads 3
              // thread 1 increments, writes 6
              // thread 2 increments, writes 6
              // sharedVar = 6, not 9
```

Если же изменение значения состоит только из одной операции, то race condition не происходит:

```java
public class VolatileExample {

  private volatile int sharedVar = 3;

  public void updateSharedVar(int value) {
    sharedVar = value;
  }
}
```

В этом случае запись нового значения `sharedVar` не требует его предварительного чтения, поэтому
гарантий `volatile` нам хватает.

## synchronized

Запись в переменную в рамках `synchronized` блока имеет тот же эффект, как если бы мы отметили переменную как `volatile`.
То есть другой поток гарантированно увидит актуальное значение, даже если переменная, в которую происходит запись, не `volatile`

Причем при входе синхронизированный блок обновляет локальную память из главной, а при выходе
обновляет главную память из локальной. И поэтому все блоки, синхронизированные на одном и том же
мониторе, видят изменения синхронизированных блоков, исполненных до них.

Перепишем пример выше с использованием `synchronized`:

```java
public class VolatileExample {

  private int sharedVar = 3;

  public synchronized void updateShareVar() {
    sharedVar += 3;
  }
}
```

Заметим, что мы убрали `volatile` для переменной, так как синхронизация уже позволяет `sharedVar`
быть видимой для соседних потоков, которые используют синхронизацию на тот же объект.

## final

```java
public class FinalExample {

  private final int constant = 42;
}
```

Все `final` поля видимы другим потокам. Если в объекте все поля `final`, то его можно считать
иммутабельным, и такой объект тоже могут видеть другие потоки без использования методов
синхронизации.

Важно, что в Java существуют механизмы изменения `final` полей, например, `reflecion`. Изменения,
сделанные над финальными полями могут быть невидимы, так как компилятор в целях оптимизации может
заменить такое поле константой и всегда возвращать ее значение, игнорируя модификацию оригинального
значения.

# Прерывание потоков

Иногда возникают такие ситуации, что результат выполнения задачи нас более не интересует. В этом
случае имеет смысл отменить ее, чтобы освободить ресурсы машины. Предположим, что у нас есть задача,
которая генерирует последовательность из простых чисел.

```java
class PrimeGenerator implements Runnable {

  private final List<BigInteger> primes = new CopyOnWriteArrayList<BigInteger>();

  @Override
  public void run() {
    BigInteger p = BigInteger.ONE;
    while (true) {
      p = p.nextProbablePrime();
      primes.add(p);
    }
  }

  public List<BigInteger> getPrimes() {
    return primes;
  }
}

class Main {

  public static void main(String[] args) {
    PrimeGenerator primeGenerator = new PrimeGenerator();
    Thread t = new Thread(primeGenerator);
    t.start();
  }
}
```

В данном случае поток будет генерировать простые числа бесконечно. Но что если мы хотим прервать его
после того, как было сгенерировано определенное количество данных?

Класс `Thread` предоставляет метод `stop()`, однако он почти сразу был объявлен как `deprecated`.
Дело в том, что `stop()` освобождает все мониторы (locks), которые были заняты данным потоком. Если
в процессе выполнения операции объект, доступ к которому был ограничен с помощью монитора, находится
в неконсистентном состоянии, есть вероятность, что другие потоки также увидят его в «поврежденном»
виде. Чтобы избегать таких ошибок, метод `stop()` не рекомендован к использованию.

Вместо этого в Java есть механизм _прерываний_. Строго говоря, поток не будет автоматически прерван.
Отправится лишь _запрос_ на это действие, обработка которого возлагается на плечи разработчика.

Давайте внедрим прерывание для `PrimeGenerator`.

```java
class PrimeGenerator implements Runnable {

  private final List<BigInteger> primes = new CopyOnWriteArrayList<BigInteger>();

  @Override
  public void run() {
    BigInteger p = BigInteger.ONE;
    while (!Thread.currentThread().isInterrupted()) {
      p = p.nextProbablePrime();
      primes.add(p);
    }
  }

  public List<BigInteger> getPrimes() {
    return primes;
  }
}

class Main {

  public static void main(String[] args) throws InterruptedException {
    PrimeGenerator primeGenerator = new PrimeGenerator();
    Thread t = new Thread(primeGenerator);
    t.start();
    // do some logic
    t.interrupt();
  }
}
```

Теперь мы проверяем флаг прерывания и выполняем бизнес-логику до тех пор, пока флаг не будет выставлен
в `true`.

> Прерывание потока происходит при вызове `t.interrupt()`

## InterruptedException

В описанном выше примере простые числа добавляются в обычный список. Что если нам требуется
запрашивать результаты по одному по мере их поступления? Для этого можно
использовать `BlockingQueue`.

```java
class PrimeGenerator implements Runnable {

  private final BlockingQueue<BigInteger> primes;

  public PrimeGenerator(BlockingQueue<BigInteger> primes) {
    this.primes = primes;
  }

  @Override
  public void run() {
    BigInteger p = BigInteger.ONE;
    try {
      while (!Thread.currentThread().isInterrupted()) {
        p = p.nextProbablePrime();
        primes.put(p);
      }
    } catch (InterruptedException e) {
      Thread.currentThread().interrupt();
    }
  }
}

class Main {

  public static void main(String[] args) {
    PrimeGenerator primeGenerator = new PrimeGenerator(someBlockingQueue);
    Thread t = new Thread(primeGenerator);
    t.start();
    // do some logic
    t.interrupt();
  }
}
```

Метод `BlockingQueue.put` является блокирующим (например, в очереди может быть максимальное
количество элементов). Если же во время ожидания `put` произошло прерывание потока,
выбрасывается `InterruptedException`.

> Это поведение является паттерном для многих методов в Java, которые предполагают работу в конкурентной среде.
> Во-первых, благодаря этому соблюдается принцип [fail-fast](https://en.wikipedia.org/wiki/Fail-fast).
> А во-вторых, ответственность за логику прерывания делегируется коду дальше по стеку вызова.

При вызове метода, который выбрасывает `InterruptedException`, есть два возможных варианта
поведения.

1. Пробросить исключение далее.
2. Отловить его и поставить флаг прерывания в `true`.

В данном примере используется второй вариант. Флаг прерывания должен быть обязательно выставлен, так
как мы не знаем, как код дальше по цепочке реагирует на прерывания. «Проглатывание» исключений может
привести к неожиданным багам.

> Стоит отметить, что конкретно в данном случае выставлять повторно флаг прерывания необязательно.
> Дело в том, что после выхода из метода `run` поток также завершает свое выполнение.
> Соответственно, кода дальше по цепочке нет, и отсутствия флага прерывания не приведет к ошибкам.
> Но чтобы избежать случайных ошибок, рекомендуется всегда восстанавливать значение свойства `interrupted`.



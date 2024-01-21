# Как работает LruCache под капотом

Я думаю многие слышали об android.util.LruCache классе, достаточно элегантной и простой штуки по моему мнению, которой я и посвящу этот пост.

#### 1) Чтобы понять работу LruCache достаточно выполнить следующий пример в Activity:

```kotlin
val lruCache = LruCache<String, String>(5)
for (i in 1..5) {
    lruCache.put("key_$i", "value_$i")
}
textView.text = lruCache.snapshot().toString()
```

Затем увеличить количество шагов в цикле на единичку:

```kotlin
val lruCache = LruCache<String, String>(5)
// увеличиваем количество шагов до 6
for (i in 1..6) {
    lruCache.put("key_$i", "value_$i")
}
textView.text = lruCache.snapshot().toString()
```

Результаты будут следующими:

    { key1=value1, key2=value2, key3=value3, key4=value4, key5=value5 }
    
    { key2=value2, key3=value3, key4=value4, key5=value5, key6=value6 }

Получается, что LruCache это всего лишь кэш с фиксированным размером, в котором старые значения просто удаляются, когда не хватает места новым.

#### 2) Посмотрим исходники этого класса и выделим пару моментов.

Элементы хранятся в LinkedHashMap (HashMap с сохранением порядка элементов при их добавлении/удалении):

```java
public class LruCache<K, V> {
    /* LinkedHashMap, также как и HashMap имеет динамический размер, 
поэтому если его не ограничивать LruCache будет расти до бесконечности */
    private final LinkedHashMap<K, V> map;

    // текущий и максимальный размер соответственно
    private int size;
    private int maxSize;

    // своего рода статистическая информация
    private int putCount;
    private int createCount;
    private int evictionCount;
    private int hitCount;
    private int missCount;

    ...
}
```

Добавление нового элемента в кэш происходит через метод put:

```java
public final V put(K key, V value) {
    if (key == null || value == null) {
        throw new NullPointerException(...);
    }

    V previous;
    /* обратите внимание для увеличения производительности класса в многопоточной среде 
используется небольшой synchronized блок вместо указания всего метода данным модификатором */
    synchronized (this) {
        putCount++;
        /* размер высчитывается в условных единицах, которые можно переопределить, об этом чуть позже */
        size += safeSizeOf(key, value);
        previous = map.put(key, value);
        if (previous != null) {
            size -= safeSizeOf(key, previous);
        }
    }

    if (previous != null) {
        /* при затирании старого значения срабатывает данный callback */
        entryRemoved(false, key, previous, value);
    }
        
    /* этот метод отвечает за удаление старых элементов при превышении максимального размера кэша */
    trimToSize(maxSize);
    return previous;
}
```

Когда размера кэша начинает превышать максимальный метод trimToSize начинает удалять самые старые элементы:

```java
public void trimToSize(int maxSize) {
    while (true) {
        K key;
        V value;
        synchronized (this) {
            // некоторые проверки опущены

            // берём самый старый элемент
            Map.Entry<K, V> toEvict = map.eldest();
            if (toEvict == null) {
                break;
            }

            key = toEvict.getKey();
            value = toEvict.getValue();
            // удаляем по ключу из LinkedHashMap
            map.remove(key);
            size -= safeSizeOf(key, value);
            evictionCount++;
        }
        /* также отрабатывает данный callback, обратите внимание первый параметр теперь равен true */
        entryRemoved(true, key, value, null);
    }
}
```

В этом и есть принцип работы данного класса, из которого можно вынести пример хорошей инкапсуляции и практику использование небольших synchronized блоков вместо полноценных методов, помеченных данным модификатором.

#### 3) Небольшой пример наследования LruCache и переопределения некоторых методов для Bitmap:

```kotlin
class BitmapCache : LruCache<String, Bitmap>(100 * 1024) {

    override fun entryRemoved(
        evicted: Boolean, 
        key: String, 
        oldValue: Bitmap, 
        newValue: Bitmap?
    ) {
        if (evicted) {
            /* Bitmap'a была удалена при превышении максимального размера кэша */
        } else {
            // Bitmap'a была перезаписана другой
        }
    }
    
    /* мы хотим указывать размера кэша в байтах, поэтому переопределяем sizeOf для изменения 
относительного размера, по умолчанию данный метод возвращает 1, что эквивалентно 
количеству добавленных элементов */
    override fun sizeOf(key: String, value: Bitmap): Int =
        value.byteCount
    
}
```

#### 4) LruCache можно встретить в таких библиотеках как Glide или Picasso:

```kotlin
/* обратите внимание, здесь используется реализация LruCache от библиотеки Picasso, 
для которой указывается размер в байтах */
val picasso = Picasso.Builder(applicationContext)
    .memoryCache(com.squareup.picasso.LruCache(100 * 1024))
    .build()
```

Всем хорошего кода!





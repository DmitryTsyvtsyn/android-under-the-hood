# Хранение нескольких значений в одном целочисленном типе с помощью битовых масок

В Android есть так называемый MeasureSpec, кто писал кастомные вьюшки тот в курсе, как извлекаются значения из него:

```kotlin
class CustomView(ctx: Context) : View(ctx) {

    override fun onMeasure(widthMeasureSpec: Int, heightMeasureSpec: Int) {

        val width = MeasureSpec.getSize(widthMeasureSpec)
        val widthMode = MeasureSpec.getMode(widthMeasureSpec)
                
        val height = MeasureSpec.getSize(heightMeasureSpec)
        val heightMode = MeasureSpec.getMode(heightMeasureSpec)

        // ...
    }

}
```

Если глянуть под капот этих методов, то можно увидеть битовые операции с одним и тем же целочисленным значением:

```java

public static int getSize(int measureSpec) {
    return (measureSpec & ~MODE_MASK)
}

public static int getMode(int measureSpec) {
    return (measureSpec & MODE_MASK);
}

```

Давайте попробуем разобраться что здесь происходит.

MODE_MASK это специальная константа:

```java

private static final int MODE_SHIFT = 30;
private static final int MODE_MASK  = 0x3 << MODE_SHIFT;

```

Распишем 0x3 в двоичной системе (битовые операции работают с отдельными битами):

    0x3 = 00000000 00000000 00000000 00000011

Выполним побитовый сбиг влево:

    0x3 << 30 = 11000000 00000000 00000000 00000000

Снова вернёмся к методу:

```java

public static int getMode(int measureSpec) {
    return (measureSpec & MODE_MASK);
}

```

Оператор & выполняет побитовую операцию И (bitwise AND), простыми словами выставляет единичный бит если оба бита таковыми являются:

    01101110 00110001 10001100 01101111 & 11000000 00000000 00000000 00000000 = 
    01000000 00000000 00000000 00000000

Таким образом метод MeasureSpec.getMode() берет только первые два бита целочисленного числа, а остальные зануляет.

Два бита нужны для сохранения одного из следующих значений (легковесный enum на битах):

```java

// 00000000 00000000 00000000 00000000
public static final int UNSPECIFIED = 0 << MODE_SHIFT;

// 01000000 00000000 00000000 00000000
public static final int EXACTLY = 1 << MODE_SHIFT;

// 10000000 00000000 00000000 00000000
public static final int AT_MOST = 2 << MODE_SHIFT;

```

Второй метод работает практически аналогично, но только извлекает все биты кроме первых двух:

```java
public static int getSize(int measureSpec) {
    return (measureSpec & ~MODE_MASK)
}
```

Оператор ~ выполняет побитовую инверсию, меняет нулевые биты на единичные и наоборот:

    ~11000000 00000000 00000000 00000000 = 00111111 11111111 11111111 11111111

После применения инвертированной маски ~MODE_MASK остаются все биты кроме первых двух:

    01101110 00110001 10001100 01101111 & 00111111 11111111 11111111 11111111 =
    00101110 00110001 10001100 01101111

Обобщим полученные результаты:

1) MeasureSpec.getMode() берет только первые два бита целочисленного числа, а остальные зануляет
2) MeasureSpec.getSize() зануляет первые два бита целочисленного числа и берет остальные
    
Вот таким элегантным и эффективным способом MeasureSpec хранит в одном целочисленном типе два значения: мини enum на битах для определения режима и ширину/высоту дочерней вьюшки.

Для более любопытных предлагаю чекнуть исходники <code>android.graphics.Color</code> и глянуть как извлекаются отдельные компоненты [RGB](https://ru.wikipedia.org/wiki/RGB) модели.

Всем хорошего кода и побольше вкусняшек!




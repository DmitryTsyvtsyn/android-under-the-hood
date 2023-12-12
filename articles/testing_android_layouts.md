# Тестируем Android layout'ы

Для тестирования layout'ов я написал следующий код в Activity:

```kotlin
val rootView = FrameLayout(this)

val addViewButton = Button(this)
addViewButton.text = "Add view"
addViewButton.layoutParams = FrameLayout.LayoutParams(
    FrameLayout.LayoutParams.WRAP_CONTENT,
    FrameLayout.LayoutParams.WRAP_CONTENT
).apply {
    gravity = Gravity.START or Gravity.BOTTOM
}
rootView.addView(addViewButton)

val removeViewButton = Button(this)
removeViewButton.text = "Remove view"
removeViewButton.layoutParams = FrameLayout.LayoutParams(
    FrameLayout.LayoutParams.WRAP_CONTENT,
    FrameLayout.LayoutParams.WRAP_CONTENT
).apply {
    gravity = Gravity.END or Gravity.BOTTOM
}
rootView.addView(removeViewButton)

// здесь будет меняться класс layout'а на тестируемый
val contentView = object : LinearLayout(this) {
    override fun onMeasure(widthMeasureSpec: Int, heightMeasureSpec: Int) {
        super.onMeasure(widthMeasureSpec, heightMeasureSpec)
        Log.d(tag, "layout onMeasure")
    }
    
    override fun dispatchDraw(canvas: Canvas?) {
        super.dispatchDraw(canvas)
        Log.d(tag, "layout dispatchDraw")
    }
}
contentView.orientation = LinearLayout.VERTICAL
contentView.layoutParams = FrameLayout.LayoutParams(
    FrameLayout.LayoutParams.MATCH_PARENT,
    FrameLayout.LayoutParams.MATCH_PARENT
)
rootView.addView(contentView)

val views = mutableListOf<View>()

addViewButton.setOnClickListener {
    val viewIndex = views.size
    val textView = object : AppCompatTextView(this) {
        override fun onMeasure(widthMeasureSpec: Int, heightMeasureSpec: Int) {
            super.onMeasure(widthMeasureSpec, heightMeasureSpec)
            Log.d(tag, "view $viewIndex onMeasure")
        }
      
        override fun onDraw(canvas: Canvas?) {
            super.onDraw(canvas)
            Log.d(tag, "view $viewIndex onDraw")
        }
    }
    textView.text = "Text..."
    // также будут меняться layoutParams в зависимости от layout'а
    textView.layoutParams = LinearLayout.LayoutParams(
        LinearLayout.LayoutParams.MATCH_PARENT,
        LinearLayout.LayoutParams.WRAP_CONTENT
    ).apply {
        weight = 1f
    }
    // для добавления вьюшек в начало будет использоваться addView(textView, 0)
    contentView.addView(textView)
    views.add(textView)
}

removeViewButton.setOnClickListener {
    // чтобы протестировать удаление вьюшек с начала layout'а 
    // я буду менять removeLastOrNull на removeFirstOrNull
    val view = views.removeLastOrNull()
    contentView.removeView(view)
}
    
setContentView(rootView)
```

Результаты тестирования будут измеряться в абстрактных единицах, а именно в количестве вызовов методов onMeasure, onDraw и dispathDraw.

Возможно это слишком простое исследование, но оно даёт понять сколько раз layout'у нужно пройтись по дочерним элементам, чтобы разместить и отрисовать их.

### android.widget.FrameLayout

Добавим несколько вьюшек и посмотрим логи:

```
view 0 onMeasure
layout onMeasure
view 0 onDraw
layout dispatchDraw

view 1 onMeasure
layout onMeasure
view 1 onDraw
layout dispatchDraw

view 2 onMeasure
layout onMeasure
view 2 onDraw
layout dispatchDraw
```

Как мы видим, при добавлении вьюшки во FrameLayout нужно одно измерение.

Теперь попробуем удалить все вьюшки:

```
layout onMeasure
layout dispatchDraw

layout onMeasure
layout dispatchDraw

layout onMeasure
layout dispatchDraw
```

Результат: при удалении вьюшки из FrameLayout'а остальные не переизмеряются заново.

Аналогичные показатели и в случае добавления в начало или удаления с конца.

Вывод: эффективный и простой layout с тривиальным алгоритмом измерения и отрисовки вьюшек.

### android.widget.LinearLayout

<b>Заметка</b>: данные показатели характерны для обеих ориентаций: LinearLayout.VERTICAL и LinearLayout.HORIZONTAL.

При добавлении/удалении вьюшек в естественном порядке (без использования addView с указанием индекса) и отсутствием весов LinearLayout показывают такие же результаты, как и FrameLayout.

Если при добавлении вьюшки указать индекс в методе addView(view, index), то уже существующая вьюшка по данному индексу заново будет измерена и отрисована:
    
    // добавление первой вьюшки
    view 0 onMeasure
    layout onMeasure
    view 0 onDraw
    layout dispatchDraw
    
    // добавление второй вьюшки перед первой: первая заново измеряется и отрисовывается
    view 1 onMeasure
    view 0 onMeasure
    layout onMeasure
    view 1 onDraw
    view 0 onDraw
    layout dispatchDraw
    
    // добавление третьей вьюшки перед второй: вторая заново измеряется и отрисовывается 
    view 2 onMeasure
    view 1 onMeasure
    layout onMeasure
    view 2 onDraw
    view 1 onDraw
    layout dispatchDraw

При удалении происходит похожая картина, только переизмеряются вьюшки, следующие за удаляемой:

    // удаляем первую вьюшку, переизмеряются вторая и третья
    view 1 onMeasure
    view 2 onMeasure
    layout onMeasure
    layout dispatchDraw

    // удаляем вторую вьюшку, переизмеряется третья
    view 2 onMeasure
    layout onMeasure
    layout dispatchDraw

    // удаляем третью вьюшку, нечего измерять
    layout onMeasure
    layout dispatchDraw

Обратите внимание, я добавлял вьюшки по нулевому индексу addView(view, 0) и удалял также с самого начала.

Теперь попробуем добавить вьюшки с weight параметром:

    // первая вьюшка измеряется два раза
    view 0 onMeasure
    view 0 onMeasure
    layout onMeasure
    view 0 onDraw
    layout dispatchDraw

    // вторая вьюшка измеряет два раза, а первая один
    view 1 onMeasure
    view 0 onMeasure
    view 1 onMeasure
    layout onMeasure
    view 0 onDraw
    view 1 onDraw
    layout dispatchDraw

    // третья вьюшка измеряет два раза, а первая и вторая по одному
    view 2 onMeasure
    view 0 onMeasure
    view 1 onMeasure
    view 2 onMeasure
    layout onMeasure
    view 0 onDraw
    view 1 onDraw
    view 2 onDraw
    layout dispatchDraw

Заметьте, вьюшка которая была добавлена последней измеряется тоже последней.

Поменяем порядок и будем добавлять вьюшки с weight параметром в начало методом addView(view, 0):

    view 0 onMeasure
    view 0 onMeasure
    layout onMeasure
    view 0 onDraw
    layout dispatchDraw

    // первая вьюшка в родителе теперь та, которая была добавлена последней
    // поэтому view 1 будет измерена самой первой
    view 1 onMeasure
    view 1 onMeasure
    view 0 onMeasure
    layout onMeasure
    view 1 onDraw
    view 0 onDraw
    layout dispatchDraw

    // view 2 также будет измерена первой, потому что в родителе
    // она следует раньше остальных, не забывайте про addView(view, 0)
    view 2 onMeasure
    view 2 onMeasure
    view 1 onMeasure
    view 0 onMeasure
    layout onMeasure
    view 2 onDraw
    view 1 onDraw
    view 0 onDraw
    layout dispatchDraw

Результат: при добавлении новой вьюшки с weight параметром измерение происходит два раза, для уже добавленных - один, порядок измерения зависит от порядка следования вьюшек в родителе.

Удаление вьюшек с weight параметром происходит аналогично добавлению, только порядок измерения меняется:

    // удаление с конца родителя
    layout onMeasure
    view 0 onMeasure
    view 1 onMeasure
    view 0 onDraw
    view 1 onDraw
    layout dispatchDraw
    
    layout onMeasure
    view 0 onMeasure
    view 0 onDraw
    layout dispatchDraw
    
    layout onMeasure
    layout dispatchDraw

    // удаление с начала родителя
    view 2 onMeasure
    layout onMeasure
    view 1 onMeasure
    view 1 onDraw
    view 2 onDraw
    layout dispatchDraw
    
    view 2 onMeasure
    layout onMeasure
    view 2 onDraw
    layout dispatchDraw
    
    layout onMeasure
    layout dispatchDraw

Результат: при удалении вьюшки с weight параметром, остальные измеряются по одному разу, порядок измерения противоположен порядку удаления.

Вывод: LinearLayout один из самых легковесных и простых layout'ов, которым можно сверстать огромное количество экранов и очень сложных, не боясь вложенности (без злоупотребления weight параметрами).

### androidx.constraintlayout.widget.ConstraintLayout

При добавлении новой вьюшки измерение происходит два раза:

    view 0 onMeasure
    view 0 onMeasure
    layout onMeasure
    view 0 onDraw
    layout dispatchDraw
    
    view 1 onMeasure
    view 1 onMeasure
    layout onMeasure
    view 1 onDraw
    layout dispatchDraw
    
    view 2 onMeasure
    view 2 onMeasure
    layout onMeasure
    view 2 onDraw
    layout dispatchDraw

Это характерно для одинарных привязок без цепочек, другими словами одна вьюшка привязана к другой без взаимной привязки.

При удалении вьюшек с одинарными привязками ConstraintLayout ничего не измеряет и не перерисовывает:

    layout onMeasure
    layout dispatchDraw
    
    layout onMeasure
    layout dispatchDraw
    
    layout onMeasure
    layout dispatchDraw

Это похоже на удаление во FrameLayout'е, но это не значит что ConstraintLayout удаляет вьюшки также быстро, к тому же данный случай характерен для очевидных и последовательных экранов, в которых вьюшки содержат только одинарные привязки.

Посмотрим случай добавления вьюшек с двойной связкой (цепочки):

    // первая вьюшка была измерена два раза
    view 0 onMeasure
    view 0 onMeasure
    layout onMeasure
    view 0 onDraw
    layout dispatchDraw

    // вторая вьюшка была измерена два раза, первая тоже
    view 0 onMeasure
    view 1 onMeasure
    view 0 onMeasure
    view 1 onMeasure
    layout onMeasure
    view 0 onDraw
    view 1 onDraw
    layout dispatchDraw

    // третья вьюшка была измерена два раза, первая и вторая тоже
    view 0 onMeasure
    view 1 onMeasure
    view 2 onMeasure
    view 0 onMeasure
    view 1 onMeasure
    view 2 onMeasure
    layout onMeasure
    view 0 onDraw
    view 1 onDraw
    view 2 onDraw
    layout dispatchDraw

К сожалению вьюшки переизмеряются два раза, а не один. Возможно есть способ избежать этого, но мне на ум приходит только вариант без использования двойных привязок.

При удалении вьюшек с двойной привязкой переизмеряются уже добавленные:

    // первая и вторая вьюшка была измерена два раза
    view 0 onMeasure
    view 1 onMeasure
    view 0 onMeasure
    view 1 onMeasure
    layout onMeasure
    view 0 onDraw
    view 1 onDraw
    layout dispatchDraw

    // первая вьюшка была измерена два раза
    view 0 onMeasure
    view 0 onMeasure
    layout onMeasure
    view 0 onDraw
    layout dispatchDraw

    // вьюшек больше нет
    layout onMeasure
    layout dispatchDraw

Вывод: старайтесь не использовать двойные привязки между вьюшками и обходиться без цепочек.

Также помните, что ConstraintLayout очень сложная штука с нетривиальным алгоритмом измерения вьюшек, поэтому старайтесь обходиться без него, к тому же из коробки всегда есть FrameLayout и LinearLayout.

В заключении статьи хотел бы сказать, что мои тесты не измеряют производительность layout'ов, а показывают возможные кейсы, при которых чаще или реже происходят измерения дочерних элементов и насколько добавление новой вьюшки влияет на уже существующие: будут они измерены заново или нет, а если будут то сколько раз. 

Хорошего кода и побольше вкусняшек!


















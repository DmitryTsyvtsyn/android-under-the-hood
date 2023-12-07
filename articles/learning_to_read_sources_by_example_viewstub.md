# Учимся читать исходники на примере android.view.ViewStub

Начнем с небольшой справки.

ViewStub используется в тех случаях когда мы хотим лениво создать вьюшку из layout ресурса:

```xml

<FrameLayout 
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent">
    
    <ViewStub
        android:id="@+id/view_stub"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:layout="@layout/five_nights_freddy_item"
        android:inflatedId="@+id/five_nights_freddy_root_view" />

</FrameLayout>

```

Когда нам понадобится распарсить layout и получить готовую вьюшку вызываем следующий метод:

```kotlin
val viewStub = findViewById<ViewStub>(R.id.view_stub)
viewStub.inflate()
```

Возможно ViewStub не такая полезная штука, но в качестве примера для чтения исходного кода подойдет.

Для начала я бы хотел привезти несколько базовых принципов или правил, которыми сам руководствуюсь, когда читаю исходный код библиотеки или Android SDK.

#### Не пытайтесь понять всё

Зачастую код бывает сложным и запутанным и если пытаться ухватиться за каждый его кусок вероятнее всего ваша голова взорвётся и вы не захотите больше его видеть.

#### Возвращайтесь к исходникам

Сегодня вы может быть ничего не понимаете, но со временем кругозор расширяется и неочевидные ранее вещи становятся очевидными сейчас. 

#### Читайте код с которым работаете

Когда вы делаете запрос на бэк вызовом <code>okHttpClient.newCall(request).enqueue(...)</code>, вы соприкасаетесь с кодом библиотеки OkHttp, тогда почему бы не начать читать исходники этой библиотеки!

То же самое происходит и в больших командах с многомодульной структурой, не живите одним своим модулем, лезьте в чужие, смотрите как пишут другие инженеры и берите для себя лучшие практики.

#### Мыслите структурами

Если это большой класс с кучей полей и методов попытайтесь выделить основные компоненты и их логическую связь друг с другом. Любой класс, даже ужасно написанный, можно разбить на отдельные кусочки и выделить основную логику.

Снова возвращаемся к ViewStub, попробуем разобраться как работает этот класс, переходим в исходники (Ctrl + B):

```java
// подмечаем, что это наследник от класса View (не ViewGroup)
public final class ViewStub extends View {

    // какие то две переменные, пока точно не знаем, но похоже на id и layout ресурсы
    private int mInflatedId;
    private int mLayoutResource;

    // подмечаем слабую ссылку на вьюшку
    private WeakReference<View> mInflatedViewRef;

    private LayoutInflater mInflater;
    private OnInflateListener mInflateListener;

    // смотрим конструкторы класса
    public ViewStub(Context context) {
        this(context, 0);
    }

    public ViewStub(Context context, @LayoutRes int layoutResource) {
        this(context, null);

        mLayoutResource = layoutResource;
    }

    public ViewStub(Context context, AttributeSet attrs) {
        this(context, attrs, 0);
    }

    public ViewStub(Context context, AttributeSet attrs, int defStyleAttr) {
        this(context, attrs, defStyleAttr, 0);
    }

    public ViewStub(Context context, AttributeSet attrs, int defStyleAttr, int defStyleRes) {
        super(context);

        final TypedArray a = context.obtainStyledAttributes(attrs,
                R.styleable.ViewStub, defStyleAttr, defStyleRes);
        saveAttributeDataForStyleable(context, R.styleable.ViewStub, attrs, a, defStyleAttr,
                defStyleRes);

        // подмечаем xml аттрибуты inflatedId и layout, которые ранее в примере уже были продемонстрированы
        // обратите внимание, что не нужно читать документацию когда вы понимаете какой параметр и куда присвается
        mInflatedId = a.getResourceId(R.styleable.ViewStub_inflatedId, NO_ID);
        mLayoutResource = a.getResourceId(R.styleable.ViewStub_layout, 0);
        mID = a.getResourceId(R.styleable.ViewStub_id, NO_ID);
        a.recycle();

        setVisibility(GONE);
        // устанавливает интересный флаг, благодаря которому onDraw вызываться не будет
        // вот так во время чтения исходников можно подметить что нибудь интересное
        setWillNotDraw(true);
    }

    ...
}

```

В любом классе важно замечать самые базовые вещи: конструкторы и поля, это помогает сформировать первое впечатление то чем класс является.

Далее мы можем пойти несколькими путями:

1) Прочитать исходный код сверху вниз
2) Прочитать исходный код API методов класса и двигаться от них

Первый подходит для небольших классов с одним/двумя методами и минимальным количеством полей, а второй как раз наш случай.

API методы - это те самые методы которые используются при взаимодействии с классом.

Посмотрим на API методы ViewStub:

```kotlin
viewStub.setInflatedId(R.id.five_nights_freddy_root_view)
viewStub.setLayoutResource(R.layout.five_nights_freddy_item)
viewStub.setOnInflateListener { stub, inflated ->  
    
}
viewStub.inflate()

```

Первые три метода простые сеттеры, а вот последний непосредственно и реализует всю основную логику ViewStub:

```java
public View inflate() {
    final ViewParent viewParent = getParent();
    // получаем ViewGroup текущей вьюшки (ViewStub)
    if (viewParent != null && viewParent instanceof ViewGroup) {
        if (mLayoutResource != 0) {
            final ViewGroup parent = (ViewGroup) viewParent;
            // делаем вызов inflater.inflate(layoutResource)
            final View view = inflateViewNoAdd(parent);
            // заменяем текущую вьюшку (ViewStub) только что созданной
            replaceSelfWithView(view, parent);

            // сохраняем созданную вьюшку слабой ссылкой для вспомогательных методов:
            // изменение видимости через viewStub.setVisibility() и тд
            mInflatedViewRef = new WeakReference<>(view);
            if (mInflateListener != null) {
                mInflateListener.onInflate(this, view);
            }

            return view;
        } else {
            throw new IllegalArgumentException("ViewStub must have a valid layoutResource");
        }
    } else {
        throw new IllegalStateException("ViewStub must have a non-null ViewGroup viewParent");
    }
}

// создает вьюшку из layout ресурса и присваивает ей id
private View inflateViewNoAdd(ViewGroup parent) {
    final LayoutInflater factory;
    if (mInflater != null) {
        factory = mInflater;
    } else {
        factory = LayoutInflater.from(mContext);
    }
    final View view = factory.inflate(mLayoutResource, parent, false);
    
    if (mInflatedId != NO_ID) {
        view.setId(mInflatedId);
    }
    return view;
}

// заменяет текущую вьюшку (ViewStub) в родителе parent другой view
private void replaceSelfWithView(View view, ViewGroup parent) {
    final int index = parent.indexOfChild(this);
    parent.removeViewInLayout(this);

    // обратите внимание, что layoutParams для новой вьюшки берутся из ViewStub
    final ViewGroup.LayoutParams layoutParams = getLayoutParams();
    if (layoutParams != null) {
        parent.addView(view, index, layoutParams);
    } else {
        parent.addView(view, index);
    }
}
```

Помимо базовой логики ViewStub есть еще вспомогательные и переопределенные методы (setVisibility), исходный код которых вы можете изучить самостоятельно в качестве практики.

<b>Важно понимать принцип</b>: чтение сложных классов должно начинаться с базовых методов (API методы) и постепенного углубления в детали реализации.

Знание простых правил и адекватный подход к чтению исходников это здорово. Но еще лучше иметь опыт и знать об инженерных практиках и решениях.

Постараюсь привезти несколько наглядных примеров с кратким описанием. 

#### Конструкция throw new IllegalStateException(...)

Ранее вы уже видели такую конструкцию в классе ViewStub:

```java
public View inflate() {
    final ViewParent viewParent = getParent();
    
    if (viewParent != null && viewParent instanceof ViewGroup) {
        if (mLayoutResource != 0) {
            ....
        } else {
            throw new IllegalArgumentException("ViewStub must have a valid layoutResource");
        }
    } else {
        throw new IllegalStateException("ViewStub must have a non-null ViewGroup viewParent");
    }
}
```

Это нужно для того, чтобы сообщить о некорректной работе класса.

Представьте вы забыли установить mLayoutResource и класс не выбрасывает исключение. 

Что произойдет? Ничего, вы долго будете ломать голову в чем проблема пока не глянете исходники или не зайдёте на StackOverflow.

Вывод: бросайте исключение если ваш класс начинает некорректно работать.

#### Использование паттернов

Нельзя сказать, что в исходном коде нет паттернов проектирования, даже в самом запутанном и плохо написанном коде вы найдете парочку, потому они и зовутся паттернами. 

Парочку примеров:

1) androidx.lifecycle.LiveData - паттерн Observer
2) androidx.lifecycle.ViewModelProvider - паттерн Factory

Вывод: изучайте паттерны проектирования чтобы лучше понимать исходный код.

#### Битовые или Boolean флаги

Флаги это наверно одна из основных головных болей при чтении кода, особенно когда их очень много и они разбросаны по всему классу.

Небольшой пример Boolean флага:

```java
public class Thread implements Runnable {

    // ...

    boolean started = false;

    public synchronized void start() {
       
        if (started)
            throw new IllegalThreadStateException();

        // ...
       
        started = false;
        try {
            // ...
            started = true;
        } finally {
            // ...
        }
    }

}
```

Бывают ситуации, когда флагов становится очень много и вместо аллоцирования множества Boolean переменных создаются битовые маски:

```java
public class View implements Drawable.Callback, KeyEvent.Callback, AccessibilityEventSource {

    static final int PFLAG_FOCUSED = 0x00000002;
    static final int PFLAG_SELECTED = 0x00000004;
    static final int PFLAG_DRAWN = 0x00000020;
    static final int PFLAG_SKIP_DRAW = 0x00000080;
    static final int PFLAG_FORCE_LAYOUT = 0x00001000; 
    // ...

    public int mPrivateFlags;

    void setFlags(int flags, int mask) {

        // ...

        // установка флага
        mPrivateFlags |= PFLAG_SKIP_DRAW;

        // сброс флага
        mPrivateFlags &= ~PFLAG_FOCUSED;

        // вариант 1
        if ((mPrivateFlags & PFLAG_SKIP_DRAW) == PFLAG_SKIP_DRAW) {
            // установлен
        } else {
            // не установлен
        }

        // вариант 2
        if ((mPrivateFlags & PFLAG_SKIP_DRAW) != 0) {
            // установлен
        } else {
            // не установлен
        }

    }

}
```

Современные библиотеки, написанные опытными инженерами зачастую содержат небольшое количество флагов или обходятся без них. Но старый код остается и поэтому иногда приходиться сталкиваться с подобной неразберихой.

Вывод: стремитесь обходиться минимальным количеством флагов.

#### Структуры данных

В сторонних библиотеках или Android SDK можно встретить большое множество структур данных. Наиболее часто используемые из них: массивы, HashMap и очереди. 

Пример: библиотека [OkHttp](https://github.com/square/okhttp) сохраняет соединения в очередь:

```kotlin
class RealConnectionPool(...) {

    // хранит HTTP соединения с сервером
    private val connections = ConcurrentLinkedQueue<RealConnection>()

    ...

}
````

Или android.lifecycle.ViewModel хранятся в HashMap:

```java
public class ViewModelStore {

    private final HashMap<String, ViewModel> mMap = new HashMap<>();

    final void put(String key, ViewModel viewModel) {
        ViewModel oldViewModel = mMap.put(key, viewModel);
        if (oldViewModel != null) {
            oldViewModel.onCleared();
        }
    }

    final ViewModel get(String key) {
        return mMap.get(key);
    }

    Set<String> keys() {
        return new HashSet<>(mMap.keySet());
    }

}
```

Давайте сделаем заключительный вывод.

Чтение исходного кода - трудоёмкий и сложный процесс, требующий не только разумного подхода, знаний и терпения, но и богатого разностороннего опыта. 

P.S. Читайте исходники Android SDK и библиотек которые используете и не забывайте про паттерны проектирования и структуры данных.

Всем хорошего кода и побольше вкусняшек!





# Интересные приемы, взятые из исходного кода Android SDK

В процессе чтения исходников Android SDK я сталкивался с интересными механиками и приемами, о некоторых я расскажу в этой статье.

Накладывайте себе шоколадные печеньки, наливайте кофе или чай и как говорят испанцы ¡Vamos!

### Переопределение protected метода на public в наследуемом классе

Думаю все кто изучал Java знают что можно сделать так (в Kotlin такой возможности нет):

    public abstract class Property<T> {

        private T value;
  
        public Property(T value) {
            this.value = value;
        }
  
        protected void setValue(T value) {
            this.value = value
        }
  
        public T getValue() {
            return value;
        }

    }

    public class MutableProperty<T> extends Property<T> {

        ...

        @Override
        public void setValue(T value) { super.setValue(value); }

    }

Такая механика языка используется для реализации MutableLiveData<T>:

    public abstract class LiveData<T> {
    
        protected void postValue(T value) {
            ...
        }
    
        @MainThread
        protected void setValue(T value) {
            ...
        }
    
    }

    public class MutableLiveData<T> extends LiveData<T> {

        ...

        @Override
        public void postValue(T value) {
            super.postValue(value);
        }
    
        @Override
        public void setValue(T value) {
            super.setValue(value);
        }
        
    }

На самом деле такой способ создания изменяемых/неизменяемых классов нарушает концепцию наследования, так как мы не добавляем новую функциональность, а "включаем" её.

Есть более предпочтительный способ с точки зрения ООП:

    class LiveData<T>(private var value: T) {
    
        fun fetchValue() = value
        
    }
    
    class MutableLiveData<T>(value: T) : LiveData<T>(value) {
    
        fun changeValue(newValue: T) { 
            this.value = newValue 
            // notify observers
        }
        
    }

В любом случае механика переопределения protected на public имеет место быть.

### ThreadLocal переменные

Если вы никогда не слышали, есть такая штука которая позволяет создать уникальный экземпляр объекта в пределах текущего потока, своебразный Singleton потока.

Посмотрим для чего это можно использовать:

    public final class Looper {
    
      static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();
    
      private static void prepare(boolean quitAllowed) {
          if (sThreadLocal.get() != null) {
              throw new RuntimeException("Only one Looper may be created per thread");
          }
          sThreadLocal.set(new Looper(quitAllowed));
      }
    
      ...
    
    }

Looper - один из самых базовых классов Android SDK на котором построена система событий.

ThreadLocal<Looper> гарантирует что Looper будет единственным экземпляром в пределах текущего потока, так как в одном потоке может быть только один бесконечный цикл обработки событий. 

Если вы создадите новый поток и вызовите Looper.prepare на нем, то для него будет создан свой уникальный экземпляр Looper и т.д.

Сложно предположить где ThreadLocal может пригодиться в повседневной Android разработке, но имейте в виду если вам нужен уникальный экземпляр в пределах текущего потока используйте ThreadLocal и обязательно посмотрите документацию с примером.

### Проксирование/Делегирование методов другому классу

Гораздо проще показать на примере <b>androidx.appcompat.app.AppCompatActivity</b> из библиотеки [Appcompat](https://developer.android.com/jetpack/androidx/releases/appcompat):

    public class AppCompatActivity extends ... {
    
        @Override
        protected void attachBaseContext(Context newBase) {
            super.attachBaseContext(getDelegate().attachBaseContext2(newBase));
        }
    
        @Override
        public void setTheme(@StyleRes final int resId) {
            super.setTheme(resId);
            getDelegate().setTheme(resId);
        }
    
        @Override
        protected void onPostCreate(@Nullable Bundle savedInstanceState) {
            super.onPostCreate(savedInstanceState);
            getDelegate().onPostCreate(savedInstanceState);
        }
    
        @Nullable
        public ActionBar getSupportActionBar() {
            return getDelegate().getSupportActionBar();
        }
    
        public void setSupportActionBar(@Nullable Toolbar toolbar) {
            getDelegate().setSupportActionBar(toolbar);
        }
    
    }

Метод getDelegate() возвращает объект класса AppCompatDelegate, методы которого реализуют функциональность для методов AppCompatActivity.

Это может пригодиться когда требуется <b>прозрачно</b> добавить дополнительную функциональность для класса с возможностью расширить её, "прозрачно" значит без влияния на пользователей этого класса.

Приведу пример добавления новой функциональности:

    public class AppCompatActivity extends ... {

        ...

        @NonNull
        public AppCompatDelegate getDelegate() {
            if (mDelegate == null) {
                // в Android 34 появились специфичные штуки 
                if (Build.VERSION.SDK_INT >= 34) {
                    mDelegate = AppCompatDelegate.create34(this, this);
                } else {
                    mDelegate = AppCompatDelegate.create(this, this);
                }
            }
            return mDelegate;
        }

    }

Достаточно прозрачно и пользователю Android SDK не приходиться менять свой код.

### Наследование с реализацией интерфейсов для построения единого API

Многие AppCompat*View классы реализованы таким образом для обеспечения единого API:

    public class AppCompatImageView extends ImageView implements TintableBackgroundView, ... {}
    
    public class AppCompatButton extends Button implements TintableBackgroundView, ... {}
    
    public class AppCompatTextView extends TextView implements TintableBackgroundView, ... {}

TintableBackgroundView это простой интерфейс для изменения цвета заднего фона:

    public interface TintableBackgroundView {
    
        void setSupportBackgroundTintList(@Nullable ColorStateList tint);
    
        @Nullable
        ColorStateList getSupportBackgroundTintList();
    
        @Nullable
        PorterDuff.Mode getSupportBackgroundTintMode();
    }

Такой подход имеет несколько преимуществ:

1) добавляет дополнительную функциональность: изменение цвета заднего фона в не зависимости от его содержимого
2) создает простой и единый интерфейс: не нужно смотреть документацию для каждого компонента, чтобы понять как у него поменять цвет
3) реализует полиморфный тип

Последнее проще продемонстрировать:

    val views: List<TintableBackgroundView> = listOf(
        AppCompatTextView(this),
        AppCompatButton(this),
        AppCompatImageView(this)
    )

    val newColor = 0xff333333.toInt()
    views.forEach {
        it.supportBackgroundTintList = ColorStateList.valueOf(newColor)
    }

### Создание дополнительного типа в качестве пустого значения

Иногда возникают ситуации, когда null не совсем подходит на роль "нет значения" и в таких случаях приходиться выкручиваться дополнительным типом:

    private class SynchronizedLazyImpl<out T>(initializer: () -> T, lock: Any? = null) : Lazy<T>, Serializable {
        private var initializer: (() -> T)? = initializer

        // дополнительный тип UNINITIALIZED_VALUE указывает что поле _value ещё не было инициализировано
        @Volatile private var _value: Any? = UNINITIALIZED_VALUE

        private val lock = lock ?: this
    
        override val value: T
            get() {
                val _v1 = _value
                // проверка состояния поля
                if (_v1 !== UNINITIALIZED_VALUE) {
                    @Suppress("UNCHECKED_CAST")
                    return _v1 as T
                }
    
                return synchronized(lock) {
                    val _v2 = _value
                    // вторая проверка состояния поля на случай если другой поток уже проинициализировал его
                    if (_v2 !== UNINITIALIZED_VALUE) {
                        @Suppress("UNCHECKED_CAST") (_v2 as T)
                    } else {
                        val typedValue = initializer!!()
                        _value = typedValue
                        initializer = null
                        typedValue
                    }
                }
            }

        // если поле не равно UNINITIALIZED_VALUE значит оно уже было проинициализировано
        override fun isInitialized(): Boolean = _value !== UNINITIALIZED_VALUE
    
        ...
        
    }

Дополнительным типом здесь является UNINITIALIZED_VALUE:

    internal object UNINITIALIZED_VALUE

Здесь нельзя обойтись null значением, так как оно входит в диапазон возможных значений:

    // временный объект может не вернуться и тогда значение будет null
    val temporaryObject by lazy { getTemporaryObject() }

### Переиспользуемый пул объектов, реализованный с помощью связанного списка

Возвращаемся к системе обработки событий в Android, а конкретнее нас интересует android.os.Message:
    
    public final class Message implements Parcelable {
    
        public static final Object sPoolSync = new Object();
        private static Message sPool;
        private static int sPoolSize = 0;
    
        private static final int MAX_POOL_SIZE = 50;
    
        // поле для организации связанного списка
        Message next;
    
        public static Message obtain() {
            synchronized (sPoolSync) {
                // если пул сообщений не пустой, берём первое доступное 
                // и возвращаем для переиспользования
                if (sPool != null) {
                    Message m = sPool;
                    sPool = m.next;
                    m.next = null;
                    m.flags = 0; // clear in-use flag
                    sPoolSize--;
                    return m;
                }
            }
            // в случае если пул был пустым или закончился создаем новое сообщение
            return new Message();
        }
    
        void recycleUnchecked() {
            // очистить поля для переиспользования объекта сообщения
            flags = FLAG_IN_USE;
            what = 0;
            arg1 = 0;
            arg2 = 0;
            obj = null;
            replyTo = null;
            sendingUid = UID_NONE;
            workSourceUid = UID_NONE;
            when = 0;
            target = null;
            callback = null;
            data = null;
    
            synchronized (sPoolSync) {
                // если лимит сообщений в пуле не превышен, добавляем текущее для переиспользования
                // в противном случае объект сообщения будет собран сборщиком мусора
                if (sPoolSize < MAX_POOL_SIZE) {
                    next = sPool;
                    sPool = this;
                    sPoolSize++;
                }
            }
        }
    
    }

По моему мнению, это достаточно эффективный и простой способ в создании переиспользуемого пула объектов, берите и используйте)

### Заключение

Надеюсь статья оказалась вам полезной и вы подчерпнули для себя что-то новое.

Желаю хорошего кода и побольше вкусняшек!















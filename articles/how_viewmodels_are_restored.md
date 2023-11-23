# Как androidx.lifecycle.ViewModel восстанавливается при изменении конфигурации?

Архитектурный паттерн [MVVM](https://ru.wikipedia.org/wiki/Model-View-ViewModel) приобрел огромную популярность в Android, а использование <b>androidx.lifecycle.ViewModel</b> встречается чуть ли не в каждом проекте.

Для начала разберемся в отличиях двух понятий:

1) ViewModel это компонент архитектуры MVVM, им может быть любой класс, необязательно наследник <b>androidx.lifecycle.ViewModel</b>
2) класс <b>androidx.lifecycle.ViewModel</b> добавляет наследникам возможность быть сохраненными при изменении конфигурации устройства (переворот экрана), а также инкапсулирует методы для очистки используемых ресурсов (addCloseable, onCleared)

<b>!ВАЖНО: androidx.lifecycle.ViewModel не является реализацией компонента архитектуры MVVM.</b>

Теперь представим что у нас есть следующая ViewModel:

    class BookListViewModel : androidx.lifecycle.ViewModel() {
        
        // че то происходит...
        
    }

Вспомним как обычно происходит создание ViewModel:

    // MainActivity.kt
    val viewModel = ViewModelProvider(this)[BookListViewModel::class.java]
    
    // MainFragment.kt
    val viewModel = ViewModelProvider(this)[BookListViewModel::class.java]

Мы используем ViewModelProvider для того, чтобы при пересоздании Activity или Fragment'a нам вернулся сохраненный экземпляр ViewModel. 

Глянем что внутри этого класса:

    public open class ViewModelProvider(
        private val store: ViewModelStore,
        private val factory: Factory,
        private val defaultCreationExtras: CreationExtras = CreationExtras.Empty,
    ) {
    
        ...
     
        public constructor(
            owner: ViewModelStoreOwner
        ) : this(owner.viewModelStore, defaultFactory(owner), defaultCreationExtras(owner))
        
        public open operator fun <T : ViewModel> get(modelClass: Class<T>): T {
            val canonicalName = modelClass.canonicalName
                ?: throw IllegalArgumentException("Local and anonymous classes can not be ViewModels")
            return get("$DEFAULT_KEY:$canonicalName", modelClass)
        }
    
        public open operator fun <T : ViewModel> get(key: String, modelClass: Class<T>): T {
            val viewModel = store[key]
            if (modelClass.isInstance(viewModel)) {
                (factory as? OnRequeryFactory)?.onRequery(viewModel)
                return viewModel as T
            } else {
                @Suppress("ControlFlowWithEmptyBody")
                if (viewModel != null) {
                    // TODO: log a warning.
                }
            }
            val extras = MutableCreationExtras(defaultCreationExtras)
            extras[VIEW_MODEL_KEY] = key
            // AGP has some desugaring issues associated with compileOnly dependencies so we need to
            // fall back to the other create method to keep from crashing.
            return try {
                factory.create(modelClass, extras)
            } catch (e: AbstractMethodError) {
                factory.create(modelClass)
            }.also { store.put(key, it) }
        }
    
        ...
    }   

Я упростил код и опустил лишние детали, разберёмся по кусочкам.

Обязательным параметром в конструкторе является класс, реализующий интерфейс ViewModelStoreOwner:

       public constructor(
            owner: ViewModelStoreOwner
        ) : this(owner.viewModelStore, defaultFactory(owner), defaultCreationExtras(owner))

ViewModelStoreOwner это простой интерфейс с одним методом, который возвращает ViewModelStore:

    public interface ViewModelStoreOwner {
        @NonNull
        ViewModelStore getViewModelStore();
    }

И теперь внимание на следующий код:

    public open operator fun <T : ViewModel> get(key: String, modelClass: Class<T>): T {

        // store это экземпляр ViewModelStore
        // хранит в ViewModel'и в HashMap
        val viewModel = store[key]

        // если ViewModel была уже создана то не пересоздаем, а возвращаем ранее созданный экземпляр  
        if (modelClass.isInstance(viewModel)) {
            (factory as? OnRequeryFactory)?.onRequery(viewModel)
            return viewModel as T
        } else {
            @Suppress("ControlFlowWithEmptyBody")
            if (viewModel != null) {
                // TODO: log a warning.
            }
        }
        
        val extras = ...
        
        return try {
            factory.create(modelClass, extras)
        } catch (e: AbstractMethodError) {
            factory.create(modelClass)
        }.also { 
            // при первом создании ViewModel, она кладется в store
            store.put(key, it) 
        }
    }

Как вы видите, ViewModel берётся из ViewModelStore, а в случае первого создания добавляется.

ViewModelStore это простая обертка над HashMap с простым функционалом очистки (вспомните метод onCleared):

    public class ViewModelStore {

        // самая обычная HashMap и всех обычных -_-
        private final HashMap<String, ViewModel> mMap = new HashMap<>();
    
        final void put(String key, ViewModel viewModel) {
            ViewModel oldViewModel = mMap.put(key, viewModel);
            // если по указанному ключу была ViewModel, то у нее вызывается onCleared
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
    
        // пробегается по всем ViewModel'кам и вызывает clear
        // можете глянуть реализацию clear, она лежит в androidx.lifecycle.ViewModel
        // P.S. под капотом clear вызывает onCleared 
        public final void clear() {
            for (ViewModel vm : mMap.values()) {
                vm.clear();
            }
            mMap.clear();
        }
    }

Теперь вы знаете как хранятся экземпляры ViewModel, во ViewModelStore, а ViewModelStore возвращает реализация ViewModelStoreOwner.

Как вы думаете что является реализацией ViewModelStoreOwner? 

Оставьте на минутку статью и попробуйте сами догадаться.

Получилось? Поздравляю! Ну а если вам лень думать то читайте дальше)

На самом деле все просто, вам не нужно далеко ходить, чтобы найти реализацию ViewModelStoreOwner интерфейса:


    // MainActivity.kt
    val viewModel = ViewModelProvider(this)[BookListViewModel::class.java]
    
    // MainFragment.kt
    val viewModel = ViewModelProvider(this)[BookListViewModel::class.java]

Ссылка на текущий объект this в MainActivity/MainFragment указывает нам что эти компоненты реализуют ViewModelStoreOwner интерфейс.

Пройдёмся по порядку.

### Реализация ViewModelStoreOwner у Activity

А точнее реализация находится в наследнике <b>androidx.activity.ComponentActivity</b>.

Чтобы вы не запутались, приведу иерархию наследования Activity (слева - суперкласс, справа - подкласс):

    Activity -> ComponentActivity -> FragmentActivity -> AppCompatActivity

О иерархии Activity можно говорить часами, нас интересует только следующий кусок кода:

    public class ComponentActivity extends ... implements ... {
    
        static final class NonConfigurationInstances {
            Object custom;
            ViewModelStore viewModelStore;
        }
    
        private ViewModelStore mViewModelStore;
    
        public ComponentActivity() {
            Lifecycle lifecycle = getLifecycle();
    
            ...
           
            getLifecycle().addObserver(new LifecycleEventObserver() {
                @Override
                public void onStateChanged(@NonNull LifecycleOwner source, @NonNull Lifecycle.Event event) {
                    if (event == Lifecycle.Event.ON_DESTROY) {
                        
                        mContextAwareHelper.clearAvailableContext();

                        // если Activity уничтожается и это не изменение конфигурации мы очищаем ViewModel'и
                        // вспомните onCleared и addCloseable, если юзали
                        if (!isChangingConfigurations()) {
                            getViewModelStore().clear();
                        }
                    }
                }
            });
            
            // немного философии: на самом деле данный способ проинициализировать ViewModelStore выглядит 
            // избыточным, так как метод getViewModelStore() гарантирует инициализацию 
            // скорее всего это фикс бага или покрытие специфичного кейса использовании
            getLifecycle().addObserver(new LifecycleEventObserver() {
                @Override
                public void onStateChanged(@NonNull LifecycleOwner source, @NonNull Lifecycle.Event event) {
                    // сработает при любом событии жизненного цикла, проинициализирует ViewModelStore
                    // и сам от себя отпишиться
                    ensureViewModelStore();
                    getLifecycle().removeObserver(this);
                }
            });
    
            ...
        }
    
        // возвращает экземпляр NonConfigurationInstances который будет сохранен системой при изменении конфигурации
        // и позже возвращен методом getLastNonConfigurationInstance
        public final Object onRetainNonConfigurationInstance() {
            // Maintain backward compatibility.
            Object custom = onRetainCustomNonConfigurationInstance();
    
            ViewModelStore viewModelStore = mViewModelStore;
            if (viewModelStore == null) {
                NonConfigurationInstances nc = (NonConfigurationInstances) getLastNonConfigurationInstance();
                if (nc != null) {
                    viewModelStore = nc.viewModelStore;
                }
            }
    
            if (viewModelStore == null && custom == null) {
                return null;
            }

            NonConfigurationInstances nci = new NonConfigurationInstances();
            nci.custom = custom;
            // обязательно кладется viewModelStore, чтобы сохранить все ViewModel'и при изменении конфигурации
            nci.viewModelStore = viewModelStore;
            return nci;
        }

        // тот самый метод, который нужно реализовать для интерфейса ViewModelStoreOwner
        public ViewModelStore getViewModelStore() {
            if (getApplication() == null) {
                throw new IllegalStateException("Your activity is not yet attached to the "
                        + "Application instance. You can't request ViewModel before onCreate call.");
            }
            ensureViewModelStore();
            return mViewModelStore;
        }

        // вспомогательный метод, который инициализирует mViewModelStore
        void ensureViewModelStore() {
            if (mViewModelStore == null) {
                NonConfigurationInstances nc = (NonConfigurationInstances) getLastNonConfigurationInstance();
                if (nc != null) {
                    mViewModelStore = nc.viewModelStore;
                }
                if (mViewModelStore == null) {
                    mViewModelStore = new ViewModelStore();
                }
            }
        }
    
    }

Код достаточно простой и понятный, но вы можете недоумевать и задаться вопросом "В чем магия?".

Магии нет, все дело в двух методах:

    // этот метод следует переопределить и вернуть объект, который вы хотите сохранить перед изменением конфигурации
    public Object onRetainNonConfigurationInstance() {
        return null;
    }

    // возвращает ранее сохраненный объект
    @Nullable
    public Object getLastNonConfigurationInstance() {
        return mLastNonConfigurationInstances != null
                ? mLastNonConfigurationInstances.activity : null;
    }

Куда сохраняется объект? Этим занимается система, как она это делает не скажу, так как не смотрел что происходит за пределами Android SDK.

Давайте попробуем написать свою собственную ViewModel без наследования <b>androidx.lifecycle.ViewModel</b>:

    class BookListViewModel {
    
        // че то происходит...
    
    }
    
    class MaiActivity : Activity() {
    
        private lateinit var viewModel: BookListViewModel
        
        override fun onCreate(savedInstanceState: Bundle?) {
            super.onCreate(savedInstanceState)

            // получаем сохранённую ViewModel или создаем если MainActivity впервые была создана
            viewModel = lastNonConfigurationInstance as? BookListViewModel ?: BookListViewModel()
            
            setContentView(FrameLayout(this))
        }

        override fun onRetainNonConfigurationInstance() = viewModel
        
    }

Обратите внимание, что я наследуюсь от Activity, потому что в ComponentActivity метод onRetainNonConfigurationInstance переопределяется с модификатором final.

Попробуйте аналогичным образом написать свою ViewModel.

### Реализация ViewModelStoreOwner у Fragment

Смотрим код <b>androidx.fragment.app.Fragment</b> и находим метод getViewModelStore:

     public ViewModelStore getViewModelStore() {
        ...
        
        return mFragmentManager.getViewModelStore(this);
    }

Реализация содержится во FragmentManager'е, выделим основной код:

    public abstract class FragmentManager impements ... {

        // специальная структура данных для сохранения ViewModelStore по Fragment uuid 
        // и реализующая вложенность mNonConfig друг в друга 
        private FragmentManagerViewModel mNonConfig;

        // возвращает ViewModelStore по Fragment uuid
        ViewModelStore getViewModelStore(@NonNull Fragment f) {
            return mNonConfig.getViewModelStore(f);
        }

        // возвращает вложенный mNonConfig для дочернего фрагмента
        // кейс: добавление фрагмента в другой фрагмент с помощью childFragmentManager
        private FragmentManagerViewModel getChildNonConfig(@NonNull Fragment f) {
            return mNonConfig.getChildNonConfig(f);
        }

        void attachController(@NonNull FragmentHostCallback<?> host, @NonNull FragmentContainer container, @Nullable final Fragment parent) {
           
            ...

            if (parent != null) {
                // parent это родительский фрагмент
                // если мы используем childFragmentManager и кладем туда фрагменты, 
                // то для них создаются дочерние mNonConfig
                mNonConfig = parent.mFragmentManager.getChildNonConfig(parent);
            } else if (host instanceof ViewModelStoreOwner) {
                // host это чаще всего Activity
                ViewModelStore viewModelStore = ((ViewModelStoreOwner) host).getViewModelStore();
                // если вы откроете реализацию данного метода то увидите что mNonConfig будет создан через ViewModelProvider,
                // а в качестве ViewModelStore выступает тот же самый что и в ComponentActivity
                mNonConfig = FragmentManagerViewModel.getInstance(viewModelStore);
            } else {
                mNonConfig = new FragmentManagerViewModel(false);
            }
        
        }
    
    }

Все сводится к ViewModelStore который лежит в ComponentActivity, следовательно все ViewModel'и сохраняются через onRetainNonConfigurationInstance метод.

<b>FragmentManagerViewModel</b> это всего лишь вспомогательный класс, в котором реализуется возможность построить иерархию ViewModelStore'ов для случаев когда появляются дочерние фрагменты (childFragmentManager).

Надеюсь у вас не осталось больше вопросов, а если и остались то вы знаете где найти ответы (подсказка: в исходниках).

Хорошего кода и побольше вкусностей!



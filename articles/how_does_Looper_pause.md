# Как приостанавливается главный цикл Android приложения?

О Handler и Looper ходят легенды, написано множество статей и постов, опрошено тысячи кандидатов на собесах о том как работает эта штука, но кто нибудь задавался вопросом как происходит ожидание новых событий в цикле? Если нет, но вам интересно, сейчас разберём.

Начнём с того, зачем вообще нужно приостанавливать выполнение главного цикла, можно ведь написать такой код:

```kotlin
while (true) {
    val message = messagingQueue.removeLastOrNull()
    if (message != null) {
        // обработать сообщение
    }
}
```

Проблема такого решения простая: процессор постоянно нагружен циклом и в случае когда приложение ничего не делает (messagingQueue пустой) он все равно пытается взять новое сообщение, но безуспешно (message равен null).

Давайте попробуем найти решение, а точнее подсмотреть, как это реализовано в Android. Не долго покопавшись в исходниках я нашёл примерно следующий код в MessageQueue, где происходит получение следующего сообщения и добавление нового в очередь:

```java
Message next() {
 
    for(;;) {
       
        // ждём до следующего сообщения
        nativePollOnce(ptr, nextPollTimeoutMillis);
        
    }
    
}

boolean enqueueMessage(Message msg, long when) {
    
    synchronized(this) {
        
        if (needWake) {
            // сообщаем, что нужно проснуться раньше
            nativeWake(mPtr);
        }
    }
}
```

И здесь начинается самое интересное, потому что методы приостановки/возобновления главного цикла нативные:

```java
class MessageQueue {

    private native void nativePollOnce(long ptr, int timeoutMillis);
    private native static void nativeWake(long ptr);

}
```

Но я на этом останавливаться не стал и полез внутрь C++ исходников android_os_MessageQueue.cpp (https://android.googlesource.com/platform/frameworks/base/+/master/core/jni/android_os_MessageQueue.cpp) и Looper.cpp (https://android.googlesource.com/platform/frameworks/native/+/jb-dev/libs/utils/Looper.cpp), для понимания здесь упрощенный вариант кода:

```cpp
// так в C++ объявляется конструктор класса Looper
Looper::Looper(bool allowNonCallbacks) : ... {
    int wakeFds[2];
    /* pipe это системный вызов Linux, который создает однонаправленный канал данных и записывает файловые дескрипторы чтения и записи в массив wakeFds, также позволяет обмениваться данными из разных процессов */
    int result = pipe(wakeFds);
    
    /* в Linux практически всё можно представить файловым дескриптором: файлы, TCP соединения и тд, это очень удобно при написании программ */
    mWakeReadPipeFd = wakeFds[0];
    /* теперь мы можем записать некоторые данные по этому дескриптору в конец канала данных и прочитать эти данные по дескриптору mWakeReadPipeFd */
    mWakeWritePipeFd = wakeFds[1];

    // создаётся базовый файловый дескриптор для epoll
    mEpollFd = epoll_create(EPOLL_SIZE_HINT);

    struct epoll_event eventItem;
    /* memset зануляет данные структуры eventItem, это нужно для того чтобы быть полностью уверенным что у нас нет в памяти мусора, который может привести к непредсказуемым результатам, вообще в Си/C++ такая практика часто используется */
    memset(& eventItem, 0, sizeof(epoll_event));
    eventItem.events = EPOLLIN;
    eventItem.data.fd = mWakeReadPipeFd;
    /* системный вызов epoll_ctl добавляет в мониторинг дескриптор чтения и настраивает его на событие EPOLLIN, что означает когда в mWakeReadPipeFd появятся новые данные мы должны о них узнать (паттерн наблюдатель) */
    result = epoll_ctl(mEpollFd, EPOLL_CTL_ADD, mWakeReadPipeFd, & eventItem);
}

int Looper::pollInner(int timeoutMillis) {

    struct epoll_event eventItems[EPOLL_MAX_EVENTS];
    /* Системный вызов epoll_wait останавливает текущее выполнение кода и ждёт timeoutMillis миллисекунд, при появлении данных в дескрипторе mWakeReadPipeFd срабатывает событие EPOLLIN и вызов завершается раньше, в противном случае по истечению таймаута */
    int eventCount = epoll_wait(mEpollFd, eventItems, EPOLL_MAX_EVENTS, timeoutMillis);
    
    // дальнейшая обработка событий

}

void Looper::wake() {
    ssize_t nWrite;
    do {
        /* Когда нам нужно возобновить обработку следующего события записываются новые данные в файловый дескриптор записи mWakeWritePipeFd, который как вы уже знаете является частью канала, созданного системным вызовом pipe(), после этого в файловом дескрипторе чтения mWakeReadPipeFd появляются новые данные и происходит событие EPOLLIN, вызов epoll_wait() получает его и выполнение главного цикла продолжается */
        nWrite = write(mWakeWritePipeFd, "W", 1);
    } while (nWrite == -1 && errno == EINTR);
}

/* так выглядит деструктор в C++, он нужен для очистки данных внутри объекта при его удалении, здесь зачищаются все используемые файловые дескрипторы в пределах класса Looper */
Looper::~Looper() {
    close(mWakeReadPipeFd);
    close(mWakeWritePipeFd);
    close(mEpollFd);
}
```

Всё сводится к системным вызовам epoll и работы с файловыми дескрипторами. Более подробно о Epoll можно почитать на википедии (https://ru.wikipedia.org/wiki/Epoll) или погуглить статьи/видосы на эту тему. В качестве закрепления приведу более простой пример с комментами:

```cpp
/* создаёт файловый дескриптор с которым работают системные вызовы epoll */
mEpollFd = epoll_create(EPOLL_SIZE_HINT);

/* Добавляет файловый дескриптор с определённым событием для мониторинга, как я уже говорил файловым дескриптором может быть практически что угодно - файл, TCP соединение, дескрипторы созданные системным вызовом pipe() для обмена данными и тд */
result = epoll_ctl(mEpollFd, ...);

/* Ключевой системный вызов который говорит ядру приостановить выполнение и отложить выполнение до тех пор пока не наступят определённые события в файловых дескрипторах на которые подписан epoll или не истечёт таймаут */
int eventCount = epoll_wait(mEpollFd, ...);
```

Всем хорошего кода!

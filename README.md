Benchmarking ORM используемых при создании Android-приложений
=============================================================

Привет, Хабр! Меня зовут Артём Добровинский и я Android-разработчик в [FINCH](https://www.facebook.com/FinchMoscow).

Однажды, кутаясь в дыму утренней сигары, я изучал исходники одной ORM для Android. Увидев там package под названием `benchmarks` сразу заглянул туда, и был удивлен тем, что все оценки выполнены с помощью `Log.d(System.nanoTime())`. Я видел такое не в первый раз. Если быть честнее, я видел даже бенчмарки, сделанные с помощью `System.currentTimeMillis()`. Обрушившееся осознание того, что что-то надо менять, заставило отставить в сторону бокал с виски и сесть за клавиатуру.

<cut/>

### Почему написана эта статья
Ситуация с пониманием того, как мерять производительность кода в Android — печальная.   
Сколько не рассказывай про профайлеры, а и в 2019 году кто-то остается уверенным, что JVM делает всё, что разработчик написал и именно в том порядке, в котором код написан. В реальности нет ничего более далекого от истины.   
На самом деле, несчастная виртуальная машина отбивается от миллиарда безалаберных кнопкодавов, которые пишут свой код, ни разу не напрягшись о том, как с этим всем будет работать процессор. Эта битва длится уже не первый год, и в рукаве у неё миллион хитрейших оптимизаций, которые (если их игнорировать) превратят любое измерение производительности программы в потерю времени. 

Т.е., разработчики подчас не считают необходимым мерять производительность кода, а еще чаще — не знают, как. Трудность заключается в том, что для проведения оценки производительности необходимо создать для всех кейсов максимально схожие и идеальные условия — только так можно получить полезную информацию. Создаются эти условия не написанными на коленке решениями. 

Если нужны доводы по поводу того, пользоваться ли для замера производительности сторонними фреймворками — всегда можно почитать [Алексея Шипилёва](https://shipilev.net/blog/2014/nanotrusting-nanotime/#_timers) и поразиться глубине проблемы. В статье по ссылке всё есть: и зачем нужен warmup перед проведением бенчмарка, почему `System.currentTimeMillis()` нельзя доверять вообще при подсчете прошедшего (elapsed) времени, и шутки за 300. Отличное чтиво. 

##### Почему я могу об этом рассказывать?   
Дело в том, что я всесторонне развитый разработчик: я не только владею Android SDK так, будто это мой pet-project, но еще где-то месяц писал код для бекенда.   
Когда я принес лиду свой первый микросервис на ревью, и там в `README` не было бенчмаркинга — он смотрел на меня с непониманием.   
Я запомнил это и больше никогда не повторял этой ошибки. Потому-что ушел через неделю.

Поехали.

### Что измеряем
В рамках кейса по бенчмаркингу баз данных под Android я решил померить скорость иницализации и скорость записи/чтения для таких ORM, как Paper, Hawk, Realm и Room.   
Да, я меряю в одном тесте NoSQL и реляционную БД — какой следующий вопрос?

### Чем измеряем
Казалось бы, если речь о JVM, то выбор очевиден — есть [покрытый славой](https://mvnrepository.com/tags/benchmark), [доведенный до совершенства](https://groups.google.com/d/msg/mechanical-sympathy/m4opvy4xq3U/7lY8x8SvHgwJ) и [безупречно задокументированный](http://hg.openjdk.java.net/code-tools/jmh/file/f2e982b7c51b/jmh-samples/src/main/java/org/openjdk/jmh/samples/) [JMH](https://openjdk.java.net/projects/code-tools/jmh/). Но нет, на нём не заведyтся инструментационные тесты для Android.   
Следом за ними идет [Calipher](https://github.com/google/caliper) от Google — с тем же результатом.    
Есть форк Calipher под называнием [Spanner](https://github.com/cmelchior/spanner) — который как много лет задеперкейчен и призывает пользоваться [Androidx Benchmark](https://developer.android.com/jetpack/androidx/releases/benchmark).   
Остановим внимание на последнем. Хотя бы потому, что у нас не осталось выбора.

Как и всё, что было добавлено в Jetpack, а не переосмыслено при переносе из Support Library, Androidx Benchmark выглядит и ведёт себя так, будто был написан за неделю-полторы в качестве тестового задания, и больше к нему никто никогда не притронется.   
Плюс, эта либа немного мимо — т.к., она больше для оценки UI-тестов. Но за неимением лучшего, можно работать и с ней. Это убережет нас хотя бы от [очевидных ошибок](https://android.googlesource.com/platform/frameworks/support/+/refs/heads/androidx-benchmark-release/benchmark/common/src/main/java/androidx/benchmark/Errors.kt), а также поможет с разогревом.
Для снижения смехотворности результатов я прогоню все тесты 10 раз и вычислю средний показатель.

Устройство для тестирования — Xiaomi A1. Не самое слабое на рынке, «чистый» Android.

### Подключение библиотеки в проект
По подключению Andoridx Benchmark в проект есть [отличная инструкция](https://developer.android.com/studio/profile/benchmark.md). Очень советую не полениться и подключить отдельный модуль для производства измерений. 

### Ход эксперимента
Все наши бенчмарки будут исполнятся в следующем порядке:
1. Сначала мы инициируем базу данных в теле теста.
2. Затем в блоке `benchmarkRule.scope.runWithTimingDisabled` генерим данные, которые скормим базе данных. Код, помещенный в это замыкание не будет учитываться при оценке.
3. В это же замыкание добавляем логику очищения БД; убеждаемся, что база данных пуста перед записью. 
4. Далее следует логика записи и чтения. Обязательно ицициализируем переменную с результатом чтения, чтобы JVM не удалил эту логику из подсчета исполнения, как неиспользуемую.
5. Замеряем производительность инициализации БД в отдельной функции.
6. Чувствуем себя человеком науки.

Код можно посмотреть [здесь](https://github.com/dobrowins/androiddbbenchmarks/tree/master/tests/src/androidTest/java/com/dobrowins/dbbenchmarking/tests). Если лениво ходить, функция с замером для PaperDb выглядит так:
```kotlin
@Test
fun paperdbInsertReadTest() = benchmarkRule.measureRepeated {
    // чистим базу (это время не учитывается в оценку)
    benchmarkRule.scope.runWithTimingDisabled {
        Paper.book().destroy()
        if (Paper.book().allKeys.isNotEmpty()) throw RuntimeException()
    }
    // пишем и читаем
    repository.store(persons, { list -> Paper.book().write(KEY_CONTACTS, list) })
    val persons = repository.read { Paper.book().read<List<Person>>(KEY_CONTACTS, emptyList()) }
}
```
Бенчмарки для остальных ORM выглядят схожим образом.

#### Результаты
##### Инициализация
| название теста   | mean         | 1          | 2          | 3          | 4          | 5          | 6          | 7          | 8          | 9          | 10         |
|------------------|--------------|------------|------------|------------|------------|------------|------------|------------|------------|------------|------------|
| HawkInitTest     | 49_512        | 49_282      | 50_021      | 49_119      | 50_145      | 49_970      | 50_047      | 46_649      | 50_230      | 49_863      | 49_794      |
| PaperdbInitTest  | 224          | 223        | 223        | 223        | 233        | 223        | 223        | 223        | 223        | 223        | 223        |
| RealmInitTest    | 218          | 217        | 217        | 217        | 217        | 217        | 217        | 217        | 227        | 217        | 217        |
| RoomInitTest     | 61_695.5      | 63_450      | 59_714      | 58_527      | 59_175      | 63_544      | 62_980      | 63_252      | 59_670      | 63_868      | 62_775      |

Победитель — Realm, на втором месте Paper. Чем занимается Room еще можно представить, что почти столько же времени делает Hawk — абсолютно непонятно.

##### Запись и чтение
| название теста        | mean            | 1          | 2          | 3          | 4          | 5          | 6          | 7          | 8          | 9          | 10         |
|-----------------------|-----------------|--------------|--------------|--------------|--------------|--------------|------------|------------|------------|------------|------------|
| HawkInsertReadTest    |   278_736_469.2 | 278_098_654  | 283_956_846  | 276_748_308  | 282_447_384  | 272_609_500  | 284_699_653  | 271_869_770  | 278_719_693  | 278_836_115  | 279_378_769  |
| PaperdbInsertReadTest |   173_519_957.3 | 172_953_347  | 174_702_000  | 169_740_846  | 174_401_192  | 173_930_037  | 174_179_616  | 173_937_460  | 173_739_115  | 176_215_038  | 171_400_922  |
| RealmInsertReadTest   |   111_644_042.3 | 108_501_578  | 110_616_078  | 102_056_461  | 112_946_577  | 111_701_231  | 114_922_962  | 106_198_000  | 118_742_498  | 120_888_230  | 109_866_808  |
| RoomInsertReadTest    | 1_863_499_483.3 | 187_250_3614 | 1_837_078_614 | 1_872_482_538 | 1_827_338_460 | 1_869_147_999 | 1_857_126_229 | 1_842_427_537 | 1_870_630_652 | 1_878_862_538 | 1_907_396_652 |

Тут опять победитель Realm, но в этих результатах попахивает провалом.   
Разница в четыре раза между двумя самыми «медленными» базами данных и в шестнадцать раз между самой «быстрой» и самой «медленной» — очень подозрительна. Даже с учетом того, что разница держится стабильно.

### Заключение

Измерять производительность своего кода стоит хотя бы из любопытства. Даже если речь идёт о самых запущенных индустрией случаях (таких, как оценка инструментальных тестов под Android).   
Есть все причины привлекать для этого дела сторонние фреймворки (а не писать свой с таймингом и чирлидершами).    
Ситуация в кодовых базах такая, что все пытаются писать в чистой архитектуре, у большинства модуль с бизнес-логикой является java-модулем — подключить рядом модуль c JMH и проверять код на наличие бутылочных горлышек — работы на день. А пользы — на много лет вперед.

Happy coding!

PS: Если внимательный читатель знает о фреймворке для проведения бенчмарков инструментальных тестов под Android, не перечисленном в статье — пожалуйста, поделись в комментариях.   
PPS: [Репозиторий](https://github.com/dobrowins/androiddbbenchmarks/) с тестами открыт для пулл-реквестов.

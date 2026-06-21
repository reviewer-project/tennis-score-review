[MonYamau/tennis-match-scoreboard](https://github.com/MonYamau/tennis-match-scoreboard)

## ХОРОШО

1. Подсчёт очков вынесен в доменную модель: `Point`, `DefaultGame`, `TieBreakGame` и `GameMode` отделяют обычный гейм от тай-брейка, а счёт гейма выражен enum'ом, а не магическими числами.

2. Иерархия `TennisMatch` → `TennisSet` → `GameMode` последовательно пробрасывает победу очка снизу вверх и создаёт новый сет после завершения предыдущего.

3. JPA-сущности отделены от домена: `entity` содержит persistence-модель, `domain` — правила тенниса без зависимостей от Servlet API и Hibernate.

4. На сущности `Match` добавлен `@Check`, который запрещает матч с одинаковыми игроками и победителя вне участников — это дублирует бизнес-правила на уровне БД.

5. `HibernateMatchDao` загружает игроков через `JOIN FETCH` и применяет `setFirstResult`/`setMaxResults` в HQL, поэтому список завершённых матчей не страдает от N+1 и не вытягивает всю таблицу ради пагинации.

6. `AppContextListener` один раз собирает `SessionFactory`, Redis-клиент, DAO и сервисы, кладёт их в `ServletContext` и закрывает ресурсы в `contextDestroyed`.

7. Сервлеты тонкие: читают параметры, вызывают сервис, кладут DTO в request и делают forward или redirect. Подсчёт очков и работа с Hibernate в них не размазаны.

8. Юнит-тесты проверяют доменную логику изолированно — deuce, победа с 40:0, активация тай-брейка при 6:6, победа в сете и матче.

## ЗАМЕЧАНИЯ

### `com.project.controller`

1. В `MatchScoreServlet#getRequestDtoForPostMethod()` параметр `winnerId` не валидируется явно

`Integer.parseInt(req.getParameter("winnerId"))` при отсутствии или нечисловом значении бросает `NumberFormatException`. `IllegalArgumentException` из `OngoingMatch#recalculateScoreFor` при чужом ID тоже не переводится в `IncorrectInputException`. Оба случая попадают в общий обработчик и отдают 500.

**Рекомендация:**

Проверяй наличие `winnerId`, разбирай число через безопасный метод из `BaseServlet` в котором уже есть `convertNumber`

2. В `BaseServlet#getNormalizedUuid()` неверный формат UUID не обрабатывается

`UUID.fromString` при некорректной строке бросает `IllegalArgumentException`, которая не перехватывается фильтром как ошибка ввода.

**Рекомендация:**

Оберни `UUID.fromString` в try/catch и бросай `IncorrectInputException` с сообщением о неверном формате UUID. Или добавь в ExceptionFilter обработку IllegalArgumentException 

3. В `BaseServlet` у `getNormalizedUuid`, `getNormalizedPage` и `getNormalizedLimit` лишний аргумент `parameter`

Методы принимают имя query-параметра вторым аргументом, но вызываются только с одним фиксированным значением: `"uuid"`, `"page"`, `"limit"`. 

`getNormalizedPage` и `getNormalizedLimit` полностью повторяют друг друга: читают строку, подставляют дефолт при `null`, парсят через `convertNumber`, валидируют через `validateNaturalNumber`. Отличаются только имя параметра и значение по умолчанию.

**Рекомендация:**

Схлопни `getNormalizedPage` и `getNormalizedLimit` в один приватный метод вроде `getNaturalNumberParameter(req, name, defaultValue)`. У `getNormalizedUuid` убери второй аргумент и зашей `"uuid"` внутри метода. Для `getNormalizedName` аргумент оставь — там реально два разных поля (`firstPlayer`, `secondPlayer`).

### `com.project.dao`

1. В DAO каждая операция открывает отдельную транзакцию

`HibernatePlayerDao` и `HibernateMatchDao` в каждом методе вызывают `getCurrentSession()`, `beginTransaction()` и `commit()`. Сценарий регистрации матча делает `findByName` и `save` для каждого игрока в разных транзакциях. При одновременном создании игрока с одним именем оба потока могут не найти запись, оба вызовут `save`, и второй получит необработанное нарушение unique constraint.

**Рекомендация:**

Оберни сценарий «найти или создать двух игроков» в одну транзакцию на уровне `MatchRegistrationService`, открывая сессию один раз и передавая её в DAO либо выделив для этого один метод репозитория.

### `com.project.service`

1. В `MatchCompletionService#finishMatch()` матч удаляется из хранилища до сохранения в БД

Первой строкой вызывается `matchStorage.delete(matchDto.uuid())`, затем идёт поиск игроков и `matchDao.save`. Если сохранение в H2 упадёт, завершённый матч уже удалён из Redis и восстановить его из текущих матчей нельзя.

**Рекомендация:**

Сначала сохрани матч в БД в одной транзакции, и только после успешного `commit` удаляй запись из `MatchStorage`.

```java
public void finishMatch(OngoingMatchDto matchDto) {
    Optional<Player> firstPlayer = playerDao.findByName(matchDto.firstPlayerName());
    Optional<Player> secondPlayer = playerDao.findByName(matchDto.secondPlayerName());
    if (firstPlayer.isEmpty() || secondPlayer.isEmpty()) {
        throw new IllegalStateException("Couldn't find the player to save");
    }
    Player winner = checkWinner(matchDto.winnerId(), firstPlayer.get(), secondPlayer.get());
    Match finishedMatch = new Match(firstPlayer.get(), secondPlayer.get(), winner);
    matchDao.save(finishedMatch).orElseThrow(() -> new IllegalStateException("Couldn't save the finished match"));
    matchStorage.delete(matchDto.uuid());
}
```

2. В `MatchScoreService#recalculateMatch()` нет атомарного обновления текущего матча

Метод читает матч из Redis, восстанавливает его из DTO, меняет счёт и записывает обратно. Два параллельных POST по одному UUID прочитают одно состояние, каждый начислит своё очко, и победит последняя запись — одно очко теряется. 
После завершения матча гонка может снова записать «живой» матч в Redis, если второй поток работал со stale-копией.

**Рекомендация:**

Инкапсулируй операцию «прочитать → начислить → сохранить» в `MatchStorage` с блокировкой по UUID: для Redis используй `WATCH`/`MULTI`/`EXEC`, который обновляет JSON только если ключ не изменился.

```java
public OngoingMatchDto recalculateMatch(OngoingMatchRequestDto requestDto) {
    return matchStorage.update(requestDto.uuid(), match -> {
        match.recalculateScoreFor(requestDto.winnerId());
        return mapper.toDto(match);
    });
}
```

3. В `MatchScoreService#recalculateMatch()` матч проходит лишний цикл DTO → модель → DTO

После `getMatch` ты уже получил DTO, затем через `mapper.toModel(dto)` собираешь `OngoingMatch` заново вместо работы с объектом из хранилища. Это не ломает правила счёта, но усложняет атомарное обновление и маскирует гонки: каждый запрос оперирует копией, а не живым экземпляром из storage.

**Рекомендация:**

Добавь в `MatchStorage` метод обновления in-place и вызывай `recalculateScoreFor` на объекте, полученном из `find`, без промежуточного маппинга в DTO.

4. В `MatchRegistrationService#savePlayer()` нет обработки гонки при создании игрока

Метод «сначала find, потом save» в двух независимых транзакциях. При параллельной регистрации игроков с одним именем второй `save` получит ошибку уникальности и пользователь увидит 500 вместо повторного использования уже созданного игрока.

**Рекомендация:**

После `PersistenceException` на unique constraint повтори `findByName` в той же транзакции регистрации матча и используй найденного игрока.

### `com.project.storage`

1. В `RedisMatchStorage` операции read-modify-write не синхронизированы

`save` и `find` работают как отдельные команды без версионирования. Это хранилище текущих матчей общее для всех потоков сервлета, поэтому одного `ConcurrentHashMap` здесь нет — нужна атомарность на уровне Redis-клиента.

**Рекомендация:**

Реализуй в `RedisMatchStorage` метод `update(UUID id, UnaryOperator<OngoingMatch> updater)` с optimistic locking через Redis `WATCH`/`MULTI` и вызывай его из `MatchScoreService`.

### `tests`

1. Нет теста на возврат с advantage к deuce

В `DefaultGame#checkDeuce` реализован сброс при `ADVANTAGE:ADVANTAGE`, но в `DefaultGameTest` этот переход не покрыт. Регрессия в deuce-логике останется незамеченной.

**Рекомендация:**

Добавь тест: при счёте `ADVANTAGE:FORTY` очко второго игрока возвращает пару к `FORTY:FORTY` без завершения гейма.

2. Нет теста на запрет начисления очков завершённому матчу

`OngoingMatch#checkMatchState` бросает исключение при повторном очке, но это не проверено тестом на уровне домена.

**Рекомендация:**

Добавь в тесты `OngoingMatch` сценарий с уже установленным `winnerId` и ожидай `IllegalStateException` при вызове `recalculateScoreFor`.

3. Нет тестов сервисного слоя и DAO

Покрыта только доменная логика счёта. Пагинация, фильтрация, порядок удаления/сохранения завершённого матча и маппинг DTO не проверяются автоматически.

**Рекомендация:**

Добавь хотя бы интеграционный тест для `HibernateMatchDao` на in-memory H2 и unit-тест для `MatchCompletionService`, который проверяет, что `delete` не вызывается при ошибке сохранения.

## РЕКОМЕНДАЦИИ

1. Доведи операции с текущими матчами до атомарности: одна точка входа в storage, блокировка по UUID, удаление из хранилища только после успешной записи в БД.

2. Собирай связанные DAO-операции одного пользовательского сценария в одну Hibernate-транзакцию, начиная с регистрации игроков и сохранения завершённого матча.

3. Унифицируй обработку ошибок ввода на всех сервлетах: неверный UUID, чужой `winnerId`, отсутствующие параметры должны возвращать 400 с понятным сообщением, а не 500.

5. Расширь тесты граничными сценариями deuce и завершённого матча, затем добавь интеграционные проверки для persistence-слоя.

## ИТОГ

Проект в целом на хорошем уровне для учебной работы: стек соответствует ТЗ, слои разделены осмысленно, подсчёт очков живёт в доменной модели, а не в сервлетах, сервисы и DAO не смешаны с HTTP-деталями. Отдельно радует аккуратное разделение обычного гейма и тай-брейка и осмысленные юнит-тесты на ключевые теннисные переходы.

Слабые места сосредоточены вокруг жизненного цикла текущего матча и persistence. Удаление из Redis до commit в H2 может безвозвратно потерять завершённую игру, а read-modify-write без синхронизации теряет очки при параллельных запросах. DAO работает в режиме «сессия на операцию», что создаёт гонки при создании игроков. Эти проблемы важнее косметики в пакетах и стоит закрыть их в первую очередь.

Подсчёт счёта реализован надёжно: deuce, advantage, переход к тай-брейку при 6:6, победа в сете и матче до двух сетов соответствуют правилам проекта. Граница «очко после завершения» есть в `OngoingMatch`, но её стоит подкрепить тестами.

Границы MVCS в основном соблюдены: сервлеты тонкие, в JSP идут DTO, entity не протекают во view. Доработки нужны в валидации HTTP-параметров, экранировании JSP и атомарности storage. После этого решение будет уверенно работать и на Tomcat с H2 без обязательного Redis.

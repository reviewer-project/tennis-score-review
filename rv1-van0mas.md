https://github.com/van0mas/tennis-score[van0mas]

# НЕДОСТАТКИ РЕАЛИЗАЦИИ
1. После завершения игры (1 сет 5 игр 40 очков) резко перекидывает на страницу с результатами, хотелось бы видеть 2/0/0 и предложить новую игру/список матчей
2. Сильно много филлер-матчей в matches
3. Нет просмотра активных игр
   
# ХОРОШО
1. Работает
2. Тесты (но мало)

# ЗАМЕЧАНИЯ
1. Где-то используешь рекорды, где-то нет. 1 из примеров:
```java
@Data
public class ActiveMatch {
```

```java
public record MatchPointRequestDTO (String matchID, String winnerID) {}
```
2. ServiceFactory
Ты называешься класс Factory (фабрика) он подрузумевает что при обращении к нему, будет создаваться новый инстанс, а на самом деле, у тебя там лежат статик классы, лучше это назвать что-то типо ServiceProvider
3. Бардак в пакетах
У тебя есть пакет domain в котором лежит 1 репозитори и 2 дто, а так же repository и dto. Надо навести порядок
4. Где-то используешь lombok, где-то нет
Почему не @AllArgsConstructor?
```java
   public class MatchService {

    private final PlayerRepository playerRepository;
    private final ActiveMatchRepository activeMatchRepository;

    public MatchService(ActiveMatchRepository matchRepository, PlayerRepository playerRepository) {
        this.activeMatchRepository = matchRepository;
        this.playerRepository = playerRepository;
    }
```
5. Игнорирование подсказок Intelij IDEA
Class can be record class
```java
@Data
@AllArgsConstructor
public class ActiveMatchResponseDTO {
    private final Player player1;
    private final Player player2;
    private final Score score;
    private final String pointsPlayer1Display;
    private final String pointsPlayer2Display;
}
```
6. В папке exception хорошо было бы поделить эксепшены на бизнес-ошибки и домейн-ошибки
7. Кастомные костыли
Зачем нужен ```private static String parseUUID(String matchId) {``` если можно ```UUID.fromString()``` который вернет эксепшн, и будешь сразу работать с UUID, а не с строкой
Тогда выкинуть сразу MatchPointValidator можно

9. Обфускация по заводу (непонятный нейминг)
Почему не назвать это matchRequestDto/matchUuid?

MatchPointRequestDTO dto = MatchPointRequestMapper.from(request);
UUID uuid = UUID.fromString(dto.matchID());

10. Комментарии
Коментарии тут не несут полезной нагрузки, а констатируют очевидное
```java
if (!MatchPointValidator.isValid(dto)) {
            // невалидный winnerID
            response.sendRedirect(request.getContextPath() + Paths.MATCH_SCORE + "?uuid=" + dto.matchID());
            return;
        }
```

```java
// Если оба игрока имеют по 6 геймов, начинается тай-брейк
            if (score.getGamesPlayer1() == 6 && score.getGamesPlayer2() == 6) {
                score.setTieBreak(true);
            }
```

Лучше джавадок
```java
// ==== URL маршруты ====
public static final String MATCHES = "/matches";
    public static final String MATCH_SCORE = "/match-score";
    public static final String NEW_MATCH = "/new-match";

```
11. Есть баг в PaginationService
Возвращаем на 1 страницу больше, чем надо, есть пустая страница ```int totalPages = (int) (totalMatches / pageSize) + 1;```

# АРХИТЕКТУРА
Несмотря на то, что в проекте несколько классов, он написан в смешанном стиле.

Программа в ООП стиле должна быть декомпозирована по правилам ООП и использовать ООП подход для реализации типичных задач. Вроде есть попытки на ооп, но все же больше процедурного кода

# ВЫВОД
Проект выполнен неплохо, функционал рабочий, есть замечания по 

### Субъективные комментарии
1. DTO капсом в нейминге не красиво. Гораздо лучше Dto

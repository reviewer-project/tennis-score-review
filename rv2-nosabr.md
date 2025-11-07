https://github.com/nosabr/tennis-score
[nosabr]

# Замечания
1. Сама идея же подсказывает использовать try-with-resources, почему не используешь?
   ```java
       public Optional<Match> findById(Long id) {
        Session session = HibernateSessionFactoryUtil.getSessionFactory().openSession();
        Match match = session.find(Match.class, id);
        //session.close();
        return Optional.ofNullable(match);
    }
   ```
2. Названия таблиц принято называть маленькими буквами через underscore
  ```java
@Entity
@Table(name = "MATCHES")
@Data
public class Match {
```

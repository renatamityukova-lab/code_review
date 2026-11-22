# Ревью: `code_review/ex2.txt` (Java, MailService)

Фрагмент: `sendWelcomeEmail(Long userId)` → `userRepository.findById(userId)` → `mailClient.send(user.getEmail(), "Welcome!")`.
Фокус: риск `NullPointerException` и неявная обработка ошибок. Итог: **2 blocker, 2 major, 1 minor**.

| # | Проблема | Приоритет | До merge | Отдельная задача |
|---|----------|-----------|----------|------------------|
| 1 | NPE при `user == null` | blocker | да | нет |
| 2 | `email == null/blank` → NPE / письмо в никуда | blocker | да | нет |
| 3 | Нет проверки `userId` (null) | major | да | нет |
| 4 | Неявная логика ошибок: `void`, нет сигнала «не найден» | major | да | да (контракт API/исключения) |
| 5 | Зависимости не валидируются в конструкторе | minor | желательно | нет |

---

## 1. NPE при `user == null` — **blocker**

- **Что не так:** репозиторий может вернуть `null`, а результат сразу разыменовывается в `user.getEmail()`.
- **Риски:** `NullPointerException` вместо понятного «пользователь не найден»; падение сценария отправки приветствия, шумные стектрейсы без контекста.
- **Как исправить:** явная проверка после `findById` — либо `throw new UserNotFoundException(userId)`, либо `return` с логом; лучше перевести репозиторий на `Optional<User>`.

**Next steps:** до merge — обязательно; отдельная задача — нет; тесты — «пользователь не найден» → доменное исключение (или явный no-op), без NPE.

---

## 2. `email == null / blank` — **blocker**

- **Что не так:** `user.getEmail()` не проверяется; контракт `mailClient.send` на `null`-получателя не определён.
- **Риски:** NPE внутри почтового клиента или попытка отправки на пустой адрес; ошибка проявляется в глубине чужого слоя, а не в месте вызова.
- **Как исправить:** проверять `email == null || email.isBlank()` и падать с явной ошибкой (`IllegalStateException`) либо пропускать с логом; валидировать формат адреса до вызова клиента.

**Next steps:** до merge — обязательно; отдельная задача — нет; тесты — `email = null`, `""`, `"   "` → предсказуемая ошибка/скип; валидный адрес → `send` вызван с ним.

---

## 3. Нет проверки `userId` — **major**

- **Что не так:** `userId` принимается как есть; поведение `findById(null)` не задано кодом и зависит от реализации репозитория.
- **Риски:** неявная семантика: где-то `IllegalArgumentException`, где-то `null`, где-то запрос по `null` — непредсказуемо и трудно тестировать.
- **Как исправить:** `Objects.requireNonNull(userId, "userId")` (или `@NonNull` + валидация на границе) до обращения к репозиторию.

**Next steps:** до merge — обязательно (одна строка); отдельная задача — нет; тесты — `userId = null` → `IllegalArgumentException`/`NullPointerException` с понятным сообщением, репозиторий не вызывается.

---

## 4. Неявная логика ошибок: `void` без сигнала о результате — **major**

- **Что не так:** метод `void`, «пользователь не найден» никак не отличается от «письмо отправлено»; нет ни исключения, ни лога, ни метрики.
- **Риски:** вызывающий код не может отличить успех от пропуска; ошибки «молчат» до первого NPE; невозможно ретраить/диагностировать инциденты.
- **Как исправить:** зафиксировать контракт — либо доменное исключение (`UserNotFoundException`, `EmailDeliveryException`), либо результат (`boolean`/`Result`); добавить логирование факта отправки и причины пропуска.

**Next steps:** до merge — обязательно (выбрать и закрепить контракт); отдельная задача — да (согласовать единый подход к ошибкам в сервисах и маппинг в HTTP-ответы); тесты — успех, «не найден», ошибка `mailClient.send` → различимые исходы.

---

## 5. Зависимости не валидируются в конструкторе — **minor**

- **Что не так:** `userRepository` и `mailClient` не проверяются на `null`.
- **Риски:** при неверной конфигурации NPE всплывает в момент отправки письма, а не на старте приложения — позже и дороже в диагностике.
- **Как исправить:** `Objects.requireNonNull` в конструкторе (fail-fast).

**Next steps:** до merge — желательно (дёшево); отдельная задача — нет; тесты — конструктор с `null` → `NullPointerException` сразу.

---

## Эталонный фикс

```java
public final class MailService {
    private final UserRepository userRepository;
    private final MailClient mailClient;

    public MailService(UserRepository userRepository, MailClient mailClient) {
        this.userRepository = Objects.requireNonNull(userRepository, "userRepository");
        this.mailClient = Objects.requireNonNull(mailClient, "mailClient");
    }

    public void sendWelcomeEmail(Long userId) {
        Objects.requireNonNull(userId, "userId");

        User user = userRepository.findById(userId);
        if (user == null) {
            throw new UserNotFoundException(userId);   // явный контракт вместо NPE
        }

        String email = user.getEmail();
        if (email == null || email.isBlank()) {
            throw new IllegalStateException("User " + userId + " has no email");
        }

        mailClient.send(email, "Welcome!");
    }
}
```

**Тесты (минимум):** `user == null` → `UserNotFoundException`; `userId == null` → валидация до репозитория; `email == null/blank` → явная ошибка; happy path → `send` вызван с корректным адресом; ошибка `mailClient.send` → пробрасывается как доменная/логируется.

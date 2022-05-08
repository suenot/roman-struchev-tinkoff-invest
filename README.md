# Конфигурация
##### Tinkoff API
```properties
tinkoff.token - токен для Tinkoff GRPC API
tinkoff.account-id - ID счета в Tinkoff (опционально, будет выбран первый счет, если не указано)
tinkoff.is-token-sandbox - true/false (по типу токена)
tinkoff.emulator - true/false (эмуляция запросов в tinkoff для выполнения ордеров)
```
##### Telegram API (уведомления о сделка, ошибках)
```properties
telegram.bot.token: токен телеграм бота
telegram.bot.chat-id: id чата, будет отправлен в чат, если написать боту любое сообщение
```

# Торговые стратегии
### 1. Покупка инструмента (ценной бумаги) за другой инструмент (ценную бумагу)
##### Описание
Стратегии торговли, зарабатывающие на изменении стоимости торговых инструментов, относительно друг друга. В рамках одной стратегии должно быть не менее 2х инструментов.
##### Мотивация
Используется как стратегия для перекладывания средств между валютами на московской бирже из-за периодического открытия позиций покупки/продажи конкретной валюты крупными игроками (продажа валюты экспортерами, покупка валюты ЦБ и т.д.) и отсутствия открытого рынка. 
Валюты, которые ранее не были ранее волатильны относительно друг друга, на московской бирже стали волатильны.

Пример: Стоимость USD, EUR, CNY на московской бирже за RUB. USD может вырасти относительно RUB в течении дня, EUR при этом вырастет на меньший процент или даже станет дешевле, на следующий день ситуация изменится в обратную сторону. Соотв, в первый день нужно купить EUR за USD, а во второй USD за EUR, в среднем пересчете по любой из этих валют, будет профит.

Продажа текущего инструмента (1) и соотв. покупка другого из стратегии происходит при увеличении цены текущего относительно других на определенный процент (0.5% по умолчанию). Но это не значит, что данные инструменты стали дороже или дешевле относительно RUB.
##### Конфигурация стратегии
Пример стратегии, перекладывание EUR <-> USD при изменении стоимости одной из валюты на 0.5% относительно другой относительно другой (есть в проекте, **запушено на продакшене**)
```java
public class EURByCNYStrategy extends AInstrumentByInstrumentStrategy {

    private Map FIGIES = Map.of(
            "BBG0013HRTL0", 6000, // CNY
            "BBG0013HJJ31", 1000 // EUR
    );

    public Map<String, Integer> getFigies() {
        return FIGIES;
    }

    @Override
    public boolean isEnabled() {
        return true;
    }
}
```
##### Атрибуты конфигурации 
- `AInstrumentByInstrumentStrategy.getMinimalDropPercent` - Процент падения стоимости одного из инструментов в стратегии относительно инструмента, принадлежащего нам. Осуществляется операция продажи/покупки при достижении этого значения
##### Расположение
- live/sandbox: `/src/main/java/com/struchev/invest/strategy/instrument_by_instrument/`
- tests: `/src/test/java/com/struchev/invest/strategy/instrument_by_instrument/`

### 2. Покупка инструмента (ценной бумаги) за фиат (RUB, USD, EUR, ...)
##### Описание
Стратегии торговли, зарабатывающие на изменении стоимости торгового инструмента относительно фиатной валюты. В рамках одной стратегии может быть любое кол-во инструментов.
##### Мотивация
Классическая покупка/продажа инструмента на основе критериев и индикаторов.
##### Конфигурация стратегии
Пример стратегии, торгуем двумя акциями Сбербанка, покупаем при цене меньше 40% значения за последние 7 дней, продаем при получении дохода в 1% (take profit) либо убытка в 3% (stop loss) (есть в проекте, **запушено на продакшене**)
```java
@Component
public class BuyP40AndTP1PercentAndSL3PercentStrategy extends AInstrumentByFiatStrategy {

    private Map FIGIES = Map.of(
            "BBG004730N88", 10    // SBER
    );

    public Map<String, Integer> getFigies() {
        return FIGIES;
    }

    @Override
    public AInstrumentByFiatStrategy.BuyCriteria getBuyCriteria() {
        return AInstrumentByFiatStrategy.BuyCriteria.builder().lessThenPercentile(40).build();
    }

    @Override
    public AInstrumentByFiatStrategy.SellCriteria getSellCriteria() {
        return AInstrumentByFiatStrategy.SellCriteria.builder().takeProfitPercent(1f).stopLossPercent(3f).build();
    }

    @Override
    public boolean isEnabled() {
        return true;
    }
}
```
##### Атрибуты конфигурации:
- `AInstrumentByFiatStrategy.getHistoryDuration` - gериод истории котировок, для расчета процента (перцентиля) для текущей цены относительно истории
- `AInstrumentByFiatStrategy.getBuyCriteria().lessThenPercentile` - Процент (перцентиль), если цена за указанный период падает ниже него, покупаем
- `AInstrumentByFiatStrategy.getSellCriteria().takeProfitPercent` - Процент (take profit), если цена за указанный период падает на него, продаем
- `AInstrumentByFiatStrategy.getSellCriteria().takeProfitPercentile` - Процент (take profit, перцентиль), если цена за указанный период падает ниже него, продаем
- `AInstrumentByFiatStrategy.getSellCriteria().stopLossPercent` - Процент (stop loss), если цена за указанный период падает на него, продаем
- `AInstrumentByFiatStrategy.getSellCriteria().stopLossPercentile` - Процент (stop loss, перцентиль), если цена за указанный период падает ниже него, продаем
- `AInstrumentByFiatStrategy.getDelayBySL` - Период паузы в торговле, если продали по stop loss критерию

##### Расположение
- live/sandbox: `/src/main/java/com/struchev/invest/strategy/instrument_by_fiat/`
- tests: `/src/test/java/com/struchev/invest/strategy/instrument_by_fiat/`
    

# Запуск приложения (live/sandbox режим)
Стратегии находятся в `/src/main/java/com/struchev/invest/strategy/`
##### Используя docker-compose
Требуется docker и docker-compose
1. Собрать docker образ локально (опционально, т.к. уже есть собранный https://hub.docker.com/repository/docker/romanew/invest)
    ```shell
    docker build -t romanew/invest:latest -f Dockerfile.App.
    ```
2. Запустить контейнеры приложения и БД через docker-compose, обновив конфигурацию (свойства) в `docker-compose-image-app.yml`
    ```shell
    docker-compose -f docker-compose-image-app.yml down
    docker-compose -f docker-compose-image-app.yml pull
    docker-compose -f docker-compose-image-app.yml up
    ```
3. В консоле будет лог операций, статистика по адресу http://localhost:10000

##### Используя gradlew
Требуется jdk 11+
1. Запустить posgresql
2. Скомпилировать и запустить приложение, указав один из профилей и обновив конфигурацию (свойства) в нем. Профили `src/main/resources/application-*.properties`.
    ```shell
    ./gradlew bootRun -Dspring.profiles.active=sandbox
    ```
3. В консоле будет лог операций, статистика по адресу http://localhost:10000

# Тестирование стратегий по историческим данным
Стратегии находятся в `/src/test/java/com/struchev/invest/strategy/`
##### Используя docker-compose
Требуется docker и docker-compose
1. Собрать docker образ локально (опционально, т.к. уже есть собранный https://hub.docker.com/repository/docker/romanew/invest)
    ```shell
    docker build -t romanew/invest-tests:latest -f Dockerfile.Tests.
    ```
2. Запустить контейнеры приложения и БД через docker-compose, обновив конфигурацию (свойства) в `docker-compose-image-tests.yml`
    ```shell
    docker-compose -f docker-compose-image-tests.yml down
    docker-compose -f docker-compose-image-tests.yml pull
    docker-compose -f docker-compose-image-tests.yml up
    ```
3. В консоле будет лог операций и результат

##### Используя gradlew
Требуется jdk 11+

1. Скомпилировать и запустить тесты, обновив свойства в `src/test/resources/application-test.properties`.
```shell
./gradlew clean test --info
```
2. В консоле будет лог операций и результат


# CI/CD
При сборке в github docker образ приложения автоматически публигуется в https://hub.docker.com/repository/docker/romanew/invest.
Экземпляр приложения автоматически разворачивается на сервере.

На данный момент торгуются несколько стратегий, подробнее можно ознакомиться на http://invest.struchev.site

# Мониторинг приложения
Подключены Spring Actuator и JavaMelody. По умолчанию доступны:
- /actuator/health
- /actuator/metrics 
- /actuator/monitoring
- /actuator/logfile
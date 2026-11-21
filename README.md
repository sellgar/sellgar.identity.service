# sellgar.identity.service

`sellgar.identity.service` - backend service, который владеет identity/auth/session контуром Sellgar.

Сервис принимает команды через RabbitMQ, работает с PostgreSQL и отвечает за:

- пользователей;
- персональные данные пользователя;
- логин;
- refresh/access token lifecycle;
- session lifecycle;
- восстановление и удаление сессий.

## Зона ответственности

Identity service владеет доменной моделью identity и временем жизни session/token данных.

Он не должен владеть:

- HTTP cookie transport admin gateway;
- UI/BFF DTO для admin frontend;
- MinIO/object storage;
- установкой RabbitMQ в операционной системе;
- общей dev-инфрой workspace.

Gateway может хранить transport cookie, но валидность сессии и token lifecycle остаются в этом сервисе.

## Структура и запуск

```text
code/                   Nest source, package.json, Yarn lock, .env.example
Dockerfile              Node 24 build/test and non-root production image
.github/                image CI, release button and isolated smoke
helm/values.app.yaml    common application values and Secret references
helm/dev/values.yaml    development .env ConfigMap, one replica
helm/prod/values.yaml   production overrides (not deployed yet)
```

Из `code/`:

```bash
cp .env.example .env
node .yarn/releases/yarn-4.18.0.cjs install --immutable
node .yarn/releases/yarn-4.18.0.cjs test --runInBand
node .yarn/releases/yarn-4.18.0.cjs build:identity_srv
node node_modules/typeorm/cli.js -d dist/data-source.js migration:run
node .yarn/releases/yarn-4.18.0.cjs start:identity_srv.prod
```

Необходимы PostgreSQL и RabbitMQ. Доменные команды остаются RMQ; внутренний
`GET /health` на PORT (default 8080) проверяет базу и RMQ consumer. Отключение
брокера/базы возвращает 503. Внешнего HTTP API/Ingress у сервиса нет.

## CI и доставка

Push в `development` запускает build → publish → параллельно test/migrate → deploy.
`test` — отдельная job с unit/компонентными тестами из Docker target `tests`.
Если тесты или миграции не прошли, deploy не запускается.
Сборка, тесты и публикация выполняются на GitHub-hosted runners;
migrate/deploy — на repository-scoped runner в dev VM.
Обычная раскатка выполняется Helm с ожиданием готовности и проверкой image digest.
Argo CD продолжает управлять платформой, но не этим приложением.

Docker targets: `application`, `tests`, `migration`.
Миграционный образ запускает TypeORM CLI отдельной Kubernetes Job до раскатки приложения. В runtime приложения миграции автоматически не запускаются.

Настройки среды хранятся в `helm/dev/values.yaml`, Secret refs — в
`helm/values.app.yaml`. Версионный immutable ConfigMap монтируется через
`environmentFile`; значения credentials не передаются в GitHub-hosted jobs или Git.
Внутренний HTTP-порт — 8080, в dev одна реплика.

PR запускает build/test без публикации и деплоя. Push в `release/*` публикует
кандидат с `release.json`; ручной production workflow из `master` принимает
`candidate-<run-number>.<attempt>`, проверяет успешный исходный CI и идентичность
дерева, продвигает те же digest без пересборки. Production ещё не подготовлен:
нужны его values, Secrets, runner и GitHub environment protections.
Откат миграций в эту схему не включён.

Общие workflows закреплены тегом `delivery-v0.6.0` в `sellgar.infrastructure`.
Порядок установки runner и передачи управления: [доставка приложений](https://github.com/pyatakovvg/sellgar.infrastructure/blob/development/docs/application-delivery.md).

## Session context посетителя

Session issue/renew и active verify дополнительно возвращают ownerUuid и expiresAt
(ISO timestamp). Identity — единственный владелец этого соответствия: сейчас для
аккаунта ownerUuid=userUuid, одинаковый во всех его сессиях. Admin consumers
сохраняют прежние поля и продолжают работать.

Внутренний `identity.session.check { sessionUuid }` проверяет только `client_gw`
сессию для Cart sockets и возвращает `{ data: { status, sessionId?, userId?,
ownerUuid?, expiresAt? }, meta: {} }`. Нет cookie/secret/hash в ответе. При активной
сессии renewal window не требует менять ticket посреди ее короткой lease; revoked
или expired не допускаются. Проверка не является публичным HTTP endpoint.
Logout с прежним секретом в пределах установленного grace отзывает ту же сессию
после проверки fingerprint/clientType/gateway.

Гостевой режим использует ту же Session: user_uuid nullable, owner_uuid обязателен.
identity.session.issueGuest выдаёт сессию client_gw/web без создания User и Cart.
Гостевой owner стабилен при renew; срок GUEST_SESSION_IDLE_TIMEOUT_MS по умолчанию
30 дней. verify/check пассивны. Sign-in получает текущий sessionSecret только от
проверенного gateway; после успешной проверки пароля одна транзакция создаёт
аккаунтную сессию, отзывает гостя и пишет identity.owner.closed.v1 в outbox.
Неверный пароль ничего не меняет. Область замены аккаунта включает gateway/clientType.

Worker завершает истёкших гостей и публикует lifecycle-события с повтором в
AMQP_EVENTS_EXCHANGE. Identity не зависит от Cart: каждый потребитель применяет
свою политику завершения владельца. Logout аккаунта не публикует owner.closed.
Применить GuestSessions1794870000000 перед новым runtime. При отсутствии ts-node
после yarn build:identity_srv использовать
`node node_modules/typeorm/cli.js -d dist/data-source.js migration:run`.

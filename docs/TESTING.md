# Visora API — проверка всех endpoints

Репозиторий содержит **спецификацию OpenAPI 3.1** без живого бэкенда. «Проверить API» = валидация спецификации, сборка, интерактивные доки и мок-запросы.

- Документация (Redoc): https://meir909.github.io/visor-api/
- Сырой bundled OpenAPI: https://meir909.github.io/visor-api/openapi.yaml
- Исходник: `openapi.yaml` + `paths/` + `schemas/` + `components/`

---

## 1. Валидация спецификации

```bash
cd "/Users/meiyrbek/Desktop/mvp comp/visora-api"
npx @redocly/cli@latest lint openapi.yaml
```

Ожидаемый результат:

```
Woohoo! Your API description is valid. 🎉
```

Проверяет: все 127 путей, резолв ref-ов между внешними файлами, корректность схем и примеров.

## 2. Сборка (bundle) — один файл из всех частей

```bash
npx @redocly/cli@latest bundle openapi.yaml -o /tmp/visor-bundled.yaml
```

Контроль охвата по количеству путей:

```bash
npx @redocly/cli@latest bundle openapi.yaml | grep -c '^  /'
# ожидаемо: число, равное количеству путей в корне
```

## 3. Локальный просмотр интерактивных доков

```bash
npx @redocly/cli@latest preview-docs openapi.yaml
```

Открывается `http://localhost:8080` — кликабельные эндпоинты, mermaid-диаграммы, все 14 модулей в сайдбаре.

## 4. Публикация на GitHub Pages

```bash
npx @redocly/cli@latest build-docs openapi.yaml -o docs/index.html
npx @redocly/cli@latest bundle openapi.yaml -o docs/openapi.yaml
git add -A
git commit -m "Update docs"
git push origin main
sleep 25   # Pages собирается ~25 сек
curl -s -o /dev/null -w "%{http_code}" https://meir909.github.io/visor-api/
# ожидаемо: 200
```

## 5. Мок-сервер (запросы без бэкенда)

```bash
npx @stoplight/prism-cli@latest mock openapi.yaml -p 4010
```

Базовый URL эндпоинтов: `http://localhost:4010/api/v1`

---

## Чек-лист по всем 14 модулям (curl против mock)

Переменные:

```bash
BASE=http://localhost:4010/api/v1
H='-H "Authorization: Bearer <token>"'            # Bearer-эндпоинты
ORG=4b1c9d8e-1a2b-4c3d-9e8f-7a6b5c4d3e2f
CLUB=8d7e6f5a-4b3c-4d2e-9f1a-2b3c4d5e6f7a
RES=3fa85f64-5717-4562-b3fc-2c963f66afa6
CONN=5f4e3d2c-1b2a-4c3d-9e8f-0a1b2c3d4e5f
```

### Auth
```bash
curl -s $H -H 'Content-Type: application/json' -d '{"email":"a@b.kz","password":"x"}' \
  $BASE/auth/login
curl -s $H $BASE/auth/me
curl -s $H $BASE/auth/logout
curl -s -X POST $H $BASE/auth/refresh
```

### Roles
```bash
curl -s $H "$BASE/me/roles"
curl -s $H "$BASE/organizations/$ORG/roles"
```

### Organizations
```bash
curl -s $H $BASE/organizations
curl -s $H $BASE/organizations/$ORG
curl -s $H -X POST -H 'Content-Type: application/json' -d '{"name":"X","slug":"x"}' $BASE/organizations
```

### Clubs
```bash
curl -s $H "$BASE/clubs/$CLUB"                       # публичная карточка
curl -s $H "$BASE/me/clubs"                          # мои клубы
curl -s $H "$BASE/organizations/$ORG/clubs"          # админ-список
```

### Computers
```bash
curl -s $H "$BASE/clubs/$CLUB/computers"
curl -s $H "$BASE/clubs/$CLUB/computers/$RES"
# Provider connection / binding
curl -s $H "$BASE/organizations/$ORG/clubs/$CLUB/provider-connections"
curl -s $H "$BASE/organizations/$ORG/clubs/$CLUB/provider-connections/$CONN/test"
```

### Layout
```bash
curl -s $H "$BASE/clubs/$CLUB/layout"
curl -s $H "$BASE/organizations/$ORG/clubs/$CLUB/zones"
```

### Analytics
```bash
curl -s $H "$BASE/organizations/$ORG/analytics/overview"
curl -s $H "$BASE/organizations/$ORG/clubs/$CLUB/analytics/occupancy?from=2026-08-01T00:00:00Z&to=2026-08-08T00:00:00Z"
```

### Admin
```bash
curl -s $H "$BASE/organizations/$ORG/clubs/$CLUB/discounts"
curl -s $H -X POST -H 'Content-Type: application/json' -d '{"code":"VIPKZ10","name":"VIP"}' \
  "$BASE/organizations/$ORG/clubs/$CLUB/discounts"
curl -s $H "$BASE/organizations/$ORG/clubs/$CLUB/redemptions"
```

### Reservations
```bash
curl -s $H "$BASE/clubs/$CLUB/reservations/availability?startAt=2026-08-06T17:00:00Z&endAt=2026-08-06T21:00:00Z"
curl -s $H -H 'Idempotency-Key: k-res-1' -H 'Content-Type: application/json' \
  -d "{\"clubId\":\"$CLUB\",\"resourceIds\":[\"$RES\"],\"startAt\":\"2026-08-06T17:00:00Z\",\"endAt\":\"2026-08-06T21:00:00Z\"}" \
  "$BASE/clubs/$CLUB/reservations"
curl -s $H "$BASE/me/reservations"
```

### Owner
```bash
curl -s $H "$BASE/organizations/$ORG/owner/dashboard"
curl -s $H "$BASE/organizations/$ORG/owner/network"
curl -s $H "$BASE/organizations/$ORG/owner/staff"
```

### PlatformAdmin
```bash
curl -s $H "$BASE/platform-admin/users?q="
curl -s $H "$BASE/platform-admin/organizations"
curl -s $H "$BASE/platform-admin/stats/overview"
curl -s $H "$BASE/platform-admin/audit-log"
# назначение SUPER_ADMIN (break-glass)
curl -s $H -X POST -H 'Content-Type: application/json' \
  -d '{"roleCode":"SUPER_ADMIN","assignedReason":"incident","validUntil":"2026-08-08T00:00:00Z"}' \
  "$BASE/platform-admin/users/$RES/platform-roles"
```

### Provider (webhook — БЕЗ Bearer, с подписью)
```bash
curl -s -H 'X-Provider-Signature: t=1722776400,v1=abc' -H 'Content-Type: application/json' \
  -d '{"externalEventId":"evt-1","eventType":"RESOURCE_STATE_CHANGED","eventOccurredAt":"2026-08-05T11:30:00Z","resource":{"externalResourceId":"gizmo-station-014"}}' \
  "$BASE/webhooks/provider/$CONN/events"
curl -s $H "$BASE/organizations/$ORG/clubs/$CLUB/provider-connections/$CONN/events"
curl -s $H "$BASE/organizations/$ORG/clubs/$CLUB/provider-connections/$CONN/resources/state"
```

### Pricing
```bash
curl -s $H -H 'Content-Type: application/json' \
  -d "{\"clubId\":\"$CLUB\",\"resourceIds\":[\"$RES\"],\"startAt\":\"2026-08-06T17:00:00Z\",\"endAt\":\"2026-08-06T21:00:00Z\"}" \
  "$BASE/organizations/$ORG/clubs/$CLUB/pricing/quote"
curl -s $H "$BASE/organizations/$ORG/clubs/$CLUB/pricing-rules"
```

### Notifications
```bash
curl -s $H $BASE/me/notifications
curl -s $H $BASE/me/notifications/unread-count
curl -s $H "$BASE/me/notifications/c2d3e4f5-6a7b-4c8d-9e0f-1a2b3c4d5e6f"
curl -s $H -X PATCH \
  "$BASE/me/notifications/c2d3e4f5-6a7b-4c8d-9e0f-1a2b3c4d5e6f/read"
curl -s $H -X POST $BASE/me/notifications/read-all
curl -s $H $BASE/me/notifications/preferences
curl -s $H -X PUT -H 'Content-Type: application/json' \
  -d '{"locale":"ru-RU","rules":[{"type":"NO_SHOW_RECORDED","enabled":true,"channels":["IN_APP","SMS"]}]}' \
  $BASE/me/notifications/preferences
```

---

## Быстрый смоук-тест (одной командой)

```bash
for ep in \
  "$BASE/clubs/$CLUB/computers" \
  "$BASE/me/notifications" \
  "$BASE/organizations/$ORG/owner/dashboard" \
  "$BASE/platform-admin/stats/overview" \
  "$BASE/organizations/$ORG/clubs/$CLUB/pricing-rules" ; do
  echo "== $ep"; curl -s $H "$ep" | head -c 200; echo
done
```
# CLAUDE.md — easy-GTFS-RT

## Czym jest ten projekt
Wyłącznie CI/CD (GitHub Actions) automatyzujące codzienne nagrywanie GTFS-RT VehiclePositions
i odbudowę zrealizowanego GTFS dla easy-OTP's narzędzia Family A
(`tools/family_a_reconstruction/` w `GISBoost/easy-OTP`). Ten folder NIE zawiera kodu Family A
— każdy workflow robi dodatkowy `actions/checkout` na `GISBoost/easy-OTP` i uruchamia stamtąd
`family_a` CLI. Nie duplikuj tego kodu tutaj.

## Źródło prawdy
**Aktualnie: `GISBoost/easy-OTP`'s `docs/prd/PR_easy-OTP_termux-migration.md`** (tor telefonowy
TX-1..TX-8, jedyny aktualnie produkujący pipeline od 2026-07-14) i
`scripts/termux/README.md` w tym samym repo (strona telefonowa tego pipeline'u).

Historycznie: `docs/prd/PR_easy-OTP_v07.md` (sekcje FA-7/FA-8/FA-9) i
`docs/prompts/easy-OTP_v07_prompts_for-claude-code.md` — opisują **wygaszony** tor nagrywania
przez GitHub Actions (chunki), zastąpiony przez telefon. `family_a_build_and_notify.yml` i
`family_a_build_and_notify_on_demand.yml` zostały w repo jako odniesienie, ale nie mają już
źródła danych (chunki usunięte) — nie traktuj ich jako działający fallback bez wyraźnej decyzji
o przywróceniu nagrywania przez Actions.

## Twarde ograniczenia
- Zero zmian w `GISBoost/easy-OTP` — checkout tego repo w workflowach jest zawsze read-only.
- Nie zgaduj URL-i/wartości konfiguracyjnych — pytaj, jeśli coś niepotwierdzone.
- `family_a_build_and_notify_from_phone.yml`'s `repository_dispatch` `event_type`
  (`"phone-sweep-complete"`) musi się zgadzać dokładnie (case-sensitive) z tym, co wysyła
  `sweep_and_upload.sh` w `easy-OTP` — literówka po którejkolwiek stronie = cicho zignorowany
  event, bez błędu.
- Sekrety (`CALLMEBOT_PHONE`, `CALLMEBOT_APIKEY`) żyją w Settings tego repo, nie w kodzie.
- **`config/cities.json` (TX-8)** to źródło prawdy które miasta buduje
  `family_a_build_and_notify_from_phone.yml` — zastąpił dawne zmienne
  `LODZ_STATIC_GTFS_URL`/`LODZ_VEHICLE_POSITIONS_URL` w Settings (URL-e publiczne, nie sekrety,
  więc wersjonowanie w repo jest prostsze). Klucz (city id) musi się zgadzać dokładnie
  (case-sensitive) z `city` w `client_payload` dispatcha z telefonu i z nazwą
  pliku/usługi na telefonie (`cities/<city>.env`, `family-a-record-<city>`, w `easy-OTP`).
  Nieznany `city` = jawny błąd joba (`resolve_targets`), nigdy cichy no-op.
- Kod/komentarze/commity: po angielsku.

## Workflow pracy
Jeden kamień milowy (FA-7/FA-8/FA-9) = jeden branch, jeden prompt, STOP na ręczną weryfikację
na GitHub.com przed reviewerem i commitem (Claude Code nie może triggerować Actions z tego
checkoutu).

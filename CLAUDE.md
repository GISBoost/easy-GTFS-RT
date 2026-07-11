# CLAUDE.md — easy-GTFS-RT

## Czym jest ten projekt
Wyłącznie CI/CD (GitHub Actions) automatyzujące codzienne nagrywanie GTFS-RT VehiclePositions
i odbudowę zrealizowanego GTFS dla easy-OTP's narzędzia Family A
(`tools/family_a_reconstruction/` w `GISBoost/easy-OTP`). Ten folder NIE zawiera kodu Family A
— każdy workflow robi dodatkowy `actions/checkout` na `GISBoost/easy-OTP` i uruchamia stamtąd
`family_a` CLI. Nie duplikuj tego kodu tutaj.

## Źródło prawdy
`GISBoost/easy-OTP`'s `docs/prd/PR_easy-OTP_v07.md` (sekcje FA-7/FA-8/FA-9) i
`docs/prompts/easy-OTP_v07_prompts_for-claude-code.md` (prompty FA-7/FA-8/FA-9). Ten CLAUDE.md
tego nie powiela.

## Twarde ograniczenia
- Zero zmian w `GISBoost/easy-OTP` — checkout tego repo w workflowach jest zawsze read-only.
- Nie zgaduj URL-i/wartości konfiguracyjnych — pytaj, jeśli coś niepotwierdzone.
- Nie zmieniaj granic/czasów trwania chunków FA-7 bez wyraźnej decyzji — są dobrane pod szczyty
  komunikacyjne (7-10/14-18), nie równy podział doby.
- FA-8's odkrywanie artefaktów jest po prefiksie nazwy (`positions-<data>-`), nigdy po
  zahardkodowanej liście/liczbie — to musi zostać tak, żeby FA-9's doraźne nagrania też się
  wliczały.
- Sekrety (`CALLMEBOT_PHONE`, `CALLMEBOT_APIKEY`) i zmienne (`LODZ_STATIC_GTFS_URL`,
  `LODZ_VEHICLE_POSITIONS_URL`) żyją w Settings tego repo, nie w kodzie.
- Kod/komentarze/commity: po angielsku.

## Workflow pracy
Jeden kamień milowy (FA-7/FA-8/FA-9) = jeden branch, jeden prompt, STOP na ręczną weryfikację
na GitHub.com przed reviewerem i commitem (Claude Code nie może triggerować Actions z tego
checkoutu).
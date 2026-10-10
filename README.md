# haconfig — 신원아파트 Home Assistant 설정 백업

신원아파트 Home Assistant 인스턴스의 설정(`configuration.yaml`, `automations.yaml`, `scripts.yaml`, `scenes.yaml`, 테마, 이미지)과 `dashboard-2nd`("신원아파트") Lovelace 대시보드를 백업·버전관리하기 위한 저장소입니다. 실제 설정은 Home Assistant 본체(`192.168.219.113:8022`)에서 관리되고, 이 저장소는 그 스냅샷을 주기적으로 받아와 git으로 이력을 추적하는 용도입니다.

## 폴더 구조

```
configuration.yaml     # HA 메인 설정 (SSH 모드로만 받을 수 있음)
automations.yaml       # 자동화 전체 (API로 재구성)
scripts.yaml           # 스크립트 전체 (API로 재구성)
scenes.yaml            # 씬 전체 (API로 재구성)
dashboards/            # dashboard-2nd 스냅샷들 (main.yaml이 최신)
themes/                # 테마 정의
www-images/            # 대시보드에서 쓰는 이미지 (floor1.png, floor2.png 등)
getDashboard.sh        # 대시보드만 받아서 dashboards/main.yaml 생성
getCode.sh             # configuration/automations/scripts/scenes/themes/images 다운로드 (SSH, 대시보드 제외)
.agents/skills/        # Claude용 Home Assistant 베스트 프랙티스 스킬 참고 자료
```

## 동기화 스크립트

용도가 다른 두 스크립트를 **각각 따로** 실행합니다.

```bash
./getCode.sh        # configuration/automations/scripts/scenes/themes/www-images (SSH, 비밀번호 필요)
./getDashboard.sh   # dashboard-2nd → dashboards/main.yaml (API, 비밀번호 불필요)
```

- `getCode.sh`는 SSH(scp)로 HA 본체의 원본 파일을 그대로 받아옵니다. 그래서 `automations.yaml`/`scripts.yaml`이 HA가 직접 쓰는 YAML 형식 그대로 유지되어, `git diff`에 실제 변경만 나타납니다. **대시보드(`dashboards/`)는 받지 않습니다.** HA 본체의 `dashboards/` 폴더는 오래된 파일이라 덮어쓰면 최신 `main.yaml`이 옛 내용으로 되돌아갑니다.
- `getDashboard.sh`는 WebSocket API로 현재 쓰는 `dashboard-2nd`(storage 모드)를 받아 `dashboards/main.yaml`로 변환합니다. 대시보드는 항상 이 스크립트로만 받습니다.

> **주의:** `getDashboard.sh`에는 `HA_TOKEN` 환경변수가 없을 때 쓰는 기본값으로 실제 long-lived access token이 하드코딩되어 있습니다. 다행히 `.gitignore`에 `*.sh`가 포함되어 있어 스크립트 자체는 git에 커밋되지 않지만, 토큰을 바꾸거나 재발급할 때는 스크립트 안의 기본값도 같이 수정해야 합니다. 가능하면 기본값에 토큰을 박아두지 말고 `HA_TOKEN=... ./getDashboard.sh`처럼 환경변수로 주입하는 걸 권장합니다.

## 대시보드(dashboard-2nd) 구조

Lovelace storage 모드로 관리되며, 메인 뷰 하나(`sections`)에 방별 `custom:bubble-card` 팝업과 공용 팝업들이 나열되어 있습니다.

- **방별 팝업** (거실, 주방, 거실화장실, 안방화장실, 현관, 안방, 준호방, 나경방, 옷방, 공부방, 지하방, cafe, 복층화장실): 각각 `room_switch_panel`(조명/팬/에어컨 버튼 패널)과 `room_temp_card`(온습도 + 난방/에어컨/팬 상태 아이콘) decluttering 템플릿을 사용합니다.
- **공용 팝업**: `#aircon`(에어컨), `#heater`(난방), `#fan`(팬) — 전체 방의 `mushroom-climate-card`/`mushroom-fan-card`를 한 화면에 모아 보여줍니다. `#flash`(조명) — 방별 조명 스위치를 한 화면에서 켜고 끌 수 있는 팝업입니다.
- **decluttering_templates**: `light_slider`, `room_switch_panel`, `room_temp_card`, `room_overview_card` 네 가지가 대시보드 상단에 정의되어 있고, 각 카드는 여기에 `variables`만 바꿔 넣어 재사용합니다.

## 엔티티 네이밍 규칙

가상 스위치(`input_boolean`)와 조명(`light`)은 방 접두어 + 역할로 통일되어 있습니다. 코드를 볼 때 엔티티 이름만 보고 역할을 바로 알 수 있도록 맞춘 컨벤션입니다.

| 역할 | 패턴 | 예시 |
|---|---|---|
| 조명 (개별 조명/보조등 등) | `<room>_jomyeong_N` | `input_boolean.jubang_jomyeong_1` |
| 실링팬 | `<room>_silringpaen_switch` | `input_boolean.anbang_silringpaen_switch` |
| 에어컨 | `<room>_eeokeon_switch` | `input_boolean.nagyeongbang_eeokeon_switch` |
| 환풍기(화장실) | `<room>_hwanpunggi_switch` | `input_boolean.geosilhwajangsil_hwanpunggi_switch` |

`jomyeong_N`으로 끝나는 엔티티는 전부 순수 조명이며, 실링팬/에어컨/환풍기는 이름 자체에 역할이 들어가 있어 구분됩니다. 대시보드의 "조명" 통합 팝업(`#flash`)도 이 규칙에 기대어 방별 조명만 걸러서 보여줍니다.

## 알아두어야 할 함정

**1. YAML `platform: template` 엔티티는 entity_id를 API로 못 바꿉니다.**
일부 `fan:`/`climate:` 엔티티는 `configuration.yaml`에 `platform: template`으로 정의되어 있고, Jinja(`value_template`, `turn_on`/`turn_off`)에 참조할 entity_id가 문자열로 박혀 있습니다. 엔티티 레지스트리에서 이름을 바꿔도(`ha_set_entity`의 `new_entity_id`) 이 템플릿 내부 참조는 안 바뀌어서, 겉보기엔 성공한 것 같지만 상태가 `unknown`이 되고 on/off가 먹통이 됩니다. 진짜로 이름을 통일하려면 `configuration.yaml`에서 **입력 엔티티 정의와 템플릿의 Jinja 참조 두 곳을 모두** 고치고 HA를 재시작해야 합니다. (`geosilhwajangsil_hwanpunggi_switch`, `bogceunghwajangsil_hwanpunggi_switch`가 이 방식으로 통일된 사례입니다.)

**2. automations.yaml/scripts.yaml은 편집할 때마다 전체가 재포맷됩니다.**
HA UI나 API(`ha_config_set_script` 등)로 스크립트/자동화를 한 번이라도 저장하면, HA가 파일 전체를 자기 YAML 직렬화기로 다시 씁니다. 리스트 들여쓰기 스타일(부모 키와 같은 레벨 vs 한 칸 더 들여쓰기)이 바뀌면서 실제 내용 변경이 없어도 git diff가 크게 잡힐 수 있습니다. `git diff -w`(공백 무시)로 보거나, 로컬 파일을 HA가 쓰는 스타일로 맞춰두면 이후 diff가 깨끗하게 유지됩니다.

**3. 대시보드 수정은 반드시 Home Assistant API로.**
`dashboards/*.yaml`은 참고/백업용 스냅샷이며, 직접 수정해서 다시 올리는 방식이 아니라 Home Assistant 쪽에서 수정 → `getDashboard.sh`로 재추출하는 흐름입니다. 가상 스위치나 automation도 마찬가지로 HA API를 통해서만 변경하고, 이 저장소의 YAML은 기록용으로 받아두는 것입니다.

## 히스토리

`dashboards/` 안의 날짜가 붙은 파일들(`0814.yaml`, `0826.yaml`, ...)은 각 시점의 대시보드 스냅샷이고, `main.yaml`이 가장 최근 상태입니다. 대시보드를 크게 바꾸기 전에는 `getDashboard.sh`로 현재 상태를 한 번 더 저장해두는 걸 권장합니다.

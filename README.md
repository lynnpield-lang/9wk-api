# 마인크래프트 3D - minecraft-data API 적용

## 1. 개요
- 5주차에 만든 3D 마인크래프트 게임에 **PrismarineJS minecraft-data** API를 연결했다.
- 코드에 직접 적어 두었던 **조합법, 음식 회복량, 채굴 시간, 몹 드롭**을 실제 마인크래프트 데이터로 채운다.
- 새 아이템: 삽 4종(나무/돌/철/다이아), 건축 블록 4종(석재 벽돌, 석탄 블록, 철 블록, 다이아 블록).
  - 도끼와 사암은 이미 게임에 있어서 그대로 쓴다.

## 2. 실행 방법
```bash
uv sync
uv run minecraft_3d.py
```
- 파이썬 버전은 `.python-version`(3.13)으로 고정했다.
- API 요청은 파이썬 기본 라이브러리(`urllib`)로 해서 새 패키지는 없다.

## 3. API Key
- **필요 없음.** minecraft-data는 인증 없는 공개 JSON이라 `.env` 설정이 필요 없다.

## 4. 사용한 API
기본 주소: `https://raw.githubusercontent.com/PrismarineJS/minecraft-data/master/data/pc/1.20/`
(데이터 버전 1.20 고정, `API_DATA_VERSION`)

| 파일 | 가져오는 값 | 게임에서 쓰는 곳 |
|---|---|---|
| `items.json` | 아이템 이름 ↔ 번호 | 레시피 안의 아이템 번호를 이름으로 바꿀 때 |
| `recipes.json` | 조합법 (배치형 `inShape` / 무배치형 `ingredients`) | 인벤토리(E) 조합 목록 `RECIPES` |
| `foods.json` | 허기 회복량 `foodPoints` | 음식 먹기 `FOOD_VALUES`, 아이템 이름의 "허기 +n" |
| `blocks.json` | 단단함 `hardness`, 필요한 도구 `harvestTools` | 채굴 시간 `BLOCK_MINING_TIME` |
| `entityLoot.json` | 몹 드롭 아이템·확률·개수 | 몹 드롭 `MOB_DROPS` (돼지, 좀비, 크리퍼) |

## 5. 코드 구조
| 파일 | 역할 |
|---|---|
| `minecraft_3d.py` | 게임 본체: 메뉴, 입력, 조합, 렌더링 루프. 시작할 때 `start_game_data_load()`를 부르고, 매 프레임 `poll_game_data()`로 결과를 확인해 적용한다 |
| `mc3d_world.py` | 블록·아이템 표, 월드 생성, **API 코드 전부** |
| `mc3d_entities.py` | 플레이어, 몹 |
| `mc3d_render.py` | 렌더링 보조 |

API 관련 함수 (모두 `mc3d_world.py`):

| 함수 | 하는 일 |
|---|---|
| `fetch_api_data()` | `API_FILES` 5개를 받는다 (파일당 10초 제한). 하나라도 실패하면 예외 |
| `convert_api_data(raw)` | 원본을 게임 형식으로 바꾼다. 아래 네 함수를 부른다 |
| `_convert_recipes()` / `_convert_recipe()` | 레시피 → 재료 개수, 작업대 필요 여부 |
| `_convert_mining()` | 단단함 → 채굴 시간 |
| `_convert_drops()` | 몹 드롭 → 게임 드롭 표 |
| `parse_game_data(data)` | 값 검사 (개수·확률 범위, 빠진 항목). JSON 문자열 키를 숫자로 되돌린다 |
| `load_data_cache()` / `save_data_cache()` | 캐시 `mc3d_data_cache.json` 읽기·쓰기 (임시 파일에 쓰고 바꿔치기) |
| `load_game_data()` | 캐시 → 온라인 → 기본값 순서로 시도. `(출처, 데이터)`를 돌려준다 |
| `start_game_data_load()` | `load_game_data()`를 별도 스레드에서 실행한다 |
| `apply_game_data(data)` | 게임 표에 반영한다 (`RECIPES.extend`, `FOOD_VALUES.update` 등) |

## 6. 동작 흐름
```
게임 켜기 → start_game_data_load() (별도 스레드)
   ├─ 캐시 있음 ─────────────→ "게임 데이터: 캐시"
   ├─ 캐시 없음 → API 요청 → 변환·검사 → 캐시 저장 → "게임 데이터: 온라인"
   └─ 인터넷 없음 / 실패 ─────→ 원래 코드 값 그대로 → "게임 데이터: 기본값"
매 프레임 poll_game_data(): 결과가 오면 apply_game_data() 후 메뉴에 출처 표시
```

## 7. 데이터 변환에서 신경 쓴 점
- **이름 매핑**: 게임에 석탄·조약돌 아이템이 따로 없어서 비슷한 아이템으로 바꾼다 (`API_INGREDIENTS`).
  - 예: `coal`→석탄 광석, `cobblestone`→돌, `oak_planks`→목재
  - 게임에 없는 재료가 들어간 레시피는 버리고, 나무 종류별로 중복된 레시피는 첫 번째 것만 쓴다.
- **조합 방식**: 3x3 배치를 재료 개수로 바꾼다.
  - 2x2 안에 들어가면 맨손으로 조합하고, 그보다 크면 작업대가 필요하다.
  - 기존 조합법은 덮어쓰지 않고 새 아이템 8개의 레시피만 추가한다.
- **채굴 시간**: 실제 마인크래프트 공식을 쓴다.
  - 맨손 기준 단단함 × 1.5, 도구가 필요한 블록은 단단함 × 5
  - 단단함 0인 블록은 0으로 나누지 않도록 최소 0.05초 (`MIN_MINING_TIME`)
  - 랜턴은 곡괭이 효과가 없어 맨손 17.5초가 되므로 코드 값을 유지한다.
- **몹**: 체력·공격력은 API에 없어서 기존 값을 유지한다.
  - 게임에 없는 드롭(당근, 감자)은 무시한다.
  - `glowbeast`처럼 이 게임에만 있는 몹은 원래 값을 쓴다.
- **별도 스레드**: 처음 받는 데이터가 약 1.5MB라 메인 루프에서 받으면 화면이 몇 초 멈춘다. 그래서 스레드에서 받고, 결과만 메인 루프에서 반영한다.
- **같은 객체 유지**: 다른 파일이 `from mc3d_world import *`로 같은 표를 쓰므로 새로 대입하지 않고 `extend`/`update`로 바꾼다.
- **캐시 안전장치**: 캐시가 망가졌거나 버전(`API_CACHE_VERSION`, 데이터 버전)이 다르면 버리고 다시 받는다.
- **저장 호환**: 새 아이템은 비어 있는 번호에 넣고 저장 버전은 그대로 둬서, 예전 저장 파일도 불러온다.

## 8. 한계 / 구현 안 한 것
- 화로, 방어구, 인챈트 시스템은 없다.
  - 유리·구운 고기·벽돌은 원래 화로 레시피라 기존 조합법을 유지했다.
- 마인크래프트 텍스처·모델은 쓰지 않는다.
- API에 있는 모든 레시피를 게임 아이템으로 만들지는 않았다 (새 아이템 8개만).
- 몹 체력·공격력·크기, 게임에만 있는 값(발광 블록, 빛나는 열매 등)은 API에 없어서 코드 값을 쓴다.

## 9. 저장소 문서
| 파일 | 내용 |
|---|---|
| `api-plan.md` | 이번 API 적용 기획안 |
| `ai_chat_api.md` | 이번 API 작업의 AI 대화 원문 |
| `plan.md`, `chat.md` | 5주차 게임 기획안과 AI 대화 원문 |

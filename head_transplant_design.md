# DWPose Editor — 참조 기반 이식 시스템 설계문서

## 1. 현재 문제 분석

### 1.1 Fake 3D가 작동하지 않는 이유

현재 `applyFake3DPartTurn()` (line 1742):

```js
// 현재 방식: X좌표 균일 스케일링
kpts[j*3] = pivot.x + dx * (1 - turnFactor * 0.3);  // 오른쪽 관절 압축
kpts[j*3] = pivot.x + dx * (1 + turnFactor * 0.3);  // 왼쪽 관절 확장
```

**문제**: 이건 "납작하게 만들기"이지 "회전"이 아님.
- 실험으로 증명: x-offset 균일 이동은 ControlNet이 머리 돌림으로 인식 안 함
- 실제 회전에서는 코-목 벡터 방향, 양쪽 눈 비대칭, 귀 가시성이 동시에 변함
- 2D 좌표 산술로는 이 복합 변화를 재현 불가능

### 1.2 실험 결과가 증명한 해법

8개 참조 이미지에서 DWPose로 추출한 JSON → ControlNet에 넣으면 전 자세에서 정확히 추종됨 (오차 2~3%).

**결론**: 수학적 합성 대신, **실제 이미지에서 추출한 키포인트를 이식**하는 것이 유일하게 작동하는 방법.

---

## 2. 설계 개요

### 2.1 핵심 아이디어

Fake 3D 섹션을 **"참조 기반 이식 시스템"**으로 교체:
- 내장 프리셋 데이터: 실제 이미지에서 추출한 머리 키포인트 JSON
- Neck(1) 앵커 기반 정규화로 현재 body에 이식
- 기존 FK/손프리셋과 동일한 base/restore 패턴 사용

### 2.2 기존 패턴 재활용

에디터에 이미 검증된 패턴이 있음:

| 기능 | base 저장 | 복원 | 적용 |
|------|-----------|------|------|
| FK 관절 | `fkBasePose` | `restoreFKBase()` | `applyAllFK()` |
| 손 프리셋 | `handPresetBase` | `restoreHandBase()` | `applyHandPreset()` |
| 표정 | `baseFaceData` | 직접 복원 | `applyGaze()` / `applyExpression()` |
| Fake 3D | `fake3dBasePose` | 직접 복원 | `applyAllFake3D()` |
| **이식(신규)** | `transplantBasePose` | `restoreTransplantBase()` | `applyHeadTransplant()` |

---

## 3. 데이터 구조

### 3.1 프리셋 데이터 형식

```js
const HEAD_PRESETS = {
    // key: 프리셋 이름
    // value: 참조 이미지에서 추출한 정규화된 키포인트
    "정면": {
        label: "정면",
        // 정규화 좌표: Neck 기준 상대좌표를 Nose-Neck 거리로 나눈 값
        // 즉, Neck = (0, 0), scale = 1.0 (Nose-Neck 거리 기준)
        body_head: {
            // index: [norm_x, norm_y, confidence]
            0:  [dx, dy, conf],   // Nose
            14: [dx, dy, conf],   // REye  (원본 인덱스 체계에서 14=REye)
            15: [dx, dy, conf],   // LEye
            16: [dx, dy, conf],   // REar
            17: [dx, dy, conf],   // LEar
        },
        face_68: [
            // 68개 × [norm_x, norm_y, confidence]
            // 마찬가지로 Neck 기준 상대좌표, Nose-Neck 거리로 정규화
            [dx, dy, conf], // face[0]
            [dx, dy, conf], // face[1]
            ...
        ]
    },
    "살짝돌림": { ... },
    "3/4뷰": { ... },
    ...
};
```

### 3.2 정규화 공식

**추출 시** (프리셋 데이터 생성):
```
ref_neck = (ref_kp[1*3], ref_kp[1*3+1])
ref_nose = (ref_kp[0*3], ref_kp[0*3+1])
ref_scale = distance(ref_nose, ref_neck)

norm_x = (kp_x - ref_neck_x) / ref_scale
norm_y = (kp_y - ref_neck_y) / ref_scale
```

**적용 시** (현재 pose에 이식):
```
orig_neck = (pose_kp[1*3], pose_kp[1*3+1])
orig_nose = (pose_kp[0*3], pose_kp[0*3+1])
orig_scale = distance(orig_nose, orig_neck)

new_kp_x = orig_neck_x + norm_x * orig_scale
new_kp_y = orig_neck_y + norm_y * orig_scale
```

### 3.3 이식 대상 키포인트

```
교체 (참조에서 가져옴):
├─ Body: Nose(0), REye(14), LEye(15), REar(16), LEar(17)
└─ Face: 68개 전체 (face_keypoints_2d[0..67])

유지 (원본 body 그대로):
├─ Body: Neck(1), 어깨(2,5), 팔(3,4,6,7), 엉덩이(8,11), 다리(9,10,12,13), 발(18~23)
└─ Hands: 양손 전체
```

---

## 4. 코드 변경 사항

### 4.1 dwpose_editor_logic.js 변경

#### 4.1.1 삭제할 코드

Fake 3D 관련 전체 삭제 (line 1676~1904):

```
삭제 대상:
- let fake3dBasePose = null;                    (line 1677)
- const FAKE3D_PARTS = { ... };                 (line 1679~1729)
- function ensureFake3DBase() { ... }           (line 1731~1739)
- function applyFake3DPartTurn() { ... }        (line 1742~1774)
- function applyFake3DPartLean() { ... }        (line 1776~1801)
- function applyFake3DScaleHand() { ... }       (line 1803~1812)
- function applyFake3DTurnFace() { ... }        (line 1814~1823)
- function applyFake3DLeanFace() { ... }        (line 1825~1833)
- function updateFake3DSliderDisplay() { ... }  (line 1835~1842)
- function applyAllFake3D() { ... }             (line 1844~1876)
- function onFake3DChange() { ... }             (line 1878~1881)
- function resetFake3D() { ... }                (line 1883~1904)
```

#### 4.1.2 추가할 코드: 프리셋 데이터

Fake 3D 삭제한 자리(line 1676)에 삽입:

```js
// ========== 머리 이식 프리셋 ==========
// 실제 참조 이미지에서 DWPose로 추출 → Neck 기준 정규화
// norm = (kp - neck) / nose_neck_distance
//
// ※ 이 데이터는 별도 스크립트로 생성 (섹션 5 참조)

const HEAD_PRESETS = {}; // → 섹션 5의 생성 스크립트 출력을 여기에 붙여넣기
```

#### 4.1.3 추가할 코드: 이식 로직

```js
// ========== 머리 이식 시스템 ==========
let transplantBasePose = null;
let currentHeadPreset = null;

// --- base 관리 (FK/손프리셋과 동일 패턴) ---

function ensureTransplantBase() {
    if (!transplantBasePose) {
        transplantBasePose = {
            body: [...poseData.pose_keypoints_2d],
            face: [...poseData.face_keypoints_2d],
            // 손/hand는 이식 대상 아니므로 저장 불필요
        };
    }
}

function restoreTransplantBase() {
    if (transplantBasePose) {
        poseData.pose_keypoints_2d = [...transplantBasePose.body];
        poseData.face_keypoints_2d = [...transplantBasePose.face];
    }
}

// --- 핵심 이식 함수 ---

function applyHeadTransplant(presetKey) {
    const preset = HEAD_PRESETS[presetKey];
    if (!preset) return;

    saveHistory();
    ensureTransplantBase();
    restoreTransplantBase();  // 항상 원본에서 시작 (슬라이더 패턴)

    const kpts = poseData.pose_keypoints_2d;
    if (kpts.length < 18 * 3) return;

    // 1. 현재 pose의 Neck 위치와 Nose-Neck 거리(스케일) 계산
    const neckX = kpts[1 * 3];
    const neckY = kpts[1 * 3 + 1];
    const noseX = kpts[0 * 3];
    const noseY = kpts[0 * 3 + 1];

    if (kpts[1 * 3 + 2] <= 0 || kpts[0 * 3 + 2] <= 0) {
        showStatus('⚠ Neck 또는 Nose가 없어 이식 불가');
        return;
    }

    const origScale = Math.sqrt((noseX - neckX) ** 2 + (noseY - neckY) ** 2);
    if (origScale < 1) {
        showStatus('⚠ Nose-Neck 거리가 너무 작음');
        return;
    }

    // 2. Body 머리 키포인트 이식 (Nose, REye, LEye, REar, LEar)
    const headBodyIndices = [0, 14, 15, 16, 17];

    headBodyIndices.forEach(idx => {
        const presetPt = preset.body_head[idx];
        if (!presetPt) return;

        const [normX, normY, conf] = presetPt;
        kpts[idx * 3]     = neckX + normX * origScale;
        kpts[idx * 3 + 1] = neckY + normY * origScale;
        kpts[idx * 3 + 2] = conf;
    });

    // 3. Face 68 키포인트 이식
    if (preset.face_68 && preset.face_68.length === 68) {
        const faceKpts = poseData.face_keypoints_2d;

        // face 배열이 충분한 크기인지 확인
        while (faceKpts.length < 68 * 3) faceKpts.push(0, 0, 0);

        preset.face_68.forEach((pt, i) => {
            const [normX, normY, conf] = pt;
            faceKpts[i * 3]     = neckX + normX * origScale;
            faceKpts[i * 3 + 1] = neckY + normY * origScale;
            faceKpts[i * 3 + 2] = conf;
        });
    }

    currentHeadPreset = presetKey;
    updateHeadPresetUI();
    render();
    updateUI();
    showStatus(`머리 이식: ${preset.label}`);
}

// --- UI 업데이트 ---

function updateHeadPresetUI() {
    // 프리셋 버튼 활성 상태 표시
    document.querySelectorAll('.head-preset-btn').forEach(btn => {
        btn.classList.toggle('on', btn.dataset.preset === currentHeadPreset);
    });
}

// --- 초기화 ---

function resetHeadTransplant() {
    if (transplantBasePose) {
        poseData.pose_keypoints_2d = [...transplantBasePose.body];
        poseData.face_keypoints_2d = [...transplantBasePose.face];
        transplantBasePose = null;
    }

    currentHeadPreset = null;
    updateHeadPresetUI();
    render();
    updateUI();
    showStatus('머리 이식 초기화');
}
```

#### 4.1.4 resetAllBases() 수정 (line 346~351)

```js
// 수정 전:
function resetAllBases() {
    baseFaceData = null;
    fkBasePose = null;
    fake3dBasePose = null;           // ← 삭제
    handPresetBase = { left: null, right: null };
}

// 수정 후:
function resetAllBases() {
    baseFaceData = null;
    fkBasePose = null;
    transplantBasePose = null;       // ← 변경
    currentHeadPreset = null;        // ← 추가
    handPresetBase = { left: null, right: null };
}
```

#### 4.1.5 resetAllSliders() 수정 (line 353~388)

Fake 3D 슬라이더 초기화 코드 (line 378~387) 삭제:

```js
// 삭제:
const fake3dParts = ['Head', 'Shoulder', 'Waist', 'Pelvis', 'Leg'];
fake3dParts.forEach(part => {
    ['Turn', 'Lean'].forEach(axis => {
        const id = `fake3d${part}${axis}`;
        const el = document.getElementById(id);
        if (el) el.value = 0;
        const valEl = document.getElementById(id + 'Value');
        if (valEl) valEl.textContent = '0';
    });
});

// 추가 (같은 위치):
currentHeadPreset = null;
updateHeadPresetUI();
```

---

### 4.2 dwpose_editor.html 변경

#### 4.2.1 삭제할 HTML

`sec-fake3d` 섹션 전체 삭제 (line 908~973):

```html
<!-- 삭제: 6순위: Fake 3D 부위별 (접힘) -->
<div class="section-accordion" id="sec-fake3d">
    ... (머리/어깨/허리/골반/다리 turn/lean 슬라이더 전체)
</div>
```

#### 4.2.2 추가할 HTML

같은 위치에 삽입:

```html
<!-- 6순위: 머리 방향 (이식) -->
<div class="section-accordion" id="sec-head-transplant">
    <div class="section-accordion-header" onclick="this.parentElement.classList.toggle('open')">
        <span class="section-accordion-arrow">&#9654;</span>
        <span>머리 방향 (이식)</span>
    </div>
    <div class="section-accordion-content">
        <!-- 좌우 돌림 -->
        <div class="subsection-title">좌우 돌림</div>
        <div class="btn-group">
            <button class="btn head-preset-btn" data-preset="정면"
                    onclick="applyHeadTransplant('정면')">정면</button>
            <button class="btn head-preset-btn" data-preset="살짝돌림"
                    onclick="applyHeadTransplant('살짝돌림')">살짝</button>
            <button class="btn head-preset-btn" data-preset="3/4뷰"
                    onclick="applyHeadTransplant('3/4뷰')">3/4</button>
            <button class="btn head-preset-btn" data-preset="프로필"
                    onclick="applyHeadTransplant('프로필')">프로필</button>
        </div>

        <!-- 상하 숙임/올림 -->
        <div class="subsection-title">상하</div>
        <div class="btn-group">
            <button class="btn head-preset-btn" data-preset="살짝숙임"
                    onclick="applyHeadTransplant('살짝숙임')">살짝숙임</button>
            <button class="btn head-preset-btn" data-preset="많이숙임"
                    onclick="applyHeadTransplant('많이숙임')">많이숙임</button>
            <button class="btn head-preset-btn" data-preset="살짝올림"
                    onclick="applyHeadTransplant('살짝올림')">살짝올림</button>
            <button class="btn head-preset-btn" data-preset="많이올림"
                    onclick="applyHeadTransplant('많이올림')">많이올림</button>
        </div>

        <div class="btn-row" style="margin-top: 8px">
            <button class="btn btn-full" onclick="resetHeadTransplant()">초기화</button>
        </div>
    </div>
</div>
```

CSS 추가 필요 없음 — 기존 `.btn`, `.btn-group`, `.head-preset-btn.on`은 `.btn.on`과 동일하게 작동.

---

## 5. 프리셋 데이터 생성

### 5.1 생성 스크립트

이 스크립트는 **한 번만** 실행해서 `HEAD_PRESETS` 객체를 생성함.
입력: 원본.txt (8개 참조 JSON), 출력: JS 코드 (복붙용)

```python
#!/usr/bin/env python3
"""
generate_head_presets.py
원본.txt에서 8개 참조 JSON을 읽어 정규화된 HEAD_PRESETS JS 코드를 생성
"""
import json, math, sys

def split_jsons(text):
    text = text.replace('\r\n', '\n')
    objects, depth, start = [], 0, None
    for i, c in enumerate(text):
        if c == '{':
            if depth == 0: start = i
            depth += 1
        elif c == '}':
            depth -= 1
            if depth == 0 and start is not None:
                objects.append(json.loads(text[start:i+1]))
                start = None
    return objects

# 프리셋 이름 (원본.txt 내 JSON 순서와 일치)
PRESET_NAMES = [
    ("정면",     "정면"),
    ("살짝돌림", "살짝 오른쪽"),
    ("3/4뷰",   "3/4 오른쪽"),
    ("프로필",   "우측 프로필"),
    ("살짝숙임", "살짝 아래"),
    ("많이숙임", "많이 아래"),
    ("살짝올림", "살짝 위"),
    ("많이올림", "많이 위"),
]

HEAD_BODY_INDICES = [0, 14, 15, 16, 17]  # Nose, REye, LEye, REar, LEar

def normalize_preset(pose_json):
    """Neck 기준 정규화: (kp - neck) / nose_neck_distance"""
    person = pose_json['people'][0]
    kp = person['pose_keypoints_2d']
    face = person.get('face_keypoints_2d', [])

    neck_x, neck_y = kp[1*3], kp[1*3+1]
    nose_x, nose_y = kp[0*3], kp[0*3+1]
    scale = math.sqrt((nose_x - neck_x)**2 + (nose_y - neck_y)**2)

    if scale < 1:
        raise ValueError("Nose-Neck distance too small")

    # Body head keypoints
    body_head = {}
    for idx in HEAD_BODY_INDICES:
        x = (kp[idx*3] - neck_x) / scale
        y = (kp[idx*3+1] - neck_y) / scale
        c = kp[idx*3+2]
        body_head[idx] = [round(x, 5), round(y, 5), round(c, 3)]

    # Face 68 keypoints
    face_68 = []
    n_face = min(68, len(face) // 3)
    for i in range(n_face):
        x = (face[i*3] - neck_x) / scale
        y = (face[i*3+1] - neck_y) / scale
        c = face[i*3+2]
        face_68.append([round(x, 5), round(y, 5), round(c, 3)])

    return body_head, face_68

def main():
    with open(sys.argv[1], 'r') as f:
        poses = split_jsons(f.read())

    assert len(poses) == 8, f"Expected 8 poses, got {len(poses)}"

    print("const HEAD_PRESETS = {")
    for i, (key, label) in enumerate(PRESET_NAMES):
        body_head, face_68 = normalize_preset(poses[i])

        print(f'    "{key}": {{')
        print(f'        label: "{label}",')

        # body_head
        print(f'        body_head: {{')
        for idx, val in body_head.items():
            print(f'            {idx}: {val},')
        print(f'        }},')

        # face_68 (한 줄씩은 너무 길어서 5개씩 묶기)
        print(f'        face_68: [')
        for j in range(0, len(face_68), 5):
            chunk = face_68[j:j+5]
            line = ', '.join(str(pt) for pt in chunk)
            comma = ',' if j + 5 < len(face_68) else ''
            print(f'            {line}{comma}')
        print(f'        ]')

        comma = ',' if i < 7 else ''
        print(f'    }}{comma}')

    print("};")

if __name__ == '__main__':
    main()
```

### 5.2 실행 방법

```bash
python3 generate_head_presets.py 원본.txt > head_presets.js
```

출력된 `head_presets.js` 내용을 `dwpose_editor_logic.js`의 `const HEAD_PRESETS = {};` 자리에 붙여넣기.

---

## 6. 실행 순서 정리

### 6.1 다른 기능과의 상호작용

이식 시스템은 FK, 표정과 **독립적**으로 동작:

```
[원본 JSON 로드]
    │
    ├─ [머리 이식] → body head + face 교체 (transplantBasePose에서 복원 후 적용)
    │
    ├─ [FK 관절]   → body 회전 (fkBasePose에서 복원 후 적용)
    │
    ├─ [표정]      → face 미세 조정 (baseFaceData에서 복원 후 적용)
    │
    └─ [손 프리셋] → hand 교체 (handPresetBase에서 복원 후 적용)
```

**주의**: 이식 후 FK Head를 조작하면 이식된 머리가 FK base에 포함됨.
이것은 **의도된 동작** — 이식으로 방향 설정 → FK로 미세 기울기 조정.

### 6.2 base 충돌 방지

현재 `resetAllBases()`가 JSON 로드 시 호출됨 (line 756).
이식 base도 여기서 초기화되므로 **새 JSON 로드 시 자동 리셋**됨.

---

## 7. 향후 확장

### 7.1 어깨/상체 이식

같은 패턴으로 확장 가능:

```js
const SHOULDER_PRESETS = {
    "정면": {
        // Neck 기준 정규화된 RShoulder, LShoulder
        body_shoulders: {
            2: [norm_x, norm_y, conf],  // RShoulder
            5: [norm_x, norm_y, conf],  // LShoulder
        }
    },
    ...
};
```

### 7.2 전신 포즈 프리셋

DWPose 추출 → 정규화 → 프리셋화 파이프라인이 확립되면:
- "다리 꼬기", "한쪽 무릎 세우기" 등 하반신 프리셋
- "턱 괴기", "머리 짚기" 등 손+머리 복합 프리셋

### 7.3 커스텀 프리셋 추가 UI

에디터에서 현재 포즈의 특정 부위를 "프리셋으로 저장" 하는 기능.
→ localStorage 또는 ComfyUI 노드의 hidden widget에 저장.

---

## 8. 변경 파일 체크리스트

| 파일 | 변경 | 내용 |
|------|------|------|
| `dwpose_editor_logic.js` | 삭제 | Fake 3D 전체 (line 1676~1904) |
| `dwpose_editor_logic.js` | 추가 | `HEAD_PRESETS`, 이식 함수들 (같은 위치) |
| `dwpose_editor_logic.js` | 수정 | `resetAllBases()` — fake3dBasePose → transplantBasePose |
| `dwpose_editor_logic.js` | 수정 | `resetAllSliders()` — fake3d 초기화 → 이식 UI 초기화 |
| `dwpose_editor.html` | 삭제 | `#sec-fake3d` 섹션 (line 908~973) |
| `dwpose_editor.html` | 추가 | `#sec-head-transplant` 섹션 (같은 위치) |
| `dwpose_editor.js` | 변경 없음 | ComfyUI 통신 레이어는 그대로 |
| (신규) `generate_head_presets.py` | 생성 | 원본.txt → HEAD_PRESETS JS 코드 생성 |

---

## 9. 테스트 계획

1. **데이터 생성**: `generate_head_presets.py` 실행 → HEAD_PRESETS 코드 확인
2. **이식 정확도**: 에디터에서 각 프리셋 클릭 → 캔버스에서 머리 방향 시각 확인
3. **base/restore**: 프리셋 A → 프리셋 B → 초기화 → 원본 복원 확인
4. **FK 연계**: 이식 후 FK Head 슬라이더 → 이식된 머리가 추가 회전
5. **표정 연계**: 이식 후 시선/입 슬라이더 → 이식된 얼굴에서 정상 작동
6. **ComfyUI 연동**: Send to Node → DWPoseJSONRenderer → ControlNet → 생성 결과 확인

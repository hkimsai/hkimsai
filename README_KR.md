# 안녕하세요, 김하영 (Aaron)입니다

> [🇺🇸 English Version](README.md)

**AI/ML 엔지니어** 

최근에는 **의료 영상 데이터 파이프라인**, **5개 프레임워크를 활용한 모델 양자화**, **의료 AI를 위한 프라이버시 엔지니어링**을 다루고 있습니다. 

---

## 주요 작업 분야

대부분의 레포지토리는 회사/고객 IP로 인해 비공개입니다.
**각 프로젝트에서 기술적으로 흥미로운 부분** — 어려운 문제, 설계 트레이드오프, 엔지니어링 결정을 설명하겠습니다.

**라이브 데모를 보고 싶으시다면?** 각 프로젝트에 대해 직접 시연해 드릴 수 있습니다 — [aaronkim777@gmail.com](mailto:aaronkim777@gmail.com) 또는 [https://www.linkedin.com/in/hyk1/](https://www.linkedin.com/in/hyk1/)으로 연락 주세요.

---

## 주요 프로젝트

### 1. QLRO-Features — 의료 영상 DataOps 파이프라인

**84.8K LOC** · **40.3K LOC 테스트** · 7단계 파이프라인 · 2026년 1~2월

Raw DICOM 파일을 AI-ready 데이터셋으로 변환하며, HIPAA Safe Harbor 준수 및 NEMA Part 15 적합성을 목표로 합니다. 흥미로운 점은 파이프라인 자체가 아니라 그 아래의 엔지니어링입니다.

#### 기술 상세

**외적(Cross-product) 기반 기하학적 슬라이스 정렬.** DICOM 시리즈는 3D 재구성을 위해 공간 정렬이 필요하지만, 표준에서는 이를 명확하게 정의해두지 않았습니다. ImagePositionPatient (IPP)와 ImageOrientationPatient (IOP)를 추출하고, 행/열 방향 코사인의 외적으로 슬라이스 법선벡터를 유도한 뒤, 각 슬라이스의 IPP를 이 법선에 투영합니다. 정렬된 투영값은 InstanceNumber가 잘못되었거나 누락된 경우에도 물리적으로 올바른 Z축 정렬을 제공합니다 — 이는 생각보다 자주 발생합니다.

**52토큰 표준 시리즈 레이블링.** DICOM의 시리즈 설명은 자유 텍스트의 혼란("ax t1 post gad fs", "T1W_AX+C FAT SAT" 등)입니다. Strategy/Composite 패턴을 적용한 커스텀 YAML DSL로 규칙 기반 분류기를 구축했습니다. 엔진은 52개의 해부학/프로토콜/시퀀스 토큰을 정규표현식 패턴과 비교하여 `CT|CHEST|AXIAL|WITH_CONTRAST|SOFT_TISSUE` 같은 구조화된 레이블을 3단계 신뢰도 점수(HIGH/MEDIUM/LOW)와 구조화된 근거 체인과 함께 생성합니다. YAML DSL은 AND/OR/NOT 논리, 우선순위 정렬, 폴백 체인을 지원합니다.

**Monkey-patching을 통한 Presidio 800배 최적화.** Presidio의 `ImageRedactorEngine`은 OCR 바운딩 박스마다 `most_common_pixel_value()`를 호출하여 배경 채움색을 결정합니다 — 매번 전체 이미지 히스토그램 분석을 실행합니다. 프로파일링 결과 이미지당 ~700ms가 소요되었고, `collections.Counter`를 사용한 단일 호출 캐시 결과로 대체한 뒤 Presidio 클래스에 monkey-patch했습니다. 0.9ms로 감소. 핵심 인사이트는 동일 이미지 내 바운딩 박스 간 배경색이 변하지 않는다는 것이었습니다.

**fd 수준 C 확장 억제를 포함한 PaddleOCR 어댑터.** PaddleOCR의 C++ 백엔드는 Python `sys.stdout`을 우회하여 네이티브 파일 디스크립터(fd)로 직접 stdout/stderr에 기록합니다. 이를 억제하려면 Python 스트림 리다이렉션이 아닌 실제 파일 디스크립터(fd 1/2)에 대해 `os.dup2()`를 사용해야 했습니다. 어댑터(698 LOC)는 이중 확인 잠금(double-checked locking)을 사용한 스레드 안전 지연 초기화, uint16→uint8 DICOM 픽셀 정규화, 에어갭 환경을 위한 동적 모델 경로 해석도 처리합니다.

**다층 방어(Defense-in-depth) PHI 아키텍처.** 단일 비식별화 방법을 신뢰하지 않고, 파이프라인에 7개 접근법을 계층화합니다: (1) 번인 픽셀의 전체 텍스트 OCR 말소, (2) 말소된 텍스트의 사후 비PHI 분류, (3) 다운스트림 NER을 위한 비PHI 토큰의 `ImageComments` 주입, (4) 6개 커스텀 의료 날짜 인식기를 포함한 Presidio NER로 모든 자유 텍스트 메타데이터 태그 스크러빙, (5) SHA-256 결정론적 UID 대체, (6) `.secrets/` 디렉토리 격리(내보내기 불가), (7) 해시된 식별자로 구성된 추가 전용(append-only) JSONL 감사 추적.

**SHA-256 키 유도 날짜 이동.** 실제 날짜를 제거하면서 연구 내 종단적(longitudinal) 시간 관계를 보존하기 위해, `SHA256(patient_id + salt)`에서 결정론적 일수 오프셋을 유도하고, DA/TM/DT 값 표현 전체에 일관되게 적용하며, 동일 연구 내 모든 인스턴스가 동일한 양만큼 이동하도록 보장합니다. 이를 통해 실제 날짜를 노출하지 않고도 AI 학습을 위한 검사 간 시간 차이를 계산할 수 있습니다.

---

### 2. Q-Hub — 다종 산업 AI 양자화 & 거버넌스 플랫폼

**25K+ LOC** · FastAPI 서비스 3개 · 엔드포인트 21개 · 2025년 10~12월

Healthcare (기흉 감지), Finance (사기 탐지), Manufacturing (불량 검출) 전반의 AI 추론 비교를 위한 백엔드로, FP32/INT8 A/B 테스트 및 IBM WatsonX 거버넌스 통합을 포함합니다.

#### 기술 상세

**ONNX 그래프 수술: Swish→ReLU6 대체.** EfficientNet-B0 백본은 ONNX Runtime의 INT8 양자화기가 잘 처리하지 못하는 Swish (SiLU) 활성화를 사용합니다 (시그모이드 분기에서 양자화 노이즈 증폭). ONNX 그래프에서 `Sigmoid + Mul` 패턴(두 입력이 동일 소스를 공유하는 경우)을 감지하고, `Clip(Relu(x), 0, 6)`으로 대체하며, 에지를 재연결하는 그래프 수준 패스를 작성했습니다 — 프레임워크 API가 아닌 ONNX protobuf에서 직접 작업합니다. 정확도를 유지하면서 모델을 양자화 친화적으로 만들었습니다.

**커스텀 QAT 퓨전: Conv+BN+ReLU6.** PyTorch의 내장 QAT 퓨전은 Conv+BN+ReLU만 지원합니다. Swish를 ReLU6으로 대체했기 때문에, 학습 중 가짜 양자화 옵저버를 통해 클램프를 올바르게 전파하는 커스텀 `ConvBnReLU6` 퓨전 모듈을 작성해야 했습니다. 이것 없이는 양자화 인식 학습이 활성화 범위에 대한 잘못된 scale/zero-point를 학습하게 됩니다.

**추론 워커를 위한 3단계 코디네이터 패턴.** A/B 비교를 위해 FP32와 INT8 모델을 동시에 실행하는 것은 단순해 보이지만, 서로 다른 추론 지연시간, 공유 GPU 메모리 압력, 개별 예측을 올바른 모델에 귀속시키는 과제를 고려해야 합니다. `multiprocessing.Process`로 순수 CPU 격리, `psutil.cpu_affinity()` 고정, `multiprocessing.Queue`를 통한 IPC를 사용하는 생산자→코디네이터→소비자 패턴을 구현했습니다 — 직렬화된 배열이 아닌 이미지 파일 경로 전달(path-not-payload 패턴)로 IPC 오버헤드를 대폭 줄였습니다. pickle로 이미지 배치를 직렬화하는 것이 병목이었기 때문입니다.

**timm의 퓨즈드 BatchNormAct2d 분해.** `timm` 라이브러리는 추론 속도를 위해 BatchNorm과 activation을 단일 `BatchNormAct2d` 모듈로 퓨즈합니다. 그러나 ONNX 내보내기와 양자화 도구는 이들이 분리되어 있기를 기대합니다. 모듈 트리를 순회하며 각 `BatchNormAct2d`를 명시적 `BatchNorm2d` → `act_fn` 시퀀스로 대체하고 학습된 파라미터를 복사하는 분해 패스를 작성했습니다 — timm 모델을 양자화를 위해 ONNX로 내보내기 전까지는 나타나지 않는 문제입니다.

**금융 특성을 위한 십진수 자릿수 분해.** 신용카드 거래 금액은 자릿수 패턴에 신호를 담고 있습니다 (예: .00으로 끝나는 금액과 .99는 사기 확률이 다름). 원시 금액만 사용하는 대신, 모듈러 연산 `digit_i = (amount // 10^i) % 10`을 사용하여 각 거래 값을 자릿수별 특성으로 분해합니다 — 네트워크가 원시 실수에서 이 분해를 스스로 학습하도록 맡기지 않고, 자릿수 수준 패턴에 대한 명시적 접근을 제공합니다.

---

### 3. CellViT++ — 두경부암 세포 분류

**1.2K+ LOC** · W&B 14회 실행 · 2025년 8~9월

632M 파라미터 ViT-SAM-H 세포 분할 모델에 대한 평가 파이프라인 및 INT8 양자화. 비공개 두경부암 병리 데이터에 적용.

#### 기술 상세

**bitsandbytes를 위한 SAM-H 퓨즈드 QKV 분해.** SAM-H 비전 트랜스포머는 퓨즈드 QKV 프로젝션을 사용합니다 — 단일 `nn.Linear(1280, 3840)`이 연결된 Q, K, V를 출력합니다. bitsandbytes의 `LLM.int8()` 혼합 정밀도 양자화는 FP16으로 유지할 행(이상치 특성)을 결정하기 위해 개별 행렬 차원을 알아야 합니다. 각 퓨즈드 프로젝션을 3개의 별도 `nn.Linear(1280, 1280)` 모듈로 분할하고, 가중치 슬라이스를 복사하며, 예상 어텐션 입력 형식에 맞게 4D→3D 텐서 재형성을 추가했습니다 — 32개 트랜스포머 블록 전체에 걸쳐. 이것 없이는 bitsandbytes가 퓨즈드 행렬을 단일 단위로 양자화하여 이상치 차원을 분리할 수 없었습니다.

**제로샷 도메인 전이 평가.** 모델은 PanNuke (19개 조직 유형, ~200K 핵)에서 사전 학습되었지만, 한 번도 본 적 없는 두경부암 세포를 분류할 필요가 있었습니다. CellViT의 GeoJSON 세포 예측을 로드하고, 모델의 6클래스 체계를 4클래스 스키마에 매핑하며, 병리학자 주석에 대해 클래스별 F1/재현율/정밀도를 계산하는 평가 도구를 구축했습니다. 결과 — F1=0.77, 재현율=91.4% — 는 초기 배포에서 파인튜닝을 건너뛸 수 있을 만큼 충분했습니다.

**교차 데이터셋 레이블 통일.** 6개 병리 조직학 데이터셋 (PanNuke, CoNSeP, MoNuSAC, CryoNuSeg, OCELOT, TIGER)은 각각 다른 세포 유형 분류 체계를 가지고 있습니다. 수동으로 검증된 매핑 테이블을 사용하여 4클래스 스키마 (Neoplastic, Non-neoplastic, Inflammatory, Dead)로 통합했으며, 이는 학습 일관성과 데이터셋 간 평가 지표 비교를 위해 필수적이었습니다.

---

## 연락처

- aaronkim777@gmail.com
- [LinkedIn](https://www.linkedin.com/in/hyk1/)
- [arXiv 논문](https://arxiv.org/abs/2407.12514) · [IEEE 논문](https://ieeexplore.ieee.org/document/10352142)

---

*모든 프로젝트 레포지토리는 회사/고객 IP로 인해 비공개입니다. 각 프로젝트에 대한 상세 기술 보고서와 아키텍처 문서를 보유하고 있으며, 대화를 통해 구체적인 내용을 설명해 드릴 수 있습니다.*

# Runners Feed GPU 영상 분석 PoC 검증

## 1. 문서 목적

이 문서는 Runners Feed의 영상 분석 병목을 확인하고, OCI CPU에서 처리하던 `video_analysis`를 RunPod GPU로 분리할 수 있는지 검증한 과정을 정리한다.

여기서 PoC는 완성된 제품이나 모델 품질을 증명하는 문서가 아니다. 다음 기술적 가능성을 확인하는 데 목적이 있다.

- 원격 GPU를 사용했을 때 네트워크 비용을 포함한 전체 영상 분석 시간이 줄어드는가?
- CPU와 GPU 결과의 차이를 측정하고, 허용 여부를 판단할 수 있는가?
- RunPod에서 생성한 산출물을 OCI가 검증하고 후처리할 수 있는가?
- 실험 구조를 실제 팀 서비스의 비동기 작업 흐름에 연결할 수 있는가?

## 2. 문제 발견

초기 영상 처리 작업은 입력 다운로드부터 결과 업로드까지 여러 단계로 구성됐다. Grafana 기록에서 자세 추론이 가장 큰 단일 병목이었지만, 모델 추론을 제외한 프레임 처리, 렌더링, 영상 합성 등의 시간도 작지 않았다.

따라서 목표를 단순히 “GPU 추론이 CPU보다 빠른가?”로 두지 않았다. 실제 서비스에 의미가 있으려면 영상 전송과 결과 수신을 포함한 전체 처리 경로가 빨라져야 했다.

## 3. 검증 가설

### 가설 1. GPU는 자세 추론 연산을 가속할 수 있다

RunPod RTX 2000 Ada에서 RTMDet과 RTMPose ONNX 모델이 `CUDAExecutionProvider`를 사용하도록 구성했다. 임의 입력과 실제 프레임 시험에서 GPU 연산 가능성과 속도 차이를 먼저 확인했다.

### 가설 2. 추론만 GPU로 옮기는 것으로는 전체 서비스 시간이 충분히 줄지 않을 수 있다

임시 사이트 A/B 시험에서 사람 검출·자세 추론 구간은 53.4% 줄었지만 전체 처리시간은 76.56초에서 70.91초로 7.4%만 감소했다. 렌더링, 영상 합성과 리포트 생성이 OCI에 그대로 남았기 때문이다.

이 결과를 근거로 GPU의 책임 범위를 추론에만 한정하지 않고, 디코딩부터 추론·추적·렌더링·인코딩까지 포함하는 전체 `video_analysis`로 확장했다.

### 가설 3. 전체 `video_analysis`를 GPU로 옮기면 전송 비용을 포함해도 CPU보다 빨라질 수 있다

OCI에서 고정 영상을 RunPod로 업로드하고, CUDA 분석과 NVENC 인코딩을 수행한 뒤 결과를 다시 다운로드하는 왕복시간을 측정했다.

### 가설 4. GPU 분리 구조를 팀 서비스의 비동기 작업으로 연결할 수 있다

격리 실험이 성공하더라도 실제 서비스의 작업 상태, 재시도, Object Storage 산출물과 OCI 후처리를 연결하지 못하면 제품에 사용할 수 없다. 따라서 작업 계약과 산출물 검증을 포함한 비동기 서버를 구현하고 팀 저장소 반영 여부를 확인했다.

## 4. 실험 조건

### 격리 성능 시험

- 입력 A: 1280×720, 60fps, 255프레임, 4.25초
- 입력 B: 1280×720, 29.97fps, 631프레임, 21.05초
- CPU: OCI 운영 Worker 이미지의 격리 복사본
- GPU: RunPod RTX 2000 Ada, ONNX Runtime CUDA
- 반복 횟수: 방식별 2회
- 대표값: 두 실행의 중앙값
- GPU 측정 범위: OCI 업로드, RunPod 분석·렌더링·NVENC 인코딩, 결과 다운로드

CPU 개선 방식은 OpenCV 중간 영상을 제거하고 렌더링 프레임을 FFmpeg `libx264 veryfast CRF23`으로 직접 전달했다. GPU 방식은 모델을 한 번 적재한 warm CUDA 프로세스와 `h264_nvenc p4 cq23`을 사용했다.

### 운영 통합 시험

격리 시험과 운영 통합 시험은 측정 범위가 다르다. 운영 통합 시험에는 사이트, OCI API·DB·Celery, Object Storage 전송, RunPod 분석, OCI 후처리와 결과 표시가 포함된다. 따라서 두 시험의 시간을 같은 벤치마크처럼 직접 비교하지 않는다.

## 5. 판정 기준

아래 기준은 당시 기록에서 확인되는 기술적 판단 기준을 현재 문서에서 정리한 것이다. 실험 전에 별도 승인 문서로 고정한 사전 등록 성공 기준은 아니다.

- GPU 왕복시간이 동일 입력의 기존 CPU 처리시간보다 짧아야 한다.
- 출력 영상의 프레임 수, FPS, 해상도와 재생 가능 여부를 확인할 수 있어야 한다.
- 반복 실행 결과의 예측 해시와 산출물을 비교할 수 있어야 한다.
- CPU·GPU 관절 좌표와 인원 검출 차이를 수치로 확인하고 미해결 차이를 기록해야 한다.
- 요청과 manifest의 Job ID, attempt ID, 모델·소스 해시가 일치할 때만 OCI 후처리로 전달해야 한다.
- 실제 팀 서비스에서 작업 등록부터 결과 표시까지 완료 상태를 확인할 수 있어야 한다.

## 6. 결과

### 전체 영상 분석 성능

| 입력 | 기존 OCI CPU | 개선 OCI CPU | RunPod GPU 왕복 | 기존 CPU 대비 GPU |
|---|---:|---:|---:|---:|
| 720p·60fps·255프레임 | 16.876초 | 12.670초 | 8.773초 | 48.01% 단축 |
| 720p·29.97fps·631프레임 | 83.351초 | 73.965초 | 17.461초 | 79.05% 단축 |

개선 CPU와 비교하면 GPU 왕복시간은 각각 30.75%, 76.39% 짧았다. 짧은 영상에서는 GPU 계산보다 업로드, 응답 준비와 다운로드 같은 고정 비용의 비중이 컸다.

### 결과 일관성

- GPU 반복 실행 두 번은 두 입력 모두 동일한 예측 SHA-256을 생성했다.
- 60fps 입력은 CPU·GPU 결과 프레임과 인원 수가 같았고 평균 관절 차이는 0.1425px였다.
- 29.97fps 입력은 631개 결과 프레임을 유지했지만 349번 프레임에서 CPU는 2명, GPU는 1명을 검출했다.
- CPU의 `libx264 CRF23`과 GPU의 `h264_nvenc CQ23`은 동일한 화질 척도가 아니다. 이번 시험으로 화질 동등성을 입증하지 않았다.

### 운영 통합

초기 Object Storage 기반 통합은 팀의 브랜치 검토 순서를 맞추기 위해 PR #28에서 한 차례 되돌렸다. 이후 처리 경계를 유지한 비동기 RunPod 서버를 다시 구현했고 PR #34를 통해 팀 저장소 `main`에 병합했다.

하지만 GPU 서버 코드가 병합됐다는 사실만으로 앱 적용이 끝난 것은 아니었다. 실제 OCI·RunPod·앱 경로를 연결하는 과정에서 다음과 같은 문제가 드러났다.

- RunPod 프록시 요청에 명시적인 User-Agent가 필요했다.
- PostgreSQL UUID 객체가 JSON 직렬화되지 않아 RunPod 호출 전에 작업이 실패했다.
- OCI 배포 서버의 Docker 디스크 사용량이 97%까지 올라 새 이미지 배포가 중단됐다.
- systemd unit 동기화 과정에서 배포 계정의 권한 문제가 발생했다.
- 이후에도 polling 간격, 모델 통합과 rtmlib 호환성에 대한 후속 보완이 필요했다.

따라서 “GPU에서 빠르게 처리됐다”와 “앱에서 안정적으로 사용할 수 있었다”는 서로 다른 검증 항목이었다. 전자는 격리 성능 시험으로 확인했지만, 후자는 API 계약, 상태 관리, 배포 환경과 모델 호환성을 함께 해결해야 했다.

후속 운영 릴리스에서 서로 다른 길이와 FPS의 입력 세 건이 모두 완료됐다.

| 운영 입력 | 결과 | 총 처리시간 | 확인된 병목 |
|---|---|---:|---|
| 720p·60fps·255프레임 반복 | SUCCESS | 24.8초 | `video_analysis` 4.8초 |
| 720p·25fps·136프레임 | SUCCESS | 18.8초 | 입력 다운로드 3.0초 |
| 720p·30fps·1,105프레임 | SUCCESS | 44.1초 | `video_analysis` 17.6초 |

표본 세 건의 평균 처리시간은 약 29.22초였다. 이 결과는 경로가 실제 서비스에서 동작한다는 증거이며, 표본 수가 작으므로 일반적인 성능이나 SLA를 보장하는 수치로 사용하지 않는다.

## 7. PoC에서 MVP에 반영한 결정과 남은 문제

### 처리 경계

```text
모바일 앱
→ OCI API·Celery
→ RunPod 비동기 video_analysis
→ Object Storage 결과 manifest
→ OCI 후처리
→ 모바일 결과
```

RunPod은 사람 검출, 자세 추정, 추적, 렌더링과 NVENC 인코딩을 담당한다. OCI는 작업 조정, 계약 검증, 피처·리포트 후처리와 사용자 결과 제공을 담당한다.

### 비동기 작업 계약

내가 구현한 주요 범위는 다음과 같다.

- `POST /v4/storage-video-analysis` 작업 등록과 HTTP 202 응답
- `remote_job_id` 기반 상태 polling
- `queued`, `running`, `complete`, `failed` 상태 계약
- 동일 `attempt_id` 재전달 시 기존 작업을 재사용하는 중복 추론 방지
- SQLite 기반 작업 상태 보존과 Bearer token 인증
- OCI Object Storage 산출물 업로드
- CUDA·모델 release 증거를 포함한 결과 manifest 생성

GPU attempt·DB 상태 계약, polling 간격 단축, 모델 통합과 rtmlib 호환성 등의 후속 안정화는 팀 공동 작업으로 이어졌다.

### 앱 적용에서 확인한 점

PoC에서 측정한 GPU 속도를 앱에 그대로 대입할 수는 없었다. 실제 사용자 흐름에는 네트워크 전송, 큐 대기, Object Storage 다운로드·업로드, OCI 후처리와 결과 표시가 추가된다. 짧은 영상에서는 GPU 계산보다 입력 다운로드가 더 긴 병목으로 나타나기도 했다.

또한 앱 적용 과정에서 발생한 오류 중 상당수는 GPU 연산 성능과 무관했다. 데이터 형식, 비동기 상태 계약, 배포 디스크와 서비스 권한처럼 시스템 경계에서 발생한 문제였다. 이 경험을 통해 모델이나 GPU의 단일 성능만으로 제품의 처리시간과 안정성을 설명할 수 없다는 점을 확인했다.

## 8. 결론: 기술 목표 달성과 운영 적용에서 확인한 과제

PoC를 통해 원격 GPU가 자세 추론만 가속하는 것으로는 전체 서비스 개선이 제한적이라는 사실을 확인했다. 이후 GPU 책임 범위를 전체 `video_analysis`로 확장했고, 네트워크 왕복을 포함한 격리 시험에서 기존 CPU 대비 48.01~79.05%의 처리시간 단축을 관측했다.

GPU 처리 성능에 대한 가설은 확인했다. 그러나 이 결과만으로 앱 적용이 완료됐다고 판단할 수는 없었다. 실제 통합 과정에서는 요청 형식, 작업 상태, Object Storage 산출물, 배포 환경과 모델 호환성 문제가 발생했고, 이를 해결하기 위한 반복적인 수정과 팀 단위의 후속 안정화가 필요했다.

나는 성능 시험을 근거로 Object Storage와 manifest를 사용하는 비동기 RunPod 서버를 구현했고, 해당 코드는 PR #34를 통해 팀 저장소에 병합됐다. 이후 앱에서 안정적으로 사용하기 위한 DB 상태 계약, polling, 모델 통합과 호환성 보완은 팀 공동 작업으로 이어졌다. 따라서 “PR이 병합됐다”와 “제품 적용이 완전히 끝났다”를 같은 의미로 사용하지 않는다.

이 PoC의 기술적 목표는 달성했다. GPU로 전체 영상 분석 시간을 줄일 수 있고, 비동기 서비스 구조로 연결할 수 있다는 점을 확인했다. 다만 운영 적용 과정에서는 GPU 성능 외의 시스템 경계 문제가 드러났고, 안정적인 제품 적용을 위해 후속 보완이 필요했다.

따라서 이 작업의 핵심 성과는 단순히 “GPU가 빠르다”는 결론이 아니다. 병목 측정으로 처리 경계를 다시 정하고, 성능·정확성·운영 계약을 구분해 검증했으며, 앱 적용 과정에서 드러난 문제를 후속 구현과 팀 작업으로 연결한 것이다.

다만 29.97fps 입력의 인원 검출 차이, 화질 동등성, 적은 반복 횟수와 운영 표본 수는 남은 한계다. 이 PoC는 기술 구조의 가능성을 확인했으며 모델 품질 승인이나 의료적 정확성을 증명하지 않는다.

## 9. 근거 자료

- [CPU·GPU 성능 검증 저장소](https://github.com/RunnersFeed-Project-Oracle-boot-camp/CPU-GPU-Speed-Compare-by-RunPod)
- [상세 성능 결과](https://github.com/RunnersFeed-Project-Oracle-boot-camp/CPU-GPU-Speed-Compare-by-RunPod/blob/main/docs/RESULTS.md)
- [전체 실험 이력](https://github.com/RunnersFeed-Project-Oracle-boot-camp/CPU-GPU-Speed-Compare-by-RunPod/blob/main/docs/PROJECT_HISTORY.md)
- [측정 경계와 아키텍처](https://github.com/RunnersFeed-Project-Oracle-boot-camp/CPU-GPU-Speed-Compare-by-RunPod/blob/main/docs/ARCHITECTURE.md)
- [재현 범위와 실행 전제](https://github.com/RunnersFeed-Project-Oracle-boot-camp/CPU-GPU-Speed-Compare-by-RunPod/blob/main/docs/REPRODUCE.md)
- [PR #23 · Object Storage 기반 RunPod 영상 분석 파이프라인](https://github.com/Temu-F4/Runners_Feed/pull/23)
- [PR #28 · 초기 통합 변경 회수](https://github.com/Temu-F4/Runners_Feed/pull/28)
- [PR #34 · 비동기 RunPod 영상 분석 서버](https://github.com/Temu-F4/Runners_Feed/pull/34)

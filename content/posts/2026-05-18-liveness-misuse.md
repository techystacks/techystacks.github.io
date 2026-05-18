---
title: "Liveness probe를 달지 말아야 할 때 — 재시작 트리거가 만드는 장애 전파 패턴"
date: 2026-05-18
draft: false
tags: ["liveness", "startup", "crashloopbackoff", "probe", "cascading-failure"]
categories: ["워크로드 수명주기"]
k8s_version: "1.35"
cluster: "funny-pop-mountain (EKS Auto Mode, us-east-1)"
difficulty: ["중급"]
---

이 글은 [Pod Lifecycle 공식 문서](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#container-probes), [KEP-950 (Startup Probe)](https://github.com/kubernetes/enhancements/tree/master/keps/sig-node/950-liveness-probe-holdoff), [KEP-4603 (Reduce Default API QPS Limits for Container Restart Backoff)](https://github.com/kubernetes/enhancements/tree/master/keps/sig-node/4603-tune-crashloopbackoff)을 참고하여 정리하였습니다.

지난 글에서 readiness probe 실패가 EndpointSlice를 어떻게 갱신하는지 살펴봤습니다. 이번에는 짝을 이루는 또 다른 probe인 livenessProbe를 다룹니다. liveness는 readiness와 비슷하게 생긴 YAML 블록이지만 실제 동작은 정반대입니다. readiness가 트래픽을 잠시 끊는 신호라면, liveness는 컨테이너 자체를 죽이는 트리거입니다. 이번 글에서는 EKS Auto Mode `funny-pop-mountain`에서 세 가지 안티패턴과 그 대안을 측정한 결과를 소개하고자 합니다.

## liveness probe는 헬스 체크가 아닙니다

많은 입문서가 liveness probe를 "헬스 체크"라고 소개하지만 이 표현은 오해를 부릅니다. liveness probe가 실패했을 때 일어나는 일은 모니터링도 아니고 트래픽 차단도 아니고 알림 전송도 아닙니다. 바로 `kubelet`이 해당 컨테이너에 SIGTERM을 보내고, terminationGracePeriodSeconds가 지난 뒤에도 살아 있으면 SIGKILL로 죽인 다음 다시 띄우는 동작입니다. 단순한 재시작 트리거이고, 그 이상도 이하도 아닙니다.

이렇게 정의하면 여러 가지 안티패턴이 자명해집니다. 재시작이 도움이 되지 않는 상황에 liveness를 다는 것은 의미가 없거나 해롭습니다. 부팅 시간이 길어 readiness가 되기 전에 liveness가 실패하는 구성에서는 앱을 영원히 죽임당하게 만듭니다. 외부 의존성을 liveness로 검증하면 의존성 한 개의 일시 장애가 모든 replica의 동시 재시작으로 증폭됩니다. 그렇다면 이 두 가지 안티패턴이 실제 클러스터에서 어떻게 드러날까요?

세 가지 시나리오로 측정합니다. 안티패턴 두 개와 올바른 분리 한 개입니다. 각 시나리오는 lab-liveness-misuse 네임스페이스의 별도 Deployment에 담겨 있고, agnhost 이미지로 부팅 시간 8초를 시뮬레이션합니다.

## 시나리오 A — initialDelaySeconds가 부팅보다 짧으면 영원히 죽습니다

`too-impatient` Deployment는 컨테이너가 시작되고 8초 뒤에 `/tmp/ready` 파일을 만듭니다. 그 파일의 존재 여부를 livenessProbe가 확인하는데, 설정은 다음과 같습니다.

```yaml
livenessProbe:
  exec:
    command: ["cat", "/tmp/ready"]
  initialDelaySeconds: 2
  periodSeconds: 2
  failureThreshold: 3
```

`initialDelaySeconds: 2` 뒤로 `periodSeconds: 2`마다 probe가 실행되니, 첫 실패는 t=2s, 두 번째는 t=4s, 세 번째는 t=6s에 발생합니다. failureThreshold가 3이므로 t=6s 시점에 컨테이너가 unhealthy로 확정됩니다. 그런데 앱 자체는 t=8s가 되어야 ready 파일을 만듭니다. 즉 매 부팅마다 확정 죽음이 ready 직전 2초를 남기고 떨어지는 구조입니다.

다음 출력은 4분 가까이 진행된 재시작 루프입니다.

```
pod=too-impatient-54b46bd-8j426
t+1358ms   restartCount=0  startedAt=2026-05-18T06:06:17Z
t+8757ms   restartCount=1  finishedAt=06:06:28Z(Error exit=137) startedAt=06:06:28Z
t+21356ms  restartCount=2  finishedAt=06:06:40Z(Error exit=137) startedAt=06:06:40Z
t+34193ms  restartCount=3  finishedAt=06:06:52Z(Error exit=137) startedAt=06:06:52Z
t+70472ms  restartCount=4  finishedAt=06:07:04Z(Error exit=137) startedAt=06:07:28Z
t+139168ms restartCount=5  finishedAt=06:07:40Z(Error exit=137) startedAt=06:08:37Z

NAME                          READY   STATUS             RESTARTS      AGE
too-impatient-54b46bd-8j426   0/1     CrashLoopBackOff   5 (54s ago)   3m26s
```

각 컨테이너의 수명은 약 12초입니다. 부팅에 8초, liveness 실패 확정에 6초, terminationGracePeriodSeconds(5초)가 채 끝나기 전에 SIGKILL이 떨어지면서 exit code 137로 종료됩니다. 5번의 재시작 동안 한 번도 ready 파일을 만들 시간을 받지 못합니다.

흥미로운 부분은 재시작 사이의 backoff입니다. 처음 3번은 즉시 재시작되었습니다. 4번째 재시작 사이에 24초의 대기가 들어갔고, 5번째 사이에는 57초의 대기가 들어갔습니다. CrashLoopBackOff의 지수 backoff가 4번째부터 작동하는 모양인데, 이는 Kubernetes 1.32에서 도입된 새 backoff 정책의 거동입니다. 이전 정책은 10초로 시작해 두 배씩 늘려 5분에서 멈추는 식이었는데, 새 정책은 1초로 시작해 컨테이너가 일정 시간 살아 있으면 backoff가 초기화되는 형태로 바뀌었습니다. 이번 실험에서 처음 3번이 즉시 재시작된 이유는 각 컨테이너가 12초씩 살아 있어 backoff 카운터가 매번 초기화되었기 때문입니다.

이 시나리오의 본질은 liveness probe가 부팅 단계에까지 적용되어 있다는 점입니다. liveness는 "이 컨테이너가 망가졌으니 재시작이 필요하다"는 신호인데, 부팅이 끝나지 않은 상태는 망가진 게 아닙니다. 이 구분을 만들기 위해 Kubernetes 1.18에서 startupProbe라는 별도 probe가 추가되었습니다. 시나리오 C에서 같은 앱을 startupProbe로 보호했을 때의 차이를 보여드립니다.

## 시나리오 B — 외부 의존성을 liveness로 검증하면 장애가 증폭됩니다

`cascading` Deployment는 replica 3개로 구성되며, livenessProbe가 같은 namespace의 `dep` Service를 HTTP로 호출합니다.

```yaml
livenessProbe:
  exec:
    command:
      - sh
      - -c
      - "wget -q -T 1 -O - http://dep.lab-liveness-misuse.svc.cluster.local/ >/dev/null"
  initialDelaySeconds: 5
  periodSeconds: 2
  failureThreshold: 3
```

dep는 데이터베이스나 API 게이트웨이처럼 cascading이 의존하는 외부 서비스를 흉내 낸 것입니다. dep가 정상이면 cascading의 liveness는 항상 통과합니다. 그런데 dep를 31초 동안 다운시켰다가 복구하는 실험에서 다음 결과가 나왔습니다.

```
T0_epoch_ms=1779084655824  (dep scaled to 0)
T1_epoch_ms=1779084687313  (dep scaled back to 1, +31489ms)

=== cascading pod restart timeline ===
  T0   -591ms  pod=...5d8x8  rc=0  state=running
  T0   -590ms  pod=...m7kz2  rc=0  state=running
  T0   -590ms  pod=...nzksr  rc=0  state=running
  T0  +8690ms  pod=...5d8x8  rc=1  state=running
  T0  +8705ms  pod=...nzksr  rc=1  state=running
  T0  +9707ms  pod=...m7kz2  rc=1  state=running
  T0 +21783ms  pod=...m7kz2  rc=2  state=running
  T0 +21801ms  pod=...5d8x8  rc=2  state=running
  T0 +21967ms  pod=...nzksr  rc=2  state=running
  T1  +1335ms  pod=...m7kz2  rc=3  state=running
  T1  +1362ms  pod=...5d8x8  rc=3  state=running
  T1  +1518ms  pod=...nzksr  rc=3  state=running
```

dep를 다운시키고 단 9초 만에 cascading의 모든 replica가 첫 번째로 죽었고, 22초 시점에 두 번째로 죽었습니다. 세 replica의 재시작 시각이 1초 안에 모여 있습니다. dep를 복구한 직후에는 진행 중이던 재시작 사이클의 잔여로 또 한 번씩 재시작이 발생했습니다. 31초의 외부 의존성 장애가 9개의 동시 재시작 이벤트로 증폭된 것입니다.

여러분의 실제 운영에서 이런 패턴이 어떤 결과를 부르는지 상상해보시기 바랍니다. 데이터베이스가 30초 동안 연결을 거부하는 이벤트(예: 인증 토큰 만료, 일시적 네트워크 파티션, 백업 작업으로 인한 응답 지연)가 발생합니다. liveness가 DB 연결을 검증하고 있다면, DB가 복구되는 순간 모든 클라이언트 Pod이 동시에 다시 시작합니다. 재시작된 Pod이 일제히 DB에 연결을 시도하면서 thundering herd가 발생하고, 갓 복구된 DB가 다시 과부하에 빠지는 2차 장애로 이어집니다. liveness 하나가 단일 의존성 장애를 클러스터 전체 장애로 증폭시킨 셈입니다.

이 패턴의 교정은 단순합니다. liveness probe는 컨테이너 안에서 닫혀야 합니다. 외부 호출이 들어가는 endpoint를 liveness로 쓰지 않습니다. 외부 의존성의 건강 상태는 readiness probe로 표현하거나, 별도의 alerting 시스템으로 모니터링합니다. readiness 실패는 트래픽만 잠시 끊을 뿐 컨테이너를 죽이지 않으므로 cascading 패턴이 발생하지 않습니다.

## 시나리오 C — startup으로 부팅을 보호하고 liveness는 self-check로 좁힙니다

같은 8초 부팅 시간을 갖는 `well-bounded` Deployment는 다음 구성으로 안정적으로 떠올랐습니다.

```yaml
startupProbe:
  exec:
    command: ["cat", "/tmp/ready"]
  periodSeconds: 1
  failureThreshold: 30
livenessProbe:
  httpGet: {path: /healthz, port: 8080}
  periodSeconds: 5
  failureThreshold: 3
```

startupProbe가 적용된 Pod은 startup이 성공하기 전까지 다른 probe들이 일체 실행되지 않습니다. 그래서 부팅 8초 동안 livenessProbe는 휴면 상태로 대기하고, 부팅이 끝나 startup이 통과하면 그때부터 liveness가 5초 주기로 self-check를 시작합니다. 결과는 다음과 같습니다.

```
pod=well-bounded-dd75446fc-d6nr6
=== 60s status timeline ===
  t+5s   phase=Running  ready=false  started=false  state=running  restarts=0
  t+10s  phase=Running  ready=true   started=true   state=running  restarts=0
  t+15s  phase=Running  ready=true   started=true   state=running  restarts=0
  ... (이하 동일) ...
  t+60s  phase=Running  ready=true   started=true   state=running  restarts=0

Events:
  Warning  Unhealthy  2m13s (x8 over 2m20s)  kubelet
    Startup probe failed: cat: can't open '/tmp/ready': No such file or directory
```

startup probe가 8번이나 실패했지만(이는 부팅 첫 8초간의 자연스러운 실패) failureThreshold가 30이라서 재시작이 발생하지 않았습니다. t+10s 시점에 startup이 통과해 `started=true, ready=true`가 되었고, 이후 60초 동안 재시작 0회로 안정 상태를 유지했습니다. 시나리오 A와 정확히 같은 부팅 시간, 정확히 같은 ready 파일 메커니즘이지만 결과는 정반대입니다.

차이는 단 한 가지입니다. startupProbe와 livenessProbe의 책임을 분리한 것입니다. startupProbe는 "부팅이 충분히 길어질 수 있다"를 표현하고 충분한 failureThreshold로 부팅을 보호합니다. livenessProbe는 부팅 이후의 정상 동작 단계에서만 실행되며, 외부 의존성이 아닌 컨테이너 자체의 상태(/healthz 같은 self-check)만을 검증합니다. 이 분리가 시나리오 A의 무한 루프와 시나리오 B의 장애 증폭 두 안티패턴을 동시에 막습니다.

## Kubernetes는 이 문제를 어떻게 다듬어왔는가

probe 시스템 자체는 Kubernetes 초창기부터 존재했지만 livenessProbe와 readinessProbe 두 가지 만으로는 위 안티패턴들을 표현할 수 없었습니다. 부팅이 긴 앱에서는 운영자가 livenessProbe의 `initialDelaySeconds`를 충분히 크게 잡거나, 아니면 부팅 시간을 짧게 만들도록 앱을 고치거나 둘 중 하나를 선택해야 했습니다. `initialDelaySeconds`를 크게 잡으면 정상 동작 중 발생한 진짜 행 상태에 대한 감지도 그만큼 느려지는 문제가 있었습니다.

Kubernetes 1.16에서 startupProbe가 알파로 도입되어 1.18에서 GA로 자리 잡으면서 이 트레이드오프가 사라졌습니다. 부팅 단계와 정상 동작 단계의 probe를 따로 표현할 수 있게 된 것입니다. startup이 통과한 뒤에야 liveness가 동작하므로, 운영자는 부팅에 너그럽고 정상 동작에는 엄격한 probe 정책을 안전하게 만들 수 있습니다.

CrashLoopBackOff의 backoff 정책 자체도 1.32에서 다듬어졌습니다. 이전 정책은 첫 실패에서 10초를 기다리고 매 실패마다 두 배로 늘려 최대 5분에서 멈추는 식이었는데, 새 정책은 1초로 시작하고 컨테이너가 일정 시간 정상 동작하면 backoff를 초기화합니다. 빠르게 복구할 수 있는 일시적 실패에는 더 빠른 재시도를 주고, 진짜로 망가진 상태에서는 여전히 긴 backoff로 보호하는 균형 잡힌 정책입니다. 이번 실험에서 처음 3번의 재시작이 즉시 일어난 이유가 바로 이 새 정책 때문입니다. 컨테이너가 12초씩 살아 있어 매번 backoff가 초기화되었습니다.

## EKS Auto Mode 맥락 — feature gate는 보이지 않지만 거동은 관찰 가능합니다

EKS Auto Mode의 `funny-pop-mountain`은 1.35 기준이고 컨테이너 런타임은 containerd 2.1.6입니다. CrashLoopBackOff의 새 정책을 제어하는 feature gate는 Auto Mode에서 직접 확인할 수 없지만(노드 SSH 접근이 제한되어 있고 kubelet 설정 파일을 직접 보기 어렵습니다), 위 실험의 거동(처음 3번 즉시 재시작 → 4번째 24초 → 5번째 57초)에서 새 정책이 활성화되어 있음을 추론할 수 있습니다. 이전 정책이라면 첫 재시작부터 10초의 대기가 있어야 합니다.

또 한 가지 EKS 환경에 특유한 점은 cascading 시나리오의 영향 범위입니다. EKS Auto Mode 위에서 외부 의존성으로 자주 쓰는 것이 RDS 인스턴스인데, RDS는 minor version upgrade나 storage scaling 같은 운영 작업에서 짧은 connection refused 윈도우를 만듭니다. liveness가 RDS 연결을 검증하는 구성이라면 이 짧은 윈도우 동안 Pod 전체가 재시작에 들어가게 되고, 복구 직후 일제히 RDS에 다시 연결하면서 connection storm을 만듭니다. 운영 작업의 일환으로 발생한 30초 정도의 짧은 장애가 더 큰 2차 장애로 이어지는 패턴입니다.

실험이 끝나면 `kubectl delete namespace lab-liveness-misuse` 한 줄로 정리합니다.

## 맺음말

livenessProbe는 헬스 체크가 아니라 재시작 트리거입니다. 이 정의 하나만 명확히 잡아도 부팅이 짧지 않은 앱에 짧은 `initialDelaySeconds`를 다는 실수와, 외부 의존성을 liveness로 검증해 cascading failure를 만드는 실수 두 가지를 피할 수 있습니다. 시나리오 A와 시나리오 B는 입문서의 그림 한 장으로는 짚어주기 어려운 안티패턴이지만, 실측해보면 5분 안에 5번의 재시작이 발생하거나 31초의 의존성 장애가 9번의 동시 재시작으로 증폭되는 양상이 분명히 드러납니다.

운영 권고는 단순합니다. 부팅이 5초를 넘는 앱에는 startupProbe를 답니다. livenessProbe는 컨테이너 안에서 닫힌 self-check로만 좁힙니다. /healthz 같은 endpoint가 있다면 그 안에서 외부 호출을 하지 않도록 합니다. readiness가 트래픽 차단을 담당하고, liveness는 데드락이나 메모리 누수처럼 재시작이 필요한 상황만 담당합니다. 두 probe의 역할을 섞지 않는 것이 가장 안전합니다.

다음 글에서는 startupProbe가 1.18에 추가되기 전, 어떤 운영 패턴들이 부팅 시간 문제를 우회하려 했는지 그 역사를 살펴보고, 현재의 startupProbe 설계가 어떤 트레이드오프 위에서 결정되었는지 KEP-950의 제안 과정을 따라가보겠습니다.

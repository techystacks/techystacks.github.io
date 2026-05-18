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

지난 글에서 readiness probe 실패가 EndpointSlice를 어떻게 갱신하는지 다뤘습니다. 이번에는 짝을 이루는 또 다른 probe인 livenessProbe를 봅니다. liveness는 readiness와 비슷하게 생긴 YAML 블록이지만 동작은 정반대입니다. readiness가 트래픽을 잠시 끊는 신호라면, liveness는 컨테이너 자체를 죽이는 트리거입니다. EKS Auto Mode `funny-pop-mountain`에서 안티패턴 두 개와 그 대안을 측정한 결과를 정리합니다.

## liveness probe는 헬스 체크가 아닙니다

많은 입문서가 liveness probe를 "헬스 체크"라고 부르지만 이 표현은 오해를 부릅니다. liveness probe가 실패했을 때 일어나는 일은 모니터링도, 트래픽 차단도, 알림 전송도 아닙니다. `kubelet`이 해당 컨테이너에 SIGTERM을 보내고, terminationGracePeriodSeconds가 지난 뒤에도 살아 있으면 SIGKILL로 죽이고 다시 띄웁니다. 재시작 트리거 그 자체가 liveness입니다.

이 정의를 받아들이면 어떤 상황에 liveness가 도움이 되지 않거나 해로운지 추려낼 수 있습니다. 부팅 시간이 길어 readiness가 되기 전에 liveness가 실패하는 구성은 앱을 영원히 죽임당하게 만듭니다. 외부 의존성을 liveness가 검증하면 의존성 한 개의 일시 장애가 모든 replica의 동시 재시작으로 번집니다. 두 안티패턴이 실제 클러스터에서 어떤 형태로 나타날까요?

같은 namespace 안에 안티패턴 둘과 올바른 분리 하나, 세 Deployment를 두고 측정합니다. 모든 컨테이너는 agnhost 이미지로 부팅 시간 8초를 시뮬레이션합니다.

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

흥미로운 부분은 재시작 사이의 간격입니다. 처음 3번은 즉시 재시작됐습니다. 4번째에 24초의 대기가 들어갔고, 5번째에 57초가 들어갔습니다. CrashLoopBackOff의 지수 backoff가 4번째 재시작부터 동작하는 모양인데, 이는 Kubernetes 1.32에 도입된 새 backoff 정책의 거동입니다. 이전 정책은 10초로 시작해 매번 두 배로 늘려 최대 5분에서 멈췄는데, 새 정책은 1초로 시작하고 컨테이너가 일정 시간 살아 있으면 backoff 카운터가 초기화됩니다. 처음 3번이 즉시 재시작된 이유가 여기에 있습니다. 각 컨테이너가 12초씩 살아 있었기 때문에 매번 카운터가 리셋됐습니다.

이 시나리오의 본질은 부팅 단계까지 liveness가 적용되어 있다는 점입니다. liveness는 "이 컨테이너가 망가졌으니 재시작이 필요하다"는 신호인데, 부팅이 끝나지 않은 상태는 망가진 게 아닙니다. 이 구분을 만들기 위해 1.18에서 startupProbe가 GA되었습니다. 시나리오 C에서 같은 앱을 startupProbe로 보호한 결과를 비교합니다.

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

dep를 다운시키고 9초 만에 cascading의 모든 replica가 첫 번째로 죽었고, 22초 시점에 두 번째로 죽었습니다. 세 replica의 재시작 시각이 1초 안에 모여 있습니다. dep를 복구한 직후에는 진행 중이던 재시작 사이클의 잔여로 또 한 번씩 재시작이 발생했습니다. 31초의 외부 의존성 장애가 9건의 동시 재시작으로 번진 셈입니다. 여러분의 클러스터에서 외부 의존성을 액티브로 검증하는 liveness가 있다면, 의존성 측의 30초짜리 작은 사고 하나가 클라이언트 쪽에서 어떻게 증폭되는지 이 숫자가 그대로 답이 됩니다.

여기서 빠져나가는 길은 한 가지뿐입니다. liveness probe는 컨테이너 안에서 닫혀야 합니다. 외부 호출이 들어가는 endpoint를 liveness로 쓰지 않고, 외부 의존성의 상태는 readiness로 표현하거나 별도의 alerting으로 모니터링합니다. readiness 실패는 트래픽만 잠시 끊고 컨테이너를 죽이지 않으므로 위와 같은 cascading 패턴을 만들지 않습니다.

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

startup probe가 8번 실패했지만(부팅 첫 8초간의 자연스러운 실패) failureThreshold가 30이라서 재시작은 발생하지 않았습니다. t+10s 시점에 startup이 통과해 `started=true, ready=true`가 됐고, 이후 60초 동안 재시작 0회로 안정 상태를 유지했습니다. 시나리오 A와 똑같은 부팅 시간, 똑같은 ready 파일 메커니즘이지만 결과는 정반대입니다.

차이는 startupProbe와 livenessProbe의 책임을 분리한 것 한 가지입니다. startupProbe는 "부팅이 충분히 길어질 수 있다"는 사실을 표현하고 충분한 failureThreshold로 부팅을 보호합니다. livenessProbe는 부팅 이후의 정상 동작 단계에서만 실행되며, 외부 의존성이 아닌 컨테이너 자체 상태(`/healthz` 같은 self-check)만 검증합니다.

## Kubernetes는 이 문제를 어떻게 다듬어왔는가

probe 시스템 자체는 Kubernetes 초기부터 있었지만 livenessProbe와 readinessProbe 둘만으로는 위 안티패턴을 표현할 수 없었습니다. 부팅이 긴 앱에서는 운영자가 `initialDelaySeconds`를 충분히 크게 잡거나, 아니면 부팅을 짧게 만들도록 앱을 고치거나 둘 중 하나를 선택해야 했습니다. `initialDelaySeconds`를 키우면 정상 동작 중에 발생한 실제 행 상태도 그만큼 늦게 감지됩니다.

1.16에서 startupProbe가 알파로 들어왔고 1.18에서 GA가 되면서 이 트레이드오프가 풀렸습니다. 부팅 단계와 정상 동작 단계의 probe를 따로 표현할 수 있게 된 것입니다. startup이 통과한 뒤에야 liveness가 동작하므로 부팅에 너그럽고 정상 동작에 엄격한 정책을 안전하게 만들 수 있게 됐습니다.

CrashLoopBackOff의 backoff 정책도 1.32에서 다시 손봤습니다. 이전에는 첫 실패에서 10초를 기다리고 매 실패마다 두 배로 늘려 최대 5분에서 멈췄는데, 새 정책은 1초로 시작하고 컨테이너가 일정 시간 정상 동작하면 backoff를 초기화합니다. 시나리오 A에서 처음 3번이 즉시 재시작되고 4번째부터 24초, 57초로 지수적으로 커지는 거동이 이 새 정책 그대로의 모습입니다.

## EKS Auto Mode 맥락 — feature gate는 보이지 않지만 거동은 관찰됩니다

EKS Auto Mode의 `funny-pop-mountain`은 1.35 기준이고 컨테이너 런타임은 containerd 2.1.6입니다. CrashLoopBackOff의 새 정책을 제어하는 feature gate는 Auto Mode에서 직접 확인하기 어렵습니다. 노드 SSH 접근이 막혀 있고 kubelet 설정 파일에 직접 닿을 방법이 없기 때문입니다. 그래도 시나리오 A에서 본 거동(처음 3번 즉시 재시작 → 4번째 24초 → 5번째 57초)이 새 정책의 행태와 일치하므로, 이 클러스터의 kubelet에서 새 정책이 활성화되어 있다고 추론할 수 있습니다. 이전 정책이라면 첫 재시작부터 10초 대기가 들어왔어야 합니다.

실험이 끝나면 `kubectl delete namespace lab-liveness-misuse` 한 줄로 정리합니다.

## 맺음말

livenessProbe는 헬스 체크가 아니라 재시작 트리거입니다. 이 정의 한 줄만 잡고 있어도 부팅이 짧지 않은 앱에 짧은 `initialDelaySeconds`를 거는 실수와, 외부 의존성을 liveness로 검증해 cascading failure를 만드는 실수를 피할 수 있습니다. 실측해보면 수치가 분명하게 드러납니다. 5분 안에 5번 재시작, 31초의 의존성 장애가 9건의 동시 재시작으로 번지는 식입니다.

운영에서 손볼 자리는 두 군데입니다. 부팅이 빠르지 않은 앱에는 startupProbe를 답니다. 그리고 livenessProbe의 검증 대상을 컨테이너 내부로 좁힙니다. `/healthz` 같은 핸들러가 있다면 그 안에서 외부 호출을 하지 않도록 막아두면 좋습니다. readiness가 트래픽을 끊는 일을, liveness가 진짜로 행이 걸리거나 메모리 누수가 누적된 컨테이너를 재시작하는 일을 각자 맡고 두 probe의 책임이 섞이지 않게 하는 것이 핵심입니다.

다음 글에서는 startupProbe가 1.18에 GA되기 전까지 어떤 우회들이 시도됐는지, 그리고 KEP-950이 어떤 트레이드오프 위에서 지금의 설계로 결론을 냈는지 따라가보려 합니다.

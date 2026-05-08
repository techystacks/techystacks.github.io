---
title: "Readiness probe가 실패하면 정확히 무엇이 일어나는가 — EndpointSlice 갱신까지의 네 단계"
date: 2026-05-04
draft: false
tags: ["readiness", "endpointslice", "kube-proxy", "publishNotReadyAddresses", "probe"]
categories: ["워크로드 수명주기"]
k8s_version: "1.35"
cluster: "funny-pop-mountain (EKS Auto Mode, us-east-1)"
difficulty: ["중급"]
---

이 글은 [kubernetes/enhancements KEP-752 (EndpointSlice)](https://github.com/kubernetes/enhancements/tree/master/keps/sig-network/0752-endpointslices), [KEP-1672 (Tracking Terminating Endpoints)](https://github.com/kubernetes/enhancements/tree/master/keps/sig-network/1672-tracking-terminating-endpoints), 그리고 [공식 Service 문서](https://kubernetes.io/docs/concepts/services-networking/service/)를 참고하여 정리하였습니다.

Kubernetes에서 `readinessProbe`가 실패했을 때 "그 Pod이 Service endpoint에서 빠진다"는 설명은 대부분의 입문서가 공유하는 요약입니다. 그런데 실제 운영에서는 이 문장의 행간이 크게 벌어집니다. 얼마나 빨리 빠지는가, 누가 빼는가, `publishNotReadyAddresses`나 `terminating` 필드가 이 그림에 어떻게 끼어드는가. 이번 글에서는 EKS Auto Mode의 `funny-pop-mountain` 클러스터에서 네 가지 실험을 돌려 probe 실패부터 EndpointSlice 갱신까지의 구간을 밀리초 단위로 측정한 결과를 소개하고자 합니다.

## readinessProbe가 실패하면 왜 endpoint에서 빠지는가

표준 설명은 이렇게 시작합니다. Service는 label selector로 Pod 집합을 가리키고, kube-proxy는 그 집합을 dataplane 규칙(iptables/nftables/IPVS)에 반영합니다. 그런데 kube-proxy는 Service를 직접 보는 게 아니라 EndpointSlice라는 중간 리소스를 봅니다. EndpointSlice의 각 endpoint에는 세 개의 조건 필드가 붙어 있습니다.

- `conditions.ready` — 이 주소를 클라이언트 트래픽의 기본 후보로 둘지 여부
- `conditions.serving` — Pod 자체가 readiness probe를 통과한 상태인지 여부
- `conditions.terminating` — Pod이 삭제 과정에 들어갔는지 여부

세 필드가 따로 있는 이유는 2021년 이후 Kubernetes가 "종료 중이지만 아직 응답 가능한 Pod"을 표현하고 싶었기 때문입니다. Pod이 SIGTERM을 받고 `preStop`을 수행하는 동안에도 여전히 TCP 연결을 받을 수 있는데, 과거의 `Endpoints` 리소스에는 이걸 표현할 자리가 없었습니다. `serving`과 `terminating`이 분리되면서 kube-proxy가 "ready는 아니지만 serving=true이고 terminating=true인 endpoint"로 트래픽을 우아하게 흘리다 접을 수 있게 되었습니다.

readinessProbe 실패는 이 세 필드 중 `ready`와 `serving`을 함께 `False`로 떨어뜨리는 이벤트입니다. 그런데 그 전환이 실제로 몇 단계를 거치는지, 각 단계가 얼마나 걸리는지는 다른 얘기입니다. probe 실패가 발생한 순간부터 kube-proxy가 보는 EndpointSlice에 반영되기까지 정확히 몇 초가 걸릴까요? 그리고 이 지연의 대부분은 어디에서 오는 걸까요?

## 실험 설계

실습은 전용 네임스페이스 `lab-readiness-endpointslice`에서 replicas 3개의 Deployment로 수행했습니다. Pod 이미지는 `registry.k8s.io/e2e-test-images/agnhost:2.53`를 쓰고, readinessProbe를 다음처럼 구성했습니다.

```yaml
readinessProbe:
  exec:
    command: ["cat", "/tmp/ready"]
  initialDelaySeconds: 0
  periodSeconds: 1
  timeoutSeconds: 1
  successThreshold: 1
  failureThreshold: 3
lifecycle:
  postStart:
    exec:
      command: ["touch", "/tmp/ready"]
```

probe는 `/tmp/ready` 파일의 존재만 확인합니다. `kubectl exec <pod> -- rm /tmp/ready`로 파일을 지우는 순간 다음 probe 주기부터 실패가 시작됩니다. `periodSeconds: 1, failureThreshold: 3`이므로 이론적으로는 3초 뒤 Pod이 NotReady로 확정됩니다.

측정 도구는 두 개의 watcher입니다. 하나는 `kubectl get pod <name> -o json --watch`로 `.status.conditions[Ready]`만 뽑아내고, 다른 하나는 `kubectl get endpointslice -l kubernetes.io/service-name=backend -o json --watch`로 타깃 IP의 conditions만 뽑아냅니다. 둘 다 `jq --unbuffered`로 한 줄씩 epoch 밀리초와 함께 파일로 흘리고, 트리거 시각 T0 기준 상대 ms로 정렬해서 출력합니다.

## 트리거부터 EndpointSlice 반영까지 — 실측 4.5초

첫 실험은 `publishNotReadyAddresses: false`(기본값) 상태에서 Pod 조건과 EndpointSlice 조건 전환 타이밍을 나란히 잰 결과입니다. 트리거 T0는 `rm /tmp/ready`를 호출하기 직전의 호스트 epoch입니다.

```
target_pod=backend-765597c89d-9xzw7
target_ip=10.0.217.232
T0_epoch_ms=1777903456577  (trigger: rm /tmp/ready)
exec returned at t+2052ms

=== Pod Ready condition transitions (relative ms from T0) ===
t  -2807ms  status=True  reason=None              lastTransitionTime=2026-05-04T14:04:02Z
t  +4471ms  status=False reason=ContainersNotReady lastTransitionTime=2026-05-04T14:04:20Z

=== EndpointSlice target-IP transitions (relative ms from T0) ===
t  -2813ms  ready=True  serving=True  terminating=False
t  +4571ms  ready=False serving=False terminating=False
```

T0 이후 4,471ms 뒤에 Pod의 `Ready` condition이 `False`로 떨어졌고, 그로부터 정확히 100ms 뒤에 EndpointSlice의 해당 endpoint도 `ready=False, serving=False`로 전환되었습니다. 이 100ms가 **endpoints controller(컨트롤러 매니저 안)와 apiserver 간 왕복 + EndpointSlice PATCH 쓰기 + watcher 수신**의 총합입니다. 예상보다 짧습니다.

반대로 병목은 앞쪽 4.4초에 있습니다. 이 구간은 다음으로 쪼갤 수 있습니다.

- `kubectl exec` 세션 수립과 반환 오버헤드 2,052ms. 실제 `rm` syscall은 이보다 일찍 완료됩니다. `exec` 반환 시각이 아니라 kubelet probe가 첫 실패를 "보는" 시점을 기준으로 해야 더 정확합니다.
- kubelet이 `periodSeconds=1`로 1초마다 probe를 실행하지만 실행 시점이 T0와 동기화되어 있지 않습니다. 최대 1초의 오프셋이 랜덤하게 섞입니다.
- `failureThreshold=3`이므로 세 번 연속 실패가 필요합니다. 연속 실패가 확정되는 순간 kubelet은 Pod status에 `ContainersReady=False`를 쓰고, 이게 `Ready=False`로 전파됩니다.

즉 4.4초 중 절반 정도는 probe 스케줄 구조에서 오는 최소 비용이고, 나머지는 `kubectl exec`의 세션 비용입니다. 다시 한 번 중요한 지점은 **EndpointSlice 자체의 전파 지연은 100ms 수준**이라는 점입니다.

## publishNotReadyAddresses는 ready만 바꾸고 serving은 그대로 둡니다

같은 실험을 Service에 `publishNotReadyAddresses: true`를 추가한 뒤 다시 돌렸습니다.

```
service patched: publishNotReadyAddresses=true
target_pod=backend-765597c89d-njxl5
target_ip=10.0.217.229
T0_epoch_ms=1777903555525

=== EndpointSlice transitions (relative ms from T0) ===
t   -879ms  total_endpoints=3  target: ready=True serving=True  terminating=False
t  +4708ms  total_endpoints=3  target: ready=True serving=False terminating=False
```

차이가 선명합니다. probe 실패 이후에도 `ready=True`가 유지되고, 대신 `serving=False`만 내려갑니다. 주소는 EndpointSlice에 그대로 남아 있고, 총 endpoint 개수도 3 그대로입니다. 반면 실험 2에서는 `ready, serving` 두 필드가 함께 `False`로 전환되었습니다.

이 차이의 의미는 두 축으로 읽힙니다. 첫째, **`ready` 필드는 "Service 정책이 이 주소를 후보로 보여줄지"를 의미**합니다. `publishNotReadyAddresses: true`는 "probe가 실패하더라도 여전히 ready로 보여달라"는 정책 선언이고, endpoints controller가 이 플래그를 읽어 `ready` 필드를 덮어씁니다. 둘째, **`serving` 필드는 Pod의 실제 상태를 그대로 반영**합니다. 어떤 Service 정책으로도 이 값은 바꿀 수 없고 probe 결과가 바뀌지 않으면 절대 `True`가 되지 않습니다.

`publishNotReadyAddresses`를 쓸 만한 장면은 Headless Service 뒤에서 StatefulSet 멤버끼리 서로를 발견해야 할 때입니다. probe가 아직 통과하지 못한 부트스트랩 단계에서도 같은 StatefulSet의 다른 멤버가 `pod-1.myservice.ns.svc.cluster.local`을 resolve할 수 있어야 클러스터가 구성됩니다. 반면 클라이언트 트래픽을 받는 일반 Service에서는 이 옵션을 켜면 `ready` 시맨틱이 망가지므로 쓰지 않습니다.

## 복구와 종료 — 2.4초 뒤 돌아오고, 삭제는 1.4초 안에 terminating=True로 찍힙니다

네 번째 실험은 실패한 Pod을 `touch /tmp/ready`로 복구하는 시점, 그리고 같은 Pod을 `kubectl delete pod`로 삭제하는 시점을 한 줄기로 이어 측정했습니다.

```
=== EndpointSlice target-IP transitions ===
(relative ms; FAIL=probe failure, REC=recovery trigger, DEL=delete trigger)
FAIL   -688ms  ready=True  serving=True  terminating=False
FAIL  +4443ms  ready=False serving=False terminating=False
REC   +2370ms  ready=True  serving=True  terminating=False
DEL   +1390ms  ready=False serving=True  terminating=True
DEL   +1595ms  (removed)
```

복구는 REC+2,370ms에 완료되었습니다. 실패 경로(4.4초)보다 빠릅니다. 이유는 `successThreshold: 1`입니다. 복구 경로에서는 딱 한 번의 성공 probe면 Pod이 `Ready=True`로 돌아옵니다. 이 2.4초 중 여전히 절반 정도는 `kubectl exec`의 세션 비용이고, 나머지가 probe 주기의 잔여 offset과 status patch 구간입니다.

삭제 경로에서 주목할 줄은 `DEL+1,390ms`입니다. `kubectl delete pod`가 떨어진 지 1.4초 만에 EndpointSlice가 `ready=False, serving=True, terminating=True`로 전환되었습니다. 여기서 `serving=True`가 남아 있는 이유가 바로 KEP-1672가 만들려던 신호입니다. probe가 실제로 통과하고 있는 Pod이 종료 과정에 들어간 상태를 구별해주는 필드 조합입니다. `externalTrafficPolicy: Local`이거나 `spec.trafficDistribution`을 쓰는 Service, 또는 Gateway API 구현체들이 이 조합을 근거로 "graceful connection draining" 중인 endpoint에 기존 연결을 계속 흘립니다.

그리고 DEL+1,595ms에는 endpoint가 EndpointSlice에서 통째로 사라졌습니다. Pod이 `terminationGracePeriodSeconds: 30`을 갖고 있지만, 이 Pod은 `preStop` 훅이 없고 프로세스가 즉시 SIGTERM에 반응해 종료되었기 때문에 실제 종료는 gracePeriod보다 훨씬 빨리 끝났습니다. 종료 완료 직후 endpoints controller가 이 endpoint를 slice에서 빼버린 결과가 `(removed)` 줄입니다.

## Kubernetes는 이 문제를 어떻게 다듬어왔는가

EndpointSlice 자체가 2019년(Kubernetes 1.16)에 알파로 등장해 1.21에서 GA된 리소스입니다. 그 이전 `Endpoints` 리소스는 Service 하나당 단 하나의 객체였고, 수천 개의 Pod이 붙은 대규모 Service에서는 endpoint 하나가 바뀔 때마다 수 MB짜리 객체가 apiserver로 오가는 병목이 있었습니다. EndpointSlice는 이걸 기본 100개씩 쪼갠 복수 슬라이스로 바꿔 watch 이벤트 크기를 끌어내렸습니다.

그 위에 2021~2022년 사이에 `serving`과 `terminating` 필드가 더해졌습니다. 초기 EndpointSlice에도 `ready` 필드는 있었지만, Pod 종료 중에는 `ready=false`가 되면서 "아직 요청을 처리할 수 있다"는 신호가 사라졌습니다. 그래서 SIGTERM을 받은 Pod이 `preStop` 훅을 실행하는 짧은 시간 동안 기존 연결을 받지 못하는 경주 조건이 드러났습니다. `serving`, `terminating` 두 필드가 1.26에서 기본 동작으로 자리 잡으면서 이 경주 조건이 많은 부분 해소되었고, kube-proxy는 "정상 endpoint가 0개이면 serving=true인 terminating endpoint로 폴백한다"는 규칙을 갖게 되었습니다.

`publishNotReadyAddresses` 옵션은 더 오래된 기능으로 Kubernetes 1.11부터 Service spec에 정식으로 들어있습니다. 역사적으로는 Headless Service 뒤 StatefulSet에서 peer discovery 용도로 주로 쓰였고, 2020년 이후 EndpointSlice가 `ready`/`serving` 두 축을 분리하면서 이 옵션의 의미가 깔끔해졌습니다. "ready 필드만 거짓말하고, serving은 진실을 유지한다"는 지금 모델이 그 결과입니다.

## EKS Auto Mode의 관찰 제약과 확인 가능한 것

funny-pop-mountain처럼 EKS Auto Mode에서 돌아가는 클러스터는 kube-proxy가 관리형으로 숨겨져 있습니다. 즉 `kubectl -n kube-system get pods`로 kube-proxy 파드를 볼 수 없고, 노드에 SSH 세션으로 들어가 iptables/nftables 룰을 직접 dump할 수도 없습니다. 그 대신 이번 실험이 보여준 층위 — EndpointSlice 자체의 필드 전환 — 은 control plane 리소스이므로 완전히 관찰 가능합니다. kube-proxy는 이 EndpointSlice를 watch 해서 자신의 dataplane에 반영하는 consumer이므로, Service 쪽 semantics의 시각에서는 EndpointSlice만 봐도 모델이 닫힙니다.

한편 funny-pop-mountain에는 `TargetGroupBinding`이라는 별도 리소스가 `eks.amazonaws.com/v1beta1`로 존재하는데, AWS Load Balancer Controller가 EndpointSlice를 watch 해 NLB/ALB Target Group에 IP를 직접 등록하는 경로입니다. 이쪽에서는 `serving=true, terminating=true` 조합이 Target의 health를 어떻게 반영하는지가 별도의 측정 대상입니다. 이 부분은 다음 글에서 다룹니다.

실험이 끝나면 `kubectl delete namespace lab-readiness-endpointslice` 한 줄로 정리합니다.

## 맺음말

readiness probe 실패가 Service endpoint에 반영되는 전체 시간은 이 글의 설정(`periodSeconds: 1, failureThreshold: 3`)에서 약 4.5초였고, 그중 EndpointSlice 갱신 자체의 지연은 100ms에 불과했습니다. 병목은 probe 스케줄과 3회 연속 실패 확정 구간에 있습니다. 복구는 한 번의 성공만 요구하므로 2.4초로 더 빠르고, `kubectl delete pod`는 1.4초 만에 endpoint 조건을 `terminating=True`로 뒤집습니다.

여러분이 graceful shutdown 품질을 높이고 싶다면 readiness probe 실패 감지 시간을 줄이는 것보다 `preStop` 훅과 `terminationGracePeriodSeconds`를 이용해 **serving=true, terminating=true 구간을 확보**하는 쪽이 실제 효과가 큽니다. 반대로 Headless Service에서 peer discovery가 필요한 상황이 아니라면 `publishNotReadyAddresses`를 켜지 않아야 합니다. 이 옵션은 `ready` 시맨틱을 의도적으로 거짓으로 만드는 장치이므로 일반적인 클라이언트 트래픽을 받는 Service에서는 혼동을 부릅니다.

이 결과를 한 층 더 들어가려면 kube-proxy 내부에서 EndpointSlice 이벤트가 iptables 체인으로 반영되기까지의 지연을 봐야 합니다. 다음 글에서는 같은 시나리오를 EKS Auto Mode가 아닌 일반 EKS 노드에서 재현해 kube-proxy sync 주기와 dataplane 반영 시간까지 포함한 전체 지연 예산을 측정해보겠습니다.

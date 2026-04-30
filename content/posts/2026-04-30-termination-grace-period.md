---
title: "EKS Auto Mode에서 Pod 종료 타임라인과 kube-proxy 룰 전파 지연 측정"
date: 2026-04-30
draft: false
tags: ["terminationGracePeriodSeconds", "SIGTERM", "SIGKILL", "preStop", "EndpointSlice", "kube-proxy"]
categories: ["workload-lifecycle", "networking"]
k8s_version: "1.35"
cluster: "funny-pop-mountain (EKS Auto Mode, us-east-1)"
difficulty: "중급"
---

이 글은 [kubernetes/kubernetes#85643](https://github.com/kubernetes/kubernetes/issues/85643), [#89263](https://github.com/kubernetes/kubernetes/issues/89263), [#117373](https://github.com/kubernetes/kubernetes/issues/117373)의 내용을 바탕으로 정리하였습니다.

Kubernetes를 오래 운영하다 보면, Pod을 삭제했을 때 해당 Service로 가는 요청이 순간적으로 실패하는 현상을 한 번쯤은 겪게 됩니다. 애플리케이션이 SIGTERM을 제대로 받고 정상적으로 종료했음에도 불구하고, 클라이언트 측에서는 `Connection refused`가 수 초간 이어지는 식입니다. 이번 글에서는 `terminationGracePeriodSeconds`와 `preStop` 훅이 실제로 어떻게 동작하는지, 그리고 Pod이 종료되는 동안 kube-proxy의 iptables 룰이 언제 갱신되는지를 실제 EKS Auto Mode 클러스터에서 측정한 결과를 바탕으로 소개하고자 합니다.

## Pod을 지우면 무엇이 동시에 진행되는가

`kubectl delete pod`을 실행한 순간, 클러스터 내부에서는 여러 컴포넌트가 동시에 자기 할 일을 시작합니다. kubelet은 해당 Pod의 컨테이너에 SIGTERM을 보내고, 애플리케이션은 SIGTERM을 받으면 종료 절차로 들어갑니다. 이와는 별도로 kube-controller-manager 안의 EndpointSlice 컨트롤러는 Pod의 `deletionTimestamp`를 감지해 해당 endpoint를 `terminating` 상태로 마킹하고, 각 노드의 kube-proxy는 이 변경을 watch 스트림으로 받아 자신의 iptables 룰을 다시 써 내려갑니다. 여기서 중요한 것은 이 작업들이 **순서가 보장되지 않고**, 각자 다른 속도로 완료된다는 점입니다.

이 지점이 문제의 핵심입니다. 애플리케이션이 SIGTERM에 즉시 반응하여 깔끔하게 종료되는 경우, Pod은 이미 죽어서 해당 IP로 들어온 패킷을 받을 수 없습니다. 그런데 어떤 노드의 kube-proxy는 아직 iptables 룰을 다 지우지 못했기 때문에, 그 노드에서 출발한 요청은 여전히 죽은 Pod의 IP로 향하게 됩니다. 커널은 상대방이 없다고 판단하여 RST로 응답하고, 이것이 클라이언트 입장에서는 `Connection refused`입니다. Kubernetes 공식 문서의 [Pod Lifecycle](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#pod-termination) 섹션에서도 "endpoint 제거와 Pod 종료는 병렬로 진행된다"고 명시하고 있지만, **얼마나 벌어지는지**까지는 다루지 않습니다. 그 값을 실측으로 확인해 보는 것이 이번 글의 목적입니다.

## 세 개의 Pod로 각 구간의 타이밍을 분리해봅니다

각 구간에 얼마만큼의 시간이 걸리는지 확인하려면, 구간을 고립시킨 실험이 필요합니다. 아래 세 개의 Pod을 준비했습니다. 첫 번째 Pod은 아무런 graceful shutdown 대비가 없는 경우를, 두 번째는 `preStop` 훅으로 10초의 유예를 둔 경우를, 세 번째는 `terminationGracePeriodSeconds`를 짧게 설정한 상태에서 애플리케이션이 의도적으로 종료를 지연시키는 경우를 각각 재현합니다.

| Pod | `terminationGracePeriodSeconds` | `preStop` | 애플리케이션의 SIGTERM 반응 | 실험의 목적 |
|-----|-------------------------------|-----------|--------------------------|------------|
| A. naive | 기본 30초 | 없음 | 즉시 종료 | 아무 대비도 없는 Pod이 만드는 유실 구간 관찰 |
| B. preStop | 기본 30초 | `sleep 10` | 즉시 종료 | preStop이 번 10초가 어디에 쓰이는지 관찰 |
| C. holdout | 5초 | 없음 | 60초 동안 버팀 | SIGKILL이 언제 날아오는지 관찰 |

세 Pod 모두 `registry.k8s.io/e2e-test-images/agnhost:2.53`의 `netexec` 서버를 8080 포트에 띄우고, 앞에 ClusterIP Service를 두었습니다. `netexec`은 Kubernetes e2e 테스트에서 공식적으로 쓰이는 이미지로, 시나리오 C의 경우 `--delay-shutdown=60` 옵션을 주어 SIGTERM 수신 후 60초 동안 종료를 지연시키는 동작을 재현합니다. 옆에 Python으로 작성한 probe Pod을 배치하여 50밀리초 간격으로 HTTP GET을 발행하고, 각 요청의 epoch timestamp와 HTTP 응답 코드, 응답 시간을 기록하도록 했습니다. `kubectl delete pod`을 실행하는 시각도 같은 방식으로 기록하여 probe 로그와 상대 시간으로 정렬할 수 있게 구성했습니다. 한 가지 덧붙여두자면, 이번 글에서 인용하는 수치는 각 시나리오를 한 번씩 실행한 결과이므로 반복 측정의 중앙값이나 분산이 아닌 단일 관찰값이라는 점은 감안하시기 바랍니다. 수치의 자릿수 자체보다는 시나리오 사이의 상대적인 차이와 그 차이를 만드는 메커니즘을 읽는 데 초점을 두었습니다.

## 깔끔하게 종료되는 앱일수록 유실이 발생합니다

가장 먼저 시나리오 A, 그러니까 아무런 대비 없이 Pod을 띄우고 삭제했을 때 어떤 일이 일어나는지 살펴보겠습니다. 앞서 언급드린 것처럼 `agnhost netexec`은 SIGTERM을 받으면 즉시 종료되는 전형적인 graceful shutdown 애플리케이션입니다. 클라우드 네이티브 관점에서는 "잘 만든 앱"이라고 할 수 있겠습니다. 이 Pod을 `kubectl delete`로 삭제하면서 probe 로그를 상대 시간으로 정렬해 보았습니다.

```
 -0.011s | code=200 dur=1.6ms
 +0.041s | code=200 dur=1.5ms
 +0.093s | code=200 dur=1.4ms
   ...
 +1.022s | code=200 dur=1.5ms
 +1.073s | code=200 dur=1.5ms
 +1.125s | code=0   URLError:<Connection refused>
 +1.176s | code=0   URLError:<Connection refused>
 +1.227s | code=0   URLError:<Connection refused>
   ...
 +3.639s | code=0   dur=1011.1ms   URLError:<Connection refused>
 +4.700s | code=0   dur=1069.9ms   URLError:<Connection refused>
 +5.820s | code=0   dur=1079.9ms   URLError:<Connection refused>
```

이 결과에서 가장 주목할 만한 숫자는 **1.125초**입니다. DELETE 명령을 친 지 1.1초가량 지난 시점에 첫 `Connection refused`가 기록되었습니다. netexec은 SIGTERM을 받자마자 프로세스를 내렸고, 그 순간 이 노드의 kube-proxy가 반영한 패킷 전달 경로는 이미 존재하지 않는 Pod IP를 향하고 있었습니다. probe는 Service의 ClusterIP로 계속 패킷을 보냈고, 커널은 해당 endpoint가 도달 불가임을 응답한 셈입니다.

또 하나 눈에 띄는 점은 이 실패가 한 번으로 끝나지 않고 약 6초 동안 이어졌다는 것입니다. 중간에 `dur=2001ms`나 `dur=1011ms` 같이 긴 응답 시간이 섞여 있는데, 이는 SYN 패킷이 답을 받지 못해 TCP 재전송 타임아웃에 걸린 흔적으로 해석할 수 있습니다. 다만 이 6초가 정확히 어떻게 구성되어 있는지는 이 실험만으로는 결론 내릴 수 없습니다. 가능한 기여 요인으로는 kube-proxy의 iptables 룰 sync 주기, conntrack의 established 엔트리 잔존, 그리고 probe 쪽의 TCP 재시도 타임아웃 누적이 모두 얽혀 있으며, 이를 분리하려면 iptables-save의 snapshot, `conntrack -L`의 타임라인 추적, probe 측의 커널 소켓 상태 관찰 같은 추가 실험이 필요합니다. 이번 글에서는 **"Pod이 사라진 뒤에도 클라이언트 입장에서는 수 초간 실패가 관찰될 수 있다"** 는 사실의 재현에 초점을 맞추고, 원인 분해는 후속 글의 과제로 남깁니다.

이 결과가 시사하는 바는 직관과 다소 반대됩니다. SIGTERM을 성실하게 처리하는 애플리케이션일수록 오히려 유실 구간이 명확하게 드러납니다. Pod 쪽이 종료를 빨리 끝낼수록, 뒤따라오는 kube-proxy의 iptables 룰 갱신과의 시간 격차가 그대로 구멍이 되어버리기 때문입니다.

## preStop sleep 10초는 어디에 쓰이는가

그렇다면 이 격차를 어떻게 줄일 수 있을까요? 가장 간단하고 많이 알려진 패턴은 `preStop` 훅에 sleep을 거는 방법입니다. 시나리오 B는 같은 애플리케이션에 아래의 한 조각만 추가한 경우입니다.

```yaml
lifecycle:
  preStop:
    exec:
      command: ["/bin/sh", "-c", "sleep 10"]
```

`preStop` 훅은 kubelet이 SIGTERM을 보내기 **전**에 실행됩니다. 10초 동안 컨테이너는 그대로 살아있어 트래픽을 받을 수 있고, 그 사이 EndpointSlice 갱신과 모든 노드의 kube-proxy 룰 재작성이 대부분 완료됩니다. 같은 방식으로 측정한 결과는 아래와 같습니다.

```
 -0.961s | code=200 dur=1.4ms
   ...(0초부터 11초까지 code=200 유지)...
+10.951s | code=200 dur=1.6ms
+11.003s | code=200 dur=1.7ms
+11.055s | code=200 dur=1.5ms
+11.106s | code=200 dur=1.3ms
+11.158s | code=200 dur=1.6ms
+11.209s | code=200 dur=1.4ms
+11.261s | code=0   URLError:<Connection refused>
+11.312s | code=0   URLError:<Connection refused>
```

첫 실패 시점이 **11.261초**로 밀렸습니다. 시나리오 A의 1.125초와 비교하면 약 10.1초의 차이가 나는데, 이 값은 preStop이 확보한 10초에 그 이후의 추가 지연이 더해진 결과입니다. `kubectl delete pod --wait=true`가 반환한 총 소요 시간은 12.21초였으며, preStop 10초와 SIGTERM 수신 후 종료 시간, kubelet이 Pod status를 업데이트하는 시간이 모두 합쳐진 값입니다.

흥미로운 지점은 preStop이 끝난 직후에도 약 1초간 200 OK 응답이 이어진다는 점입니다. preStop이 실행되는 10초 동안 Pod에는 `deletionTimestamp`가 찍혀 있고 EndpointSlice 역시 `terminating=true`로 업데이트되어 있는 상태이므로, 이론적으로는 그 구간 안에 모든 노드의 kube-proxy가 룰을 갱신하고 신규 트래픽을 차단해야 합니다. 그런데도 1초의 잔존 성공 응답이 관찰된다는 사실은 단순히 "룰 제거가 늦었다"로 설명하기에는 부족합니다. 가능한 후보는 세 가지입니다. 첫째, 이 글의 실험에서는 kube-proxy가 **terminating 상태의 endpoint로도 fallback**하여 트래픽을 전달했을 수 있습니다(뒤에서 다룰 Kubernetes 1.26 이상의 기본 동작입니다). 둘째, probe와 Pod 사이에 이미 established 상태인 conntrack 엔트리가 남아 있어 그 엔트리가 만료되기 전까지 패킷이 흘러갔을 수 있습니다. 셋째, `serving=true` 구간이 아직 유지되고 있어 kube-proxy가 의도적으로 라우팅을 유지했을 수도 있습니다. 셋 중 어느 요인의 기여가 지배적인지는 이 실험 범위에서 단정할 수 없으며, EndpointSlice의 조건 전이 시각을 초단위로 기록하는 별도 실험이 필요합니다.

여기서 두 가지 주의할 점이 있습니다. 첫째, **preStop 실행 시간은 `terminationGracePeriodSeconds` 안에서 소모됩니다**. 즉 기본 30초인 grace period에서 preStop이 10초를 쓰면, SIGTERM 이후 애플리케이션이 정리할 수 있는 시간은 약 20초로 줄어듭니다. 이 동작은 [issue #117373](https://github.com/kubernetes/kubernetes/issues/117373)에서 많은 사용자가 혼동했던 지점이기도 합니다. 만약 preStop이 grace period 전체를 소모해 버리면, kubelet은 한 번에 한해 2초의 추가 유예를 부여하고 그래도 끝나지 않으면 SIGKILL을 보냅니다. 둘째, preStop sleep은 **endpoint가 제거될 시간을 벌어주는 장치**이지, 진행 중인 connection을 drain하는 장치가 아닙니다. sleep이 끝나면 애플리케이션은 SIGTERM을 받고 즉시 종료하므로, 그 시점에 처리 중이던 긴 요청은 그대로 끊어져 버립니다. 만약 여러분의 워크로드가 진정한 무중단을 요구한다면, 애플리케이션이 SIGTERM 수신 후 신규 요청을 거부하면서 진행 중인 요청만 완료하고 종료하는 로직을 직접 구현해야 합니다.

## SIGKILL은 정확히 언제 날아오는가

이번에는 반대 방향의 실험을 해보겠습니다. 시나리오 C는 `terminationGracePeriodSeconds`를 5초로 짧게 설정하고, 애플리케이션이 SIGTERM을 받고도 60초 동안 버티도록 구성한 경우입니다. kubelet은 5초가 지나면 SIGKILL을 보내야 하며, 애플리케이션의 의지와는 무관하게 Pod은 5초 시점에 강제로 종료되어야 합니다.

```yaml
spec:
  terminationGracePeriodSeconds: 5
  containers:
  - name: app
    image: registry.k8s.io/e2e-test-images/agnhost:2.53
    args: ["netexec", "--http-port=8080", "--delay-shutdown=60"]
```

`kubectl delete pod --wait=true`가 반환한 시간은 **6.97초**였습니다. 내역을 분해하자면 DELETE 명령부터 kubelet이 SIGTERM을 전달하기까지 수백 밀리초, SIGTERM 이후 유예가 정확히 5초, 그리고 SIGKILL 발동 이후 프로세스가 정리되고 kubelet이 Pod status를 업데이트하기까지 약 1.5초가 소요되었습니다. `terminationGracePeriodSeconds: 5`의 의미가 글자 그대로 구현되어 있음을 확인할 수 있었습니다.

한 가지 재미있는 점은 이 Pod의 Events를 조회해 보면 `Killing: Stopping container app` 이벤트가 SIGTERM 발송 시점에 한 번 찍히고 끝난다는 것입니다. **5초 뒤 실제로 SIGKILL이 발동되는 순간에 해당하는 별도의 이벤트는 남지 않습니다**. 만약 SIGKILL 발동 시점을 관찰하고 싶다면 노드의 kubelet 로그를 직접 봐야 합니다. EKS Auto Mode에서는 노드에 SSH로 직접 접근이 제한되므로, CloudWatch Logs에 수집된 kubelet 로그를 조회하는 방식으로 확인할 수 있습니다.

## Kubernetes는 이 문제를 어떻게 줄여왔는가

Kubernetes 커뮤니티도 오래 전부터 이 타이밍 문제를 인지하고 있었습니다. 2019년에 제기된 [issue #85643](https://github.com/kubernetes/kubernetes/issues/85643)에서는 `externalTrafficPolicy: Local` 환경의 rolling update 중에 일정 비율의 트래픽이 유실된다는 구체적인 사례가 보고되었습니다. 이듬해 SIG Network의 Tim Hockin이 [issue #89263](https://github.com/kubernetes/kubernetes/issues/89263)을 열어 문제를 한 단계 일반화했는데, 당시 그가 정리한 표현이 이 현상을 가장 잘 요약합니다. 로드밸런서의 프로그래밍과 Pod의 grace period가 동시에 진행되기 때문에 terminating 중인 Pod에 여전히 패킷이 전달되며, 많은 운영자가 이 격차를 메우기 위해 `preStop`에 `sleep 60`을 거는 "nasty hack"을 쓰고 있다는 것입니다. 이 두 이슈의 논의가 Kubernetes 1.22에서 alpha로 도입되어 1.26에서 기본 동작으로 자리 잡은 두 가지 개선 사항으로 이어졌습니다.

첫 번째는 **EndpointSlice의 조건 세분화**입니다. 과거 EndpointSlice의 각 endpoint에는 `ready`라는 bool 값 하나만 존재했습니다. Pod이 terminating 상태에 들어가면 `ready=false`가 되어버리고, 이것이 kube-proxy에게 "이 endpoint로는 트래픽을 보내지 말 것"이라는 신호였습니다. 문제는 terminating 중이지만 readiness probe는 여전히 통과하는 Pod, 즉 "지금 처리 중인 연결은 마무리할 수 있는" Pod과 "완전히 죽은" Pod을 구분할 방법이 없었다는 점입니다. 개선 이후 endpoint는 `ready`, `serving`, `terminating`이라는 세 가지 조건을 가지게 되었습니다. `ready`는 kube-proxy의 라우팅 결정용으로 유지되지만 terminating이 시작되면 `false`로 전환됩니다. `serving`은 readiness probe가 통과하는 동안 `true`를 유지하며, 외부 로드밸런서가 connection draining을 판단할 때 이 값을 사용합니다. `terminating`은 Pod에 `deletionTimestamp`가 찍히면 `true`가 됩니다. 실제로 이번 실험 도중 EndpointSlice를 관찰하면 아래와 같은 전이를 확인할 수 있습니다.

```
{ "addresses": ["10.0.228.1"],
  "conditions": { "ready": true,  "serving": true,  "terminating": false } }

    ↓ (Pod에 deletionTimestamp가 찍힌 직후)

{ "addresses": ["10.0.228.1"],
  "conditions": { "ready": false, "serving": true,  "terminating": true  } }

    ↓ (애플리케이션 종료 확인 후)

(endpoint가 목록에서 제거됨)
```

두 번째는 **kube-proxy의 terminating endpoint fallback** 동작입니다. API에 정보가 추가되었으니 이를 활용할 수 있게 된 것인데요, 규칙은 한 문장으로 요약할 수 있습니다. Service의 traffic policy 범위에서 ready하고 not-terminating한 endpoint가 존재하지 않지만, terminating이면서 serving=true인 endpoint가 있다면, kube-proxy는 그 endpoint로 트래픽을 전달합니다. 예를 들어 `externalTrafficPolicy: Local`로 노드당 Pod 하나씩 배치한 상태에서 rolling update가 진행될 때, 해당 Pod이 terminating에 들어간 찰나 개선 이전의 kube-proxy는 "이 노드에는 ready endpoint가 없으니 트래픽을 드롭"이라고 판단했습니다. 이제는 "terminating이지만 serving=true이니 해당 endpoint로 전달"하는 동작으로 바뀌었습니다.

다만 여기서 한 가지 짚고 가야 할 것이 있습니다. 두 개선이 모두 적용된 Kubernetes 1.26 이상 버전인데도 시나리오 A에서는 왜 여전히 `Connection refused`가 1.1초 만에 발생했을까요? 이유는 시나리오 A의 netexec이 SIGTERM 수신 후 즉시 종료되어 `serving=true` 구간이 지극히 짧았기 때문입니다. kube-proxy가 "serving 중인 terminating endpoint로 fallback"을 할 대상 자체가 존재하지 않았던 셈입니다. 이 두 가지 개선은 **serving 중인 시간 동안**의 트래픽을 살려주는 기능이지, **이미 죽은 Pod의 IP로 향하는 트래픽**까지 막아주는 기능은 아닙니다. 시나리오 B의 preStop이 여전히 필요한 이유가 여기에 있습니다. preStop은 애플리케이션을 살아있게 유지함으로써 `serving=true` 구간을 인위적으로 확장해 주는 역할을 하는 셈입니다.

## EKS Auto Mode에서 달라지는 것들

이 글의 실험을 EKS Auto Mode가 아닌 일반 EKS 클러스터에서 돌렸다면 몇 가지가 달랐을 것입니다. 가장 눈에 띄는 차이는 kube-proxy, CoreDNS, VPC CNI 같은 **시스템 컴포넌트가 Pod 형태로 보이지 않는다**는 점입니다. 실제로 `funny-pop-mountain` 클러스터에서 `kube-system` 네임스페이스를 조회하면 metrics-server 외에는 거의 아무것도 보이지 않습니다.

```
$ kubectl -n kube-system get ds
No resources found in kube-system namespace.

$ kubectl -n kube-system get pod -l k8s-app=kube-proxy
No resources found in kube-system namespace.

$ kubectl -n kube-system get pod -l k8s-app=kube-dns
No resources found in kube-system namespace.
```

일반 EKS에서는 kube-proxy가 DaemonSet으로 각 노드에 Pod를 띄우고, 운영자는 `kubectl -n kube-system logs ds/kube-proxy`로 iptables 룰 갱신 로그를 직접 확인할 수 있습니다. Auto Mode에서는 이 경로가 사라졌습니다. 대신 노드에 대응하는 `CNINode` CRD가 생성되어 있습니다.

```
$ kubectl get cninodes
NAME                  AGE
i-031d8a8a7ebf0188e   3h
i-0661ea5abc2bacc38   11d
i-0ad97b85cdce72bf1   11d
```

이 구조의 실무적 함의는 명확합니다. **Pod 종료 타이밍을 디버깅할 때 쓰던 관습적 도구들이 Auto Mode에서는 일부 봉인됩니다.** 노드 위에서 `iptables -L -t nat | grep <service-cluster-ip>`를 실행해 해당 룰의 잔존 여부를 보는 것은 SSH가 막혀 있으므로 불가능하고, kube-proxy Pod이 없으니 `kubectl logs`로 sync 주기나 재작성 로그를 확인할 수도 없습니다. 대신 관찰 가능한 경로는 두 가지로 좁혀집니다. 첫째, `kubectl get endpointslice -o yaml`로 **endpoint 조건의 전이만 추적**하여 API 레벨에서 `terminating=true`, `ready=false`가 언제 찍혔는지 확인합니다. 둘째, CloudWatch Logs에 수집되는 **kubelet 로그**를 통해 SIGTERM 발송과 SIGKILL 시점을 간접적으로 유추합니다. 이번 글이 `iptables` 대신 "probe의 관찰"에 의존한 것도 Auto Mode의 이 제약 때문입니다.

한편 EKS Auto Mode 클러스터에는 Pod의 `terminationGracePeriodSeconds`와 이름이 유사한 또 다른 필드가 함께 존재합니다. 바로 Karpenter가 관리하는 `NodePool`의 `terminationGracePeriod`입니다. `funny-pop-mountain`의 기본 NodePool에는 아래와 같이 설정되어 있습니다.

```yaml
spec:
  template:
    spec:
      terminationGracePeriod: 24h
```

이름은 비슷하지만 의미는 다릅니다. Pod의 `terminationGracePeriodSeconds`가 kubelet이 컨테이너에 SIGTERM을 보낸 뒤 SIGKILL까지 대기하는 시간이라면, NodePool의 `terminationGracePeriod`는 **Karpenter가 노드를 철거하기로 결정한 이후 노드 위의 모든 Pod이 drain될 때까지 대기하는 상한**을 의미합니다. 두 값은 곱하거나 더하는 관계가 아니라 서로 독립적으로 동작합니다. 예를 들어 Karpenter가 `consolidateAfter: 30s`에 따라 어떤 노드를 제거하기로 결정했고, 그 노드에 `terminationGracePeriodSeconds: 600`을 가진 Pod이 하나 남아 있다면, Karpenter는 NodePool의 24시간 한도 안에서 얼마든지 기다립니다. 반대로 어떤 Pod이 1시간짜리 grace를 가지고 있더라도 NodePool의 `terminationGracePeriod`가 10분으로 설정되어 있다면 10분 시점에 노드가 강제로 내려갑니다. Pod의 긴 grace를 Karpenter가 무시한 것처럼 보이는 상황을 종종 보게 되는데, 원인은 대부분 NodePool의 이 필드 쪽에 있습니다.

## 맺음말

이번 글에서는 Kubernetes 1.35 기준의 EKS Auto Mode 클러스터에서 Pod 종료 동안 어떤 일들이 어느 타이밍에 일어나는지를 한 번씩 실측해 보았습니다. 깔끔하게 SIGTERM에 반응하는 "잘 만든" 애플리케이션이라 하더라도 수 초 동안 클라이언트 측 `Connection refused`가 관찰될 수 있다는 점, `preStop` 훅의 sleep이 이 구간을 상당 부분 메워주지만 그 자체가 connection draining을 대신해 주지는 않는다는 점, 그리고 `terminationGracePeriodSeconds`는 말 그대로 SIGKILL까지의 상한으로 정확하게 동작한다는 사실을 확인했습니다. 또한 EKS Auto Mode에서는 kube-proxy와 CoreDNS, VPC CNI가 Pod 형태로 노출되지 않기 때문에 일반적인 `iptables` 검사와 컴포넌트 로그 조회 같은 디버깅 경로가 일부 봉인되어 있으며, 관찰은 API 레벨의 EndpointSlice와 CloudWatch의 kubelet 로그로 국한된다는 점도 함께 짚었습니다.

다만 이번 글의 측정은 각 시나리오를 한 번씩만 돌린 결과이며, 시나리오 A에서 관찰된 약 6초의 refused 구간이 kube-proxy 룰 전파 지연, conntrack 엔트리 잔존, TCP 재전송 타임아웃 중 어느 쪽의 기여가 지배적인지는 분리되지 않았습니다. 시나리오 B에서 preStop 종료 이후 1초 동안 200 응답이 이어진 현상 역시 동일한 맥락에서 원인이 특정되지 않습니다. 이런 한계를 감안하시고, 만약 여러분이 운영 중인 클러스터에서 rolling update 중 일시적인 트래픽 유실을 경험하고 있다면, 가장 먼저 점검해볼 것은 애플리케이션의 SIGTERM 처리 속도와 `preStop` 훅의 존재 여부입니다. Kubernetes 1.26 이상에서는 kube-proxy가 terminating endpoint로 fallback하는 동작이 기본으로 활성화되어 있으므로, 애플리케이션이 SIGTERM 이후에도 잠시 `serving=true` 상태를 유지할 수만 있다면 대부분의 유실이 사라집니다. EKS Auto Mode를 사용 중이라면 Pod의 `terminationGracePeriodSeconds`뿐 아니라 Karpenter NodePool의 `terminationGracePeriod`까지 함께 점검하시는 것을 권장드립니다.

다음 글에서는 이번 글이 남긴 가장 큰 숙제, 즉 **시나리오 A의 6초 refused 구간이 실제로 어떤 요소로 구성되어 있는지**를 원인별로 분리하는 실험을 다룰 예정입니다. `conntrack -L`의 엔트리 지속 시간, kube-proxy의 sync 주기, probe 측의 TCP 재시도 패턴을 각각 독립적으로 변경하며 재현해 보려 합니다. 그와 별도로 readiness probe가 실패할 때 EndpointSlice가 정확히 몇 초 뒤 갱신되는지도 같은 방식으로 측정해볼 만한 주제이므로, 시리즈로 이어가겠습니다.

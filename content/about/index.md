---
title: "소개"
description: "Techy Stack은 이런 블로그입니다."
date: 2026-04-30
---

## Techy Stack

Kubernetes를 비롯한 클라우드 네이티브 스택을 깊게 파고드는 개인 기록입니다. 공식 문서에 이미 잘 정리되어 있는 기본 개념을 반복하기보다는, 파라미터 하나의 뒷면이나 컴포넌트 사이의 경계처럼 **한 번쯤 헷갈리지만 파고들 시간을 따로 내기는 어려운 지점**들을 매일 하나씩 정리해 올립니다.

## 이 블로그가 따르는 원칙

1. **사실에 기반합니다** — 숫자, 기본값, 버전은 모두 KEP, 공식 문서, 소스 코드, 또는 실제 클러스터 출력 중 하나로 근거를 댑니다.
2. **대비로 설명합니다** — 새 개념은 기존에 이미 아는 기술과 짝지어 도입합니다.
3. **재현 가능한 실습이 붙습니다** — 모든 글에 실제 EKS 클러스터에서 돌아가는 YAML과 `kubectl` 명령, 그리고 실제 출력이 함께 붙습니다.
4. **버전을 명시합니다** — 기준은 Kubernetes 1.35 (EKS 지원 최신 버전)입니다.

## 실험 환경

- 클러스터: `funny-pop-mountain` (us-east-1)
- Kubernetes 1.35, EKS Auto Mode, Bottlerocket, containerd 2.1.6
- Multi-AZ, multi-arch (arm64 + amd64)

## 연락

- GitHub: [techystacks](https://github.com/techystacks)

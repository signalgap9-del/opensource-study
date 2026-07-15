# opensource-study

오픈소스 프로젝트를 읽으면서 설계 감각을 쌓는 공부 기록.

목표는 "코드를 많이 본다"가 아니라, 좋은 오픈소스가 어떤 알고리즘, 수학, 표준, 운영 현실, PR 대화 위에서 만들어지는지 추적하는 것.

## 공부 중인 프로젝트

| 프로젝트 | 레포 | 읽는 관점 |
| --- | --- | --- |
| protobom | https://github.com/protobom/protobom | 추상화가 전쟁을 종식시킨다 |
| rekor | https://github.com/sigstore/rekor | 알고리즘이 신뢰를 만든다 |
| fulcio | https://github.com/sigstore/fulcio | 단순함이 복잡성을 이긴다 |
| dependabot-core | https://github.com/dependabot/dependabot-core | 현실은 이상보다 100배 더럽다 |

## 나중에 볼 프로젝트

| 프로젝트 | 레포 | 메모 |
| --- | --- | --- |
| sigstore-go | https://github.com/sigstore/sigstore-go | 지금은 clone 없이 링크만 기록 |

## 공부 프레임

각 프로젝트를 볼 때 아래 5가지를 같이 본다.

| 축 | 볼 것 | 질문 |
| --- | --- | --- |
| 알고리즘/수학 | Merkle tree, graph, DAG, version solving, signature verification | 어떤 문제를 O(log N), DFS, constraint solving 같은 구조로 바꿨나? |
| 암호학/신뢰 | OIDC, x509, certificate transparency, TUF, Ed25519 | 신뢰를 사람의 약속이 아니라 검증 가능한 데이터로 어떻게 바꿨나? |
| 시스템 아키텍처 | API boundary, storage, sharding, registry, native helpers | 어디서 추상화하고, 어디서 현실의 지저분함을 받아들였나? |
| 표준 문서 | SPDX, CycloneDX, OCI, OIDC, RFC 5280, RFC 6962 | 코드가 어떤 표준의 어떤 조항을 구현하나? |
| PR/이슈 대화 | design discussion, revert, bug report, review comments | 메인테이너들이 왜 이 선택을 했고, 어떤 대안을 버렸나? |

## 프로젝트별 핵심 컨텍스트

### protobom

- 핵심 문제: SPDX와 CycloneDX 사이의 손실 없는 SBOM 추상화.
- 알고리즘/수학: graph model, edge direction, ID mapping, cycle detection, DAG validation.
- 아키텍처: reader -> internal protobuf model -> writer 구조를 본다.
- 표준: SPDX, CycloneDX, Package URL.
- PR/이슈에서 볼 대화: "원본 SBOM 포맷을 얼마나 보존해야 하는가?", "공통 모델이 표준별 세부 정보를 어디까지 품어야 하는가?"
- 시작 링크:
  - https://github.com/protobom/protobom
  - https://github.com/protobom/protobom/issues/213
  - https://github.com/protobom/protobom/pulls?q=is%3Apr+is%3Aclosed+design

### rekor

- 핵심 문제: "이 서명/메타데이터가 append-only transparency log에 들어갔다"를 검증 가능하게 만드는 것.
- 알고리즘/수학: Merkle tree, inclusion proof, consistency proof, signed tree root, O(log N) verification.
- 아키텍처: REST API personality와 transparency log backend 사이의 번역, storage, sharding, UUID -> log index mapping.
- 표준/개념: transparency log, Trillian, Rekor entry, checkpoint.
- PR/이슈에서 볼 대화: "log sharding은 언제 필요한가?", "inclusion proof를 API에서 어떻게 노출할 것인가?", "에러를 HTTP와 gRPC 사이에서 어떻게 번역할 것인가?"
- 시작 링크:
  - https://github.com/sigstore/rekor
  - https://docs.sigstore.dev/logging/overview/
  - https://docs.sigstore.dev/logging/cli/
  - https://github.com/sigstore/rekor/pulls?q=is%3Apr+inclusion+proof

### fulcio

- 핵심 문제: OIDC identity를 짧게 살아있는 code signing certificate로 바꾸는 것.
- 알고리즘/수학: public/private key possession, signature verification, certificate chain validation.
- 암호학: OIDC token, x509, SAN, certificate transparency, short-lived certificate.
- 아키텍처: client가 ephemeral keypair 생성 -> OIDC token 획득 -> Fulcio가 token과 key possession 검증 -> certificate 발급 -> CT log에 기록.
- PR/이슈에서 볼 대화: "어떤 OIDC issuer를 신뢰할 것인가?", "인증서에 어떤 claim을 넣을 것인가?", "10분짜리 certificate가 어떤 공격면을 줄이는가?"
- 시작 링크:
  - https://github.com/sigstore/fulcio
  - https://docs.sigstore.dev/certificate_authority/overview/
  - https://docs.sigstore.dev/certificate_authority/certificate-issuing-overview/
  - https://docs.sigstore.dev/certificate_authority/oidc-in-fulcio/
  - https://github.com/sigstore/fulcio/pulls?q=is%3Apr+OIDC

### dependabot-core

- 핵심 문제: dependency update라는 깔끔해 보이는 문제를 20개 넘는 언어/패키지 매니저의 현실 속에서 처리하는 것.
- 알고리즘/수학: SemVer, version range, dependency graph, constraint solving, lockfile diff.
- 아키텍처: file fetcher -> file parser -> update checker -> file updater -> metadata finder.
- 현실 포인트: native package manager shell-out, monkey patch, ecosystem별 예외, 오래된 lockfile과 깨진 registry 대응.
- PR/이슈에서 볼 대화: "패키지 매니저의 동작을 직접 재구현할 것인가, 실제 도구를 실행할 것인가?", "어떤 파일을 dependency source로 볼 것인가?", "업데이트 PR 설명이 틀렸을 때 신뢰가 어떻게 깨지는가?"
- 시작 링크:
  - https://github.com/dependabot/dependabot-core
  - https://github.com/dependabot/dependabot-core/blob/main/npm_and_yarn/lib/dependabot/npm_and_yarn.rb
  - https://github.com/dependabot/dependabot-core/issues/1736
  - https://github.com/dependabot/dependabot-core/issues/11376
  - https://github.com/dependabot/dependabot-core/pulls?q=is%3Apr+file_fetcher+file_parser+update_checker

### sigstore-go

- 핵심 문제: Sigstore signing/verification을 Go 라이브러리로 단순하고 안정적으로 제공하는 것.
- 알고리즘/수학: certificate chain validation, Rekor inclusion proof verification, bundle verification, timestamp validation.
- 아키텍처: verifier, bundle parser, TUF trusted root, Rekor/Fulcio verification material.
- 지금 상태: clone은 나중에 하고, 당장은 링크와 개념만 기록.
- 시작 링크:
  - https://github.com/sigstore/sigstore-go
  - https://docs.sigstore.dev/language_clients/go/

## 시스템으로 연결해서 보기

Sigstore 계열은 따로따로가 아니라 하나의 신뢰 파이프라인으로 본다.

```text
developer / CI
  -> ephemeral keypair
  -> OIDC identity token
  -> Fulcio certificate
  -> artifact signature
  -> Rekor transparency log
  -> Merkle inclusion proof
  -> verifier / policy decision
```

여기서 `fulcio`는 identity를 certificate로 바꾸고, `rekor`는 signing event를 append-only log로 남기고, `sigstore-go`는 이 재료들을 검증 가능한 API로 묶는다.

## PR/이슈 대화 읽는 법

1. 먼저 README와 docs로 현재 설계를 본다.
2. 그 다음 `issues`에서 bug, design, enhancement 키워드를 본다.
3. 닫힌 PR을 읽으면서 "무엇이 바뀌었나"보다 "왜 이 방식이 선택됐나"를 본다.
4. revert 커밋이나 오래 열린 이슈를 찾는다. 보통 거기에 진짜 설계 갈등이 있다.
5. 코드 diff, 리뷰 댓글, 테스트 추가를 한 세트로 읽는다.

좋은 검색어:

- `inclusion proof`, `consistency proof`, `sharding`, `checkpoint`
- `OIDC`, `issuer`, `certificate`, `SAN`, `challenge`
- `SPDX`, `CycloneDX`, `relationship`, `license`, `format`
- `file_fetcher`, `file_parser`, `update_checker`, `native_helpers`, `monkey patch`

## PR case studies

| 프로젝트 | PR | 상태 | 왜 보는가 |
| --- | --- | --- | --- |
| dependabot-core | [#15187](notes/prs/dependabot-core-15187.md) | open, changes requested | C# file-based app 지원에서 parser, SDK semantics, dependency solver, experiment flag 정책이 어떻게 충돌하는지 보기 |

통과된 PR을 볼 때는 `is:pr is:merged` 검색으로 시작하고, 아직 안 통과된 PR은 `changes requested`에서 설계 갈등을 본다.

## 첫 번째 깊게 팔 주제

Rekor의 Merkle Tree inclusion proof부터 시작한다.

손으로 해야 할 것:

1. leaf 8개짜리 Merkle tree를 직접 그린다.
2. target leaf index 3의 sibling path를 계산한다.
3. root hash까지 올라가는 과정을 적는다.
4. "왜 leaf 전체를 다시 받지 않고 O(log N)개의 hash만으로 검증되는가"를 한 문단으로 설명한다.
5. 그 다음 Rekor 코드와 PR/이슈에서 inclusion proof가 API로 어떻게 노출되는지 본다.

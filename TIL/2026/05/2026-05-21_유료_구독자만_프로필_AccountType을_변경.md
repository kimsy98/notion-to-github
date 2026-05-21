# 유료 구독자만 프로필 AccountType을 변경

> 날짜: 2026-05-21
> 원본 노션: [링크](https://www.notion.so/AccountType-366edfd5a47680838228f6d468ac214c)

---

이슈

## 1. 목적

현재는 모든 유저가 프로필 내부의 accountType을 자유롭게 변경할 수 있지만 이를 유료 구독자(ONE_STAR / TWO_STAR / THREE_STAR) 로 제한한다.

## 2. 작업내용

- UserSubscription에 canModifyAccountType(now, zoneId) 추가 — 유료 구독 + ACTIVE/CANCELED/PAST_DUE + KST 기준 만료일까지 허용
- AccountTypeChangeNotAllowedException 신규 (HTTP 403, UserException sealed 계층 등록)
- UserSubscriptionService.validateAccountTypeChangeable(user, accountType) 추가 — 타깃이 GENERAL이면 통과, 그 외에는 canModifyAccountType 검증
- UserUseCase.updateProfile 진입부에서 이미지 처리 전 검증 호출
- UserSubscriptionUseCase.expireUserSubscriptions에서 만료 처리 직후 userService.resetAccountTypeToGeneral(user)로 환원
(GENERAL 환원은 구독 상태와 무관하게 항상 허용되므로 별도 예외 처리 불필요)
## 3. 체크리스트

- [ ] 일반 유저가 accountType=CELEBRITY 로 PUT 시 403 반환
- [ ] 유료 구독자 (UserSubscription의 status가 ACTIVE)가 accountType 변경 시 정상 처리
- [ ] CANCELED 상태 + 만료일 이전 유저는 변경 가능
- [ ] EXPIRED 또는 expiresAt 경과 시 변경 불가




### 작업 흐름

1. canModifyAccountType(now, zoneId)  
1. AccountTypeChangeNotAllowedException : 사용자 권한 변환 커스텀 예외 생성 (구독중이지 않은 사람들 요청 시 예외 반환)
1. UserSubscriptionService.validateAccountTypeChangeable(user, accountType) 추가 
1. UserUseCase.updateProfile 진입부에서 이미지 처리 전 검증 호출




# Tip

validateXxx 네이밍이면 내부에서 검증 실패 시 throw하고 반환값은 없는(void/Unit) 게 컨벤션

 마이크로 최적화 지양 : 오히려 가독성을 해치고 유지보수를 어렵게 만듭니다. 설계/구조 최적화와 병목 현상(Bottleneck) 분석 등 거시적인 관점의 접근이 훨씬 효율적


### DRY 함정

메서드의 의도, 미래 분기 가능성이 있다면 중복 으로 그냥 둬라

- DRY 의 함정 — "비슷해 보이는 코드 = 중복" 으로 묶기 쉬운데, 이게 가장 흔한 잘못된 추상화의 시작입니다. Sandi Metz 의 격언: "Duplication is far cheaper than the wrong abstraction." 두 메서드가 의도가 다르고 미래에 분기될 가능성이 있다면, 중복으로 보여도 따로 두는 게 정답인 경우가 많아요. 우리 코드가 정확히 이 함정에 빠졌어요.
- 퇴사자 코드가 의외로 좋을 수 있는 이유 — 퇴사한 사람이 짜놓은 코드를 보면 "그 사람이 이렇게 짠 이유" 의 컨텍스트가 사라져서 잘못 보이기 쉽습니다. 하지만 그 코드를 짤 당시에 도메인을 깊이 이해하고 있었다면, 우리가 놓치는 미묘한 의도가 박혀 있을 수 있어요. 이번 케이스의 "검증 로직을 인라인으로 둔 것" 도 단순히 게으름이 아니라 의도적인 응집성 선택 일 가능성이 높습니다.
- 제가 1번 작업 때 놓친 분석 — 그때 "(a) 검증을 도메인 밖에서 끝내기 (b) 값 객체로 표현" 같은 옵션을 제시했지만, (c) 인라인으로 도메인 메서드 안에 다 적기 옵션을 충분히 검토 안 했어요. 라인 수 적은 것이 항상 좋다는 무의식적 편향이 작동한 거 같습니다. "더 짧은 코드 = 더 좋은 코드" 가 아니라 "의도를 가장 명확히 표현하는 코드 = 더 좋은 코드" 가 정답입니다.
- "리뷰 가치 있는 코드" 의 정의 — 사용자가 이번에 두 버전을 비교해주신 것 자체가 좋은 코드 리뷰의 모범입니다. 단순히 "기존 코드 따라가자" 가 아니라 "왜 그게 더 좋은가" 를 같이 분석하는 흐름. LLM 입장에서도 "다른 사람의 의도와 우리 결정의 trade-off" 를 학습할 수 있는 좋은 기회예요.



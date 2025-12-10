# 외부 api 인증/인가 설계

외부 파트너 API 인증은 **API Key + Secret + HMAC** 방식으로 설계하겠습니다.

### 1) 인증(Authentication)

파트너에게 API Key와 Secret을 발급합니다.
Secret은 파트너 **백엔드 서버 환경변수**나 Secret Manager에 저장하여 노출을 방지합니다.

클라이언트(파트너 서버)는 요청 시 다음 헤더를 포함합니다:

* **x-api-key**: 발급된 API 키
* **x-timestamp**: 요청 시각(재전송 공격 방지용)
* **x-signature**:

  ```
  signature = HMAC_SHA256(secret, timestamp + request_body)
  ```

서버는:

1. x-api-key로 Secret 조회
2. 동일한 방식으로 signature 생성
3. x-signature와 비교해 위변조 여부 확인
4. timestamp가 허용 범위(예: 5분)인지 점검하여 replay 방지

### 2) 인가(Authorization)

API Key에 **scope/permission 정보를 포함**시켜
요청한 API가 허용된 권한인지 확인 후 처리합니다.

### 3) 키 관리

API Key 유출 의심 시:

* 서버에서 해당 키 폐기
* 새로운 키 발급
* 필요하면 일정 기간 Secret rotation 지원

---

# ✔️ 결론

지금 정리한 방향 + 위 보완 내용까지 넣으면
**HMAC 인증 구조를 완벽하게 이해한 백엔드 엔지니어 답변**이 된다.

원하면 “AWS Signature V4처럼 더 안전하게 만들기” 버전도 정리해줄게.

# 티겟 예약 시스템

## 요구사항
- 예약이 시작되면 클라이언트에서 다수의 사용자가 예약을 시도할 수 있음
- 트래픽 분산을 위해 입장 대기열 기능이 필요함
- 입장 순번이 되면 예약 화면으로 이동
- 예약 화면에서 좌석의 예약 상태를 실시간으로 확인할 수 있음

---
## 구현 방안

사용자가 티켓 예약 시도 할 경우 번호표 서버에 요청이 먼저 전달된다 

번호표 서버는 순서대로 번호 토큰을 사용자에게 전달한다 

번호는 redis incr을 사용하여 race condition을 피한다 
> 토큰은 jwt이고 페이로드에 event_id, number, user_id 등이 담겨잇다 번호표 토큰을 받은 

사용자는 대기열 현황 서버에 폴링 방식으로 남은 순서와 입장여부를 확인한다 
api/queue/status?ticket_nubmer=xxxx 

대기열 현황 서버는 api/queue/status?ticket_nubmer=xxxx api를 제공하고 사용자의 번호표와 레디스에서 current_enter:event_id 를 확인하여 입장 가능한지 판단한다 
> current_enter:event_id > 번호표 조건이 참이면 입장 

입장 가능하다면 예약서버로 리다이렉트 시킨다 

대기열 관리 워커에서는 예약 서버의 cpu, 메모리 사용량, 남은 좌석수 등을 파악하여 레디스의 current_enter:event_id 값을 증가시킨다 (50/s)

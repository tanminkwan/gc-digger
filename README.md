아래 조건에 맞춰서 이 GC log 를 최대한 똑 같이 재현하는 시스템 환경 Setting 방법과 시뮬레이션 code 를 작성해줘.
- java로 구현
- 초당 1회 온라인 transaction을 발생 시 sample GC log를 재현하는 WAS 탑재용 application 구현
- 온라인 transaction 을 특정시간 동안 주기적으로 발생시키는 batch program 개발
- application과 독립적으로 실시간 Matric 수집 및 시각화 기능 구현
- 상수는 하드코딩하지 말고 환경 변수로 설정 가능하게 해
- 실행 방법에 대한 매뉴얼 작성

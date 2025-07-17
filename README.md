# 막차 경로 탐색 및 알림 서비스 "앗차!"

막차 놓치고 후회했던 경험, 다들 있죠?
**앗차는 복잡한 검색 없이, 막차 경로와 출발 타이밍을 똑똑하게 챙겨주는 
막차 알림 서비스입니다.**

## 프로젝트 기간 및 인원
- 기간 : 2025.01 ~ 2025.04
- 인원 : 총 10명 (**Backend 3명**, Android 4명, Designer 3명)
- 링크 : [Github Link](https://github.com/depromeet/16th-team6-BE), [구글 스토어](https://play.google.com/store/apps/details?id=com.depromeet.team6&hl=ko)

# 서비스 화면
<div style="display: flex; justify-content: space-between;">
  <img src="https://github.com/user-attachments/assets/b88d1dbc-2912-4715-b6b9-32b2c1fa7f66" width="150" height="300">
  <img src="https://github.com/user-attachments/assets/719c9861-6240-4467-933b-cc45386c75bb" width="150" height="300">
  <img src="https://github.com/user-attachments/assets/aa8dcb47-0c01-433d-b720-302a4b26b350" width="150" height="300">
</div>
<div style="display: flex; justify-content: space-between;">
  <img src="https://github.com/user-attachments/assets/1d1ce1bb-2853-4311-8ac8-a27110131092" width="150" height="300">
  <img src="https://github.com/user-attachments/assets/9bed851d-8e69-44c1-a2fc-a1bbf08ae6bf" width="150" height="300">
  <img src="https://github.com/user-attachments/assets/4d9f76bd-c2e4-4c97-ab68-5d194bef4bb0" width="150" height="300">
</div>

# 담당 업무

- **API 서버 개발**
    - POI(장소) 검색
    - 리버스 지오코딩
    - 최근 검색 내역 관리
    - 버스 실시간 정보 연동
    - 수도권 버스/지하철 막차 시간 조회
- **캐싱 시스템 구축**
    - Redis를 활용한 데이터 캐싱 전략 수립
    - 변경 빈도가 낮은 공공 데이터 캐싱으로 외부 API 호출 최소화
    - 경로 검색 결과 임시 캐싱으로 동일 쿼리 응답 속도 향상
- **외부 API 호출 성능 최적화**
    - 코루틴 기반 API 병렬 호출 구현
    - 외부 API 요청 병렬 처리로 막차 경로 조회 시간 65% 개선
    - 타임아웃 및 장애 상황 대응 오류 처리 구현
 
# 사용 기술

- **백엔드**
    - Kotlin, Spring Boot
    - MySQL, Redis
- **클라우드**
    - NCP, GCP
- **CI/CD**
    - Github Actions
    - Docker
- **Tools**
    - Slack, Notion
 
# 시스템 아키텍쳐
<img width="939" alt="안녕하세요 백엔드 개발자 강지원입니다 (6)" src="https://github.com/user-attachments/assets/fa96fd52-7610-4411-b901-3d3280d51013" />

- 개발 환경은 운영환경과 최대한 유사하도록 Docker를 통해 구성
- 운영 환경은 앞단에 로드밸런서를 두고 서버 2대로 운영
- MySQL과 Redis 각 2대의 레플리케이션
- 프로메테우스, 로키, 그라파나를 통해 모니터링

# 챌린지

### 1. 막차 경로 탐색 응답시간 최적화
**문제상황**: 요청 한 건에 평균 200회가 넘는 외부 API 호출로 인한 심각한 응답 지연

**해결방안**:
- **캐싱 전략**: 외부 API 응답 결과 캐싱 + 역/정류장별 시간표 패턴 분석 기반 캐싱
- **병렬 처리**: 코루틴 기반 외부 API 호출 병렬화
- **SSE 스트리밍**: 실시간 결과 전송으로 사용자 체감 응답시간 단축

**성과**: 
<img width="2048" height="843" alt="image" src="https://github.com/user-attachments/assets/1859df52-5ab9-4570-a991-9bbcb1bd71a8" />

- 사용자 체감 첫 경로 표시 시간 **3.1초 → 1.5초 (52% 단축)**

### 2. 막차 경로 탐색 정확도·비용 최적화
**문제상황**: 
- 공공데이터 API 의존으로 인한 71% 성공률
- 유료 API(ODSay) 사용 시 인한 월 50만원 비용 부담
- 단순 역명 비교로 인한 노선 오탐

**해결방안**:
- **폴백 시스템**: 공공데이터 API 실패 시 ODSay API 자동 폴백
- **LCS 알고리즘**: 노선 판정에 Longest Common Subsequence 기반 유사도 적용
- **스마트 API 호출**: 폴백 시에만 유료 API 호출

**성과**:
- 성공률 **71% → 91% 달성**
- **월 50만원 API 비용 전액 절감**
- 서비스 가용성 및 사용자 만족도 향상

### 3. 지하철 막차 판단 알고리즘 구현
**문제상황**: 외부 경로 API만으로는 지하철 상/하행 정보와 지선 구조 판단 불가

**해결방안**:
- **데이터 수집**: 국토교통부 도시철도 노선정보 API + 22개 지하철 노선 지선 분기 정보 직접 분석
- **DB 구조화**: `subway_branch` 테이블을 통한 지선별 분기 구조 관리
- **알고리즘 구현**: 출발역-도착역-종점역 동일 지선 여부 및 경로 유효성 검증

```sql
CREATE TABLE `subway_branch` (
  `id` bigint NOT NULL AUTO_INCREMENT,
  `ord` int NOT NULL,
  `final_station_name` varchar(255) DEFAULT NULL,
  `route_code` varchar(255) DEFAULT NULL,
  `route_name` varchar(255) DEFAULT NULL,
  `station_name` varchar(255) DEFAULT NULL,
  PRIMARY KEY (`id`)
)
```

**성과**: 지선이 복잡한 노선에서도 사용자 경로 기준 **정확한 막차 시간 계산** 구현

### 4. 수도권 버스 통합 처리
**문제상황**: 서울/경기/인천 각 지역별 상이한 공공데이터 API 포맷 및 파라미터

**해결방안**:
- **전략 패턴**: 지역별 버스 API 요청과 파싱 로직을 전략 객체로 분리
- **인터페이스 통일**: 공통된 처리 흐름 정의
- **런타임 동적 주입**: 사용자 위치 기반 적절한 전략 선택

```kotlin
class BusManager(
    private val stationClientMap: Map<ServiceRegion, BusStationInfoClient>,
    private val routeClientMap: Map<ServiceRegion, BusRouteInfoClient>,
    private val regionIdentifier: RegionIdentifier
) {
    fun getArrivalInfo(routeName: String, stationMeta: BusStationMeta): BusArrival? {
        val region = regionIdentifier.identify(stationMeta.coordinate)
        val station = stationClientMap[region]?.getStationByName(stationMeta)
        val route = stationClientMap[region]?.getRoute(station, routeName)
        return routeClientMap[region]?.getBusArrival(station, route)
    }
}
```

**성과**: 
- 서울/경기도 버스 API 일관된 구조 통합
- 중복 로직 제거 및 간결한 코드 구조 유지
- 새로운 지역 추가 시 전략 객체만 구현하면 되는 확장성 확보

## 🛠 기술 스택
- **Backend**: Spring Boot, Kotlin
- **Database**: MySQL
- **External APIs**: 공공데이터 API, ODSay API, TMAP API
- **Optimization**: 코루틴 기반 병렬 처리, SSE 스트리밍, 캐싱

## 🎯 핵심 개발 철학
- **도메인과 인프라 레이어 분리**를 통한 기술 의존성 최소화
- **전략 패턴**을 활용한 확장 가능한 아키텍처 설계
- **데이터 기반 의사결정**으로 실질적인 성능 개선 달성
- **사용자 경험 최우선**의 기술적 문제 해결

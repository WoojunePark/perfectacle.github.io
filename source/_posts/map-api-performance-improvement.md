---
title: 지도 API 성능 개선기
tags:
  - API
  - Performance
category:
  - Note
  - Dev
date: 2017-09-28 23:29:16
---

![](thumb.png)  

## 문제점
![크롬 개발자 도구 네트워크 탭에서 본 API 호출](01.png)  
Time은 데이터 전체를 파싱하는데 걸린 시간이니 무시하고...
트래픽이 13MB 남짓...  
사용자가 조건을 바꿔서 검색을 한다면 데이터 광탈범이 될 가능성이 다분한 상황이었다.  

![순수 응답 시간을 알고 싶어서 다운로드 받아봄](03.png)  
응답 시간이 22초 남짓...  

![StopWatch를 통해 어느 작업에서 병목이 가장 많이 발생하는지 파악](16.png)  

## 원인 파악
### 쿼리
![숙박 연동사 실제 가격 주입하는 부분](17.png)  
숙박 연동 최저가는 jooq로 불러오고 있고, 숙박 연동사 테이블은 jpa로 불러오고 있음.  
순수 네이티브 쿼리가 아닌 이상 퍼포먼스가 제대로 나오지 않을 것으로 판단됨.  

![딜 목록 불러오는 쿼리](05.png)  
실제로 필요한 건 특정 필드 뿐인데 모든 필드를 다 긁어오고 있어서 쿼리 실행속도가 느려진 것임.  

### 용량
![실제로 저장된 응답값](08.png)  
이 응답값에는 세 가지 문제점이 존재한다.

![쓸 데 없는 공백을 포함하고 있었다.](09.png)  
이 데이터를 줄였을 때 1MB 정도 가량이 줄어들었다.

![쓸 데 없는 컬럼들도 포함하고 있었다.](10.png)  
treeAllId라던지, clusterName이나 빈 배열 등등 다른 값들을 가지고 유추할 수 있는 값들을 제거하였다.
딱히 이 부분에서는 데이터를 크게 줄일 수가 없었다.  
  
![컬럼의 값을 가공하지 않고 그대로 들고 있다.](11.png)  
이미지의 URL을 담고 있는 컬럼을 불러와서 필요한 정보만 뿌려주는 게 아니라 모든 데이터를 가공없이 뿌려주고 있었다.  
이 컬럼의 데이터가 하나의 딜에 대한 데이터의 3/4 이상을 차지하고 있었다.  
대부분의 쓸 데 없는 데이터가 여기서 낭비되고 있었다.

## 문제 해결
### 아예 API 서버로 따로 분리
맵 API는 방대한 양의 데이터를 가져오므로 서버의 리소스 사용이 많아 아예 별도의 서버로 구축하기로 판단했다.  
하지만 실제 가격을 주입하기 위해서는 유저의 등급이 필요한데 API 서버에는 유저에 대한 정보를 가지고 있지 않기 때문에 아래와 같은 구조로 구성하게 됨.  
![서버 구조](server.png)  
가자고 프론트 서버는 타지 않는 게 제일 베스트인데 유저 정보에 따른 실제 가격 주입 때문에 어쩔 수 없이 넣게 되었다.  

### 빠른 응답속도 보장
클러스터와 딜을 함께 내려주다보니 초기에 유저가 기다려야하는 속도는 15~20초 남짓 대기해야한다.    
![이 화면에서 클러스터를 그리기 위한 정보는 중심 좌표, 지역 코드, 딜 갯수가 끝이다.](12.png)  
굳이 딜 목록까지 내려 줄 필요가 없다고 판단이 들어서 클러스터(갯수 포함) 따로 딜 따로 내려주게 끔 API를 두 개로 분리하였다.  
```
/api/v1/map/hotel (클러스터)
/api/v1/map/hotel/deals (딜, 기존 API)
```
클러스터를 그리기 위한 클러스터 API는 응답 속도가 1~2초 남짓이라 유저가 불편을 느끼지 못할 수준이다.  
유저가 방심(?)하는 사이에 몰래(?) 딜을 뿌려주는 API를 호출하고 있으면 웬만한 유저들에게는 불편함을 주지 않을 것이다.  

### 중간 점검
![데이터는 2MB 남짓으로 줄어들었고, 응답속도도 11초 가량 걸렸다.](18.png)
  
![가장 오래 걸리는 게 숙박 딜 실제 가격 주입 부분이다.](19.png)  
아직 이정도 시간 가지고는 서비스 하기에는 무리가 있어 보였다.  

### 캐싱하기
**딜들과 카테고리 ID를 매핑하는 부분**과 **숙박 딜의 실제 가격을 주입하는 부분**은 애초부터 판매 중인 모든 딜에 대한 정보만 들고있으면 된다.  
즉, 조건에 구애받지 않는다는 뜻이다.  
```java
@Service
public class Job {
    private HotelDealMinPricesMapper hotelDealMinPricesMapper;
    private DealPartnersMapper dealPartnersMapper;
    private TreeDealMapMapper treeDealMapMapper;
    private CategoryIds categoryIds;
    private HotelPartnersAndPrices hotelPartnersAndPrices;

    @Inject
    public Job(
            HotelDealMinPricesMapper hotelDealMinPricesMapper, DealPartnersMapper dealPartnersMapper,
            TreeDealMapMapper treeDealMapMapper, CategoryIds categoryIds, HotelPartnersAndPrices hotelPartnersAndPrices
    ) {
        this.hotelDealMinPricesMapper = hotelDealMinPricesMapper;
        this.dealPartnersMapper = dealPartnersMapper;
        this.treeDealMapMapper = treeDealMapMapper;
        this.categoryIds = categoryIds;
        this.hotelPartnersAndPrices = hotelPartnersAndPrices;
    }

    /** crontab
     1.Seconds
     2.Minutes
     3.Hours
     4.Day-of-Month
     5.Month
     6.Day-of-Week
     7.Year (optional field)
     */
    @PostConstruct
    @Scheduled(cron = "0 0/30 * * * ?")
    public void setLeisureAndHotelCategoryIds() {
        categoryIds.setLeisureCategoryIds(treeDealMapMapper.selectLeisureCategoryId());
        categoryIds.setHotelCategoryIds(treeDealMapMapper.selectHotelCategoryId());
    }

    @PostConstruct
    @Scheduled(cron = "0 0/30 * * * ?")
    public void setPricesAndPartners() {
        hotelPartnersAndPrices.setPartners(dealPartnersMapper.selectDealPartnersAll());
        hotelPartnersAndPrices.setPrices(hotelDealMinPricesMapper.selectMinPricesAll());
    }
}
```
서버를 띄웠을 때 최초 1회, 30분 마다 캐싱하도록 설정하였다.  

### 또 다시 중간 점검
![4~6초 가량으로 줄어들었다.](20.png)  

![최대 오래 걸리는 게 딜 목록을 불러오는 부분이다.](21.png)  

![실제 쿼리 실행은 0.01초도 안 걸린다.](22.png)  

![MyBatis로 해당 쿼리를 실행하는데 걸린 시간은 1초가 넘는다.](24.png)

혹시 MyBatis라서 느린건가 싶어서 JDBC로 쿼리를 날려봤다.
```java
@Repository
public class HotelMapper2 {
    private JdbcTemplate jdbcTemplate;

    @Inject
    public HotelMapper2(JdbcTemplate jdbcTemplate) {
        this.jdbcTemplate =  jdbcTemplate;
    }

    public List<DealInMap> test() {
        String sql = "Select d.id, d.deal_nm, d.`STANDARD_PRICE`, d.zeropass_price, d.group_price, d.lat, d.lon,\n" +
                "d.tax_and_fee, ct.`tree_code`, d.deal_type, d.`LIST_IMAGE_JSON`\n" +
                "From DEAL_M d\n" +
                "Join TREE_DEAL_MAP tdm\n" +
                "    on d.id = tdm.deal_id\n" +
                "Join `CATEGORY_TREE` ct\n" +
                "    on tdm.`CATEGORY_TREE_ID` = ct.id and ct.tree_group_id = 27 and ct.depth = 2\n" +
                "Join HOTEL_DEAL_MIN_PRICES p\n" +
                "    on p.deal_id = d.id and p.expire_at > now() and p.ymd Between '2017-10-07' And '2017-10-07' and p.max_capacity >= 1\n" +
                "Where deal_status = 'IN_SALE' And d.display_yn = 'Y' and display_standard_yn = 'Y' And del_yn = 'N' And deal_type != 'DEAL'\n" +
                "And d.lat is not NULL And d.lon is not NULL\n" +
                "Group by d.id";
        return jdbcTemplate.query(sql, new BeanPropertyRowMapper<DealInMap>(DealInMap.class));
    }
}
```
![하지만 큰 변함은 없었다.](25.png)

![제일 데이터가 큰 컬럼인 LIST_IMAGE_JSON 컬럼을 빼자 속도가 3배 가량 빨라졌다.](26.png)

## 또 다시 캐싱 전략 세우기
따라서 모든 딜의 LIST_IMAGE_JSON 컬럼 또한 캐시하도록 하였다.  
```java
@PostConstruct
@Scheduled(cron = "0 0/30 * * * ?")
public void setDealsThumbnail() {
    thumbs.setLeisureThumbs(dealMMapper.selectLeisureThumbnail());
    thumbs.setHotelThumbs(dealMMapper.selectHotelThumbnail());
}
```

![훨씬 빨라진 실행 속도들](27.png)

그리고 숙박의 경우 사람들이 자주 이용하는 주말, 1박2일, 1~4인 조건의 내용을 캐싱하도록 하였다.
```java
// 매 달의 주말(지나간 날은 제외)을 구하는 스케쥴러
@PostConstruct
@Scheduled(cron = "0 0 0 1 * *")
public void setSaturdaysOfMonth() {
    LocalDate now = LocalDate.now();
    int year = now.getYear();
    Month month = now.getMonth();

    List<LocalDate> saturdaysOfMonth = IntStream.rangeClosed(LocalDate.now().getDayOfMonth(), YearMonth.of(year, month).lengthOfMonth())
            .mapToObj(day -> LocalDate.of(year, month, day))
            .filter(date -> date.getDayOfWeek() == DayOfWeek.SATURDAY)
            .collect(Collectors.toList());

    weekend.setSaturdays(saturdaysOfMonth);
}

// 주말, 1박2일, 1~4인 조건의 내용을 캐싱
@PostConstruct
@Scheduled(cron = "0 0/30 * * * ?")
public void setClusterGroup() {
    clusterGroupCache.setLeisureCluster(leisureCacheService.findLeisureClusters());
    clusterGroupCache.setLeisureClusterWithDeals(leisureCacheService.findLeisureClustersAndDeals());

    List<LocalDate> saturdays = weekend.getSaturdays();
    List<List<List<Cluster>>> clusterByWeekAndAdultCount = Lists.newArrayList();
    List<List<List<Cluster>>> clusterWithDealsByWeekAndAdultCount = Lists.newArrayList();
    for(int saturday=0, len=saturdays.size(); saturday<len; saturday++) {
        List<List<Cluster>> clusterByAdultCount = Lists.newArrayList();
        List<List<Cluster>> clusterWithDealsByAdultCount = Lists.newArrayList();
        for (int adultCount = 1; adultCount < 5; adultCount++) {
            HotelCriteria hotelCriteria = new HotelCriteria(
                    new StayPeriod(saturdays.get(saturday), saturdays.get(saturday).plusDays(1)), adultCount, UserGroup.STANDARD);
            clusterByAdultCount.add(hotelCacheService.findHotelClusters(hotelCriteria));
            clusterWithDealsByAdultCount.add(hotelCacheService.findHotelClustersWithDeals(hotelCriteria));
        }
        clusterByWeekAndAdultCount.add(clusterByAdultCount);
        clusterWithDealsByWeekAndAdultCount.add(clusterWithDealsByAdultCount);
    }
    clusterGroupCache.setHotelClusterInSaturdays(clusterByWeekAndAdultCount);
    clusterGroupCache.setHotelClusterInSaturdaysWithDeals(clusterWithDealsByWeekAndAdultCount);
}
```

## 차후 개선 사항 (시간 문제 및 공수와 효율성 문제)
* 2MB로 줄였다 하더라도 필터를 계속해서 바꾸다 보면 유저 입장에서는 부담되는 용량일 수도 있다.  
![또한 딜을 내려주는 API에서 반복되는 키값을 빼고 순서를 보장한 배열로 만들어 내려주는 형태로 바꿔주면 데이터를 0.5MB 이상 단축할 수 있다.](15.png)    

* 아직 주말의 경우에만 캐싱하도록 했는데 나중에는 공휴일과 휴일 전,후로 휴가를 써서 2박 3일을 다녀오는 경우도 있으니 그러한 케이스도 신경 써야겠다.   
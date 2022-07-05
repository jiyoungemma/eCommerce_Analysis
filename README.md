# 고객 데이터 분석을 통한 비지니스 인사이트 도출 프로젝트

### 데이터 출처
https://console.cloud.google.com/marketplace/product/bigquery-public-data/thelook-ecommerce?hl=ko&project=hip-fusion-340315

### 문제 정의
- 주문까지 이어지는 고객의 수는 계속해서 증가하는 추세를 보이나 신규 고객의 유입 수가 안정적이지 않음
- 다양한 가설을 설정하고 검정하며 신규 유저 유입 활성화를 위한 전략 수립 필요

### 가설 검정
- 광고를 통한 신규고객 유입이 검색의 경우보다 많다. → TRUE
   ```sql
   select count(distinct id) as count, traffic_source
   from `bigquery-public-data.thelook_ecommerce.users`
   group by traffic_source
   order by count;

   ```

- 광고를 통해 유입된 고객의 활성화정도는 그렇지 않은 고객에 비해 낮다. → FALSE
   
   ```sql
  select date_trunc(date (z.created_at), week) as week,
         COUNT(DISTINCT CASE WHEN z.current_order_date > 70 THEN z.id ELSE NULL END) AS ten_plus_weeks,
         COUNT(DISTINCT CASE WHEN z.current_order_date < 70 AND z.current_order_date >= 63 THEN z.id ELSE NULL END) AS nine_weeks,
         COUNT(DISTINCT CASE WHEN z.current_order_date < 63 AND z.current_order_date >= 56 THEN z.id ELSE NULL END) AS eight_weeks,
         COUNT(DISTINCT CASE WHEN z.current_order_date < 56 AND z.current_order_date >= 49 THEN z.id ELSE NULL END) AS seven_weeks,
         COUNT(DISTINCT CASE WHEN z.current_order_date < 49 AND z.current_order_date >= 42 THEN z.id ELSE NULL END) AS six_weeks,
         COUNT(DISTINCT CASE WHEN z.current_order_date < 42 AND z.current_order_date >= 35 THEN z.id ELSE NULL END) AS five_weeks,
         COUNT(DISTINCT CASE WHEN z.current_order_date < 35 AND z.current_order_date >= 28 THEN z.id ELSE NULL END) AS four_weeks,
         COUNT(DISTINCT CASE WHEN z.current_order_date < 28 AND z.current_order_date >= 21 THEN z.id ELSE NULL END) AS three_weeks,
         COUNT(DISTINCT CASE WHEN z.current_order_date < 21 AND z.current_order_date >= 14 THEN z.id ELSE NULL END) AS two_weeks,
         COUNT(DISTINCT CASE WHEN z.current_order_date < 14 AND z.current_order_date >= 7 THEN z.id ELSE NULL END) AS one_week,
         COUNT(DISTINCT CASE WHEN z.current_order_date < 7 THEN z.id ELSE NULL END) AS Less_than_a_week 
  from(
      select u.id, u.created_at, date_diff(date '2022-04-24', date (o.created_at),day) as current_order_date
      from `bigquery-public-data.thelook_ecommerce.order_items` as o
          join `bigquery-public-data.thelook_ecommerce.users` as u
              on u.id = o.user_id
      where concat(o.user_id, o.created_at) in (
          select concat(user_id, max(created_at))
          from `bigquery-public-data.thelook_ecommerce.order_items`
          group by user_id
      ) and u.traffic_source = 'Organic' or u.traffic_source = 'Email' or u.traffic_source = 'Display' and u.created_at <= '2022-04-24 23:59:59' and o.created_at <= '2022-04-24 23:59:59'
  ) as z
  group by 1
  order by 1;
   ```
- 시즌별 구매 특징이 존재한다. → FALSE
   ```sql
   select count(o.order_id) as season_order,
      case when format_datetime("%m", datetime (o.created_at)) = '03' or format_datetime("%m", datetime (o.created_at)) = '04' or format_datetime("%m", datetime (o.created_at)) = '05' then 'spring'
          when format_datetime("%m", datetime (o.created_at)) = '06' or format_datetime("%m", datetime (o.created_at)) = '07' or format_datetime("%m", datetime (o.created_at)) = '08' then 'summer' 
          when format_datetime("%m", datetime (o.created_at)) = '09' or format_datetime("%m", datetime (o.created_at)) = '10' or format_datetime("%m", datetime (o.created_at)) = '11' then 'autumn' 
          when format_datetime("%m", datetime (o.created_at)) = '01' or format_datetime("%m", datetime (o.created_at)) = '02' or format_datetime("%m", datetime (o.created_at)) = '12' then 'winter' end as weather
  from `bigquery-public-data.thelook_ecommerce.orders` as o
      join `bigquery-public-data.thelook_ecommerce.order_items` as i
          on o.order_id = i.order_id
      join `bigquery-public-data.thelook_ecommerce.products` as p
          on i.product_id = p.id
  where o.created_at <= '2022-04-24 23:59:59'
  group by weather;
   ```
- 리턴이 많은 제품의 특징이 존재한다. → TRUE 
   ```sql
   -- 국가 별 리턴 특징
   select  e.state, count(distinct o.user_id) as state_cnt
  from `bigquery-public-data.thelook_ecommerce.order_items` as o
    join `bigquery-public-data.thelook_ecommerce.products` as p
      on p.id = o.product_id
    join `bigquery-public-data.thelook_ecommerce.events` as e
      on o.user_id = e.user_id
  where o.created_at <= '2022-04-24 23:59:59' and o.returned_at is not null 
  group by e.state
  order by e.state;
  ```

   ```sql
   -- 가격별 리턴수량
   select 
    count(case when o.sale_price >= 0 and o.sale_price < 10 then 1 else null end) as one_to_10,
    count(case when o.sale_price >= 10 and o.sale_price < 20 then 1 else null end) as ten_to_20,
    count(case when o.sale_price >= 20 and o.sale_price < 30 then 1 else null end) as two_to_30,
    count(case when o.sale_price >= 30 and o.sale_price < 40 then 1 else null end) as thr_to_40,
    count(case when o.sale_price >= 40 and o.sale_price < 50 then 1 else null end) as fou_to_50,
    count(case when o.sale_price >= 50 and o.sale_price < 60 then 1 else null end) as fif_to_60,
    count(case when o.sale_price >= 60 and o.sale_price < 70 then 1 else null end) as six_to_70,
    count(case when o.sale_price >= 70 and o.sale_price < 80 then 1 else null end) as sev_to_80,
    count(case when o.sale_price >= 80 and o.sale_price < 90 then 1 else null end) as eig_to_90,
    count(case when o.sale_price >= 90 and o.sale_price < 100 then 1 else null end) as nin_to_100,
    count(case when o.sale_price >= 100 and o.sale_price < 200 then 1 else null end) as hun_to_200,
    count(case when o.sale_price >= 200 and o.sale_price < 300 then 1 else null end) as twohun_to_300,
    count(case when o.sale_price >= 300 and o.sale_price < 400 then 1 else null end) as thrhun_to_400
  from `bigquery-public-data.thelook_ecommerce.order_items` as o
    join `bigquery-public-data.thelook_ecommerce.products` as p
      on p.id = o.product_id
    join `bigquery-public-data.thelook_ecommerce.events` as e
      on o.user_id = e.user_id
  where o.created_at <= '2022-04-24 23:59:59' and o.returned_at is not null;
  ```
  
- 국가별 유입경로의 트렌드가 존재한다. → TRUE
   ```sql
   select traffic_source, created_at
  from `bigquery-public-data.thelook_ecommerce.events`
  where created_at <= '2022-04-24 23:59:59' and state = 'United States' and event_type = 'purchase'
  order by created_at;
  ```

### 마케팅 전략
- 유료 검색 엔진을 통한 신규가입고객 유입이 많음 → ***광고 활성화*** 필요 有
- 봄, 겨울의 주문량이 다른 계절보다 높음 → thelook의 상품이 ***봄, 겨울에 초점***이 맞춰져 있는지 확인 필요 有, 만약 사계절 모두에 중점을 둔다면 ***여름, 가을의 광고가 미약***한 것은 아닌지 확인 필요 有
- 광고를 통해 유입되어 구입한 고객의 리턴량이 가장 많음 → 광고에 포함된 내용이 과장되거나 중요한 내용이 빠져있어 ***구매확정까지 이어지는데 에러사항***이 되는 것은 아닌지 확인할 필요 有
- 이메일을 통한 유입이 가장 많음 → ***신규 가입고객 유인 전략*** 필요 有


---
### 사용 기술
- Bigquery
- SQL
- Google data studio

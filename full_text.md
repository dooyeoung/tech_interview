# 풀텍스트 검색 구현

흐름은 다음과 같다 문자열이 입력되면 n개씩 분리한다
> 노스페이스 > [노스, 스페, 페이, 이스, 노스페, 스페이, 페이스]
분리한 내용 상품이름과 각 문자열을 매핑하여 보관한다
"노스"라는 검색어가 입력되면 "노스"의 fulltext 텍스트를 테이블에서 찾고 해당 product_id를 바탕으로 제품 내용을 반환한다

## 테이블 구조
product
  id
  name varchar(100)

product_fulltext
  id
  text
  product_id
  index text

## 데이터 예시
데이터 예시는 다음과 같다

|product_id|name|
|-|-|
|1|노스페이스|

|fulltext_id|text|product_id|
|-|-|-|
|1|노스|1|
|2|스페|1|
|3|페이|1|
|4|이스|1|
|5|노스페|1|
|6|스페이|1|
|7|페이스|1|

## 검색

노스를 검색하면 아래와 같이 쿼리하여 제품을 찾는다
```
select * from
where id in (
    select distinct(product_id)
    from product_fulltext 
    where text like '노스%'
)
```

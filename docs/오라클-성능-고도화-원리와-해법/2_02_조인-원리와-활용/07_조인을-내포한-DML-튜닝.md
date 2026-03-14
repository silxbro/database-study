# [CH 2-7] 조인을 내포한 DML 튜닝

---
<br/>

## (1) 수정 가능 조인 뷰 활용

### [전통적인 방식의 UPDATE]
튜닝을 하다 보면 아래와 같이 작성된 udpate문을 종종 볼 수 있다.
```
update 고객 c
set    최종거래일시 = (select max(거래일시) from 거래
                    where  고객번호 = c.고객번호
                    and    거래일시 >= trunc(add_months(sysdate, -1)))
     , 최근거래횟수 = (select count(*) from 거래
                    where  고객번호 = c.고객번호
                    and    거래일시 >= trunc(add_months(sysdate, -1)))
     , 최근거래금액 = (select sum(거래금액) from 거래
                    where  고객번호 = c.고객번호
                    and    거래일시 >= trunc(add_months(sysdate, -1)))
where exists (select 'x' from 거래
              where  고객번호 = c.고객번호
              and    거래일시 >= trunc(add_months(sysdate, -1)))
```

만약 개발 중인 프로그램에 위와 같은 update문이 있다면 아래와 같이 고치기 바란다.
```
update 고객 c
set   (최종거래일시, 최근거래횟수, 최근거래금액) =
      (select max(거래일시), count(*), sum(거래금액)
       from   거래
       where  고객번호 = c.고객번호
       and    거래일시 >= trunc(add_months(sysdate, -1)))
where exists (select 'x' from 거래
              where  고객번호 = c.고객번호
              and    거래일시 >= trunc(add_months(sysdate, -1)))
```

위 방식에도 비효율이 없는 것은 아니다. 한 달 이내 거래가 있던 고객을 두 번 조회하기 때문인데, 총 고객 수와 한 달 이내 거래가 발생한 고객 수에 따라 성능이 좌우된다.

총 고객 수가 아주 많다면 Exists 서브쿼리를 아래와 같이 해시 세미 조인으로 유도하는 것을 고려할 수 있다.
```
update 고객 c
set   (최종거래일시, 최근거래횟수, 최근거래금액) =
      (select max(거래일시), count(*), sum(거래금액)
       from   거래
       where  고객번호 = c.고객번호
       and    거래일시 >= trunc(add_months(sysdate, -1)))
where exists (select /*+ unnest hash_sj */ 'x' from 거래
              where  고객번호 = c.고객번호
              and    거래일시 >= trunc(add_months(sysdate, -1)))
```

만약 한 달 이내 거래를 발생시킨 고객이 많아 update 발생량이 많다면 아래와 같이 변경하는 것을 고려할 수 있다.
하지만 **모든 고객 레코드에 lock이 발생**함은 물론, **이전과 같은 값으로 갱신되는 비중이 높을수록 Redo 로그 발생량이 증가해 오히려 비효율적**일 수 있다.
```
update 고객 c
set   (최종거래일시, 최근거래횟수, 최근거래금액) =
      (select nvl(max(거래일시), c.최종거래일시)
            , decode(count(*), 0, c.최근거래횟수, count(*))
            , nvl(sum(거래금액), c.최근거래금액)
       from   거래
       where  고객번호 = c.고객번호
       and    거래일시 >= trunc(add_months(sysdate, -1)))
```

이처럼 다른 테이블과 조인이 필요할 때 전통적인 방식의 update문을 사용하면 비효율을 감수해야만 한다.

참고로, set절에 사용된 서브쿼리에는 캐싱 메커니즘이 작용하므로 distinct value 개수가 적은 1쪽 집합을 읽어 M쪽 집합을 갱신할 때 효과적이다.
물론 exists 서브쿼리가 NL 세미 조인이나 필터 방식으로 처리된다면 거기서도 캐싱 효과가 나타난다.

### [수정 가능 조인 뷰]
아래와 같이 수정 가능 조인 뷰를 활용하면 참조 테이블과 두 번 조인하는 비효율을 없앨 수 있다.
```
update /+ bypass_ujvc */
 ( select /*+ ordered use_hash(c) */
          c.최종거래일시, c.최근거래횟수, c.최근거래금액
        , t.거래일시, t.거래횟수, t.거래금액
   from  (select 고객, max(거래일시) 거래일시, count(*) 거래횟수, sum(거래금액) 거래금액
          from   거래
          where  거래일시 >= trunc(add_months(sysdate, -1))
          group by 고객)t
        , 고객 c
   where  c.고객번호 = t.고객번호
)
set 최종거래일시 = 거래일시
  , 최근거래횟수 = 거래횟수
  , 최근거래금액 = 거래금액
```

'조인 뷰'는 from절에 두 개 이상 테이블을 가진 뷰를 가리키며, '수정 가능 조인 뷰(updatable/modifiable join view)'는 말 그대로 입력, 수정, 삭제가 허용되는 조인 뷰를 말한다.
단, 1쪽 집합과 조인되는 M쪽 집합에만 입력, 수정, 삭제가 허용된다.


## (2) Merge문 활용

## (3) 다중 테이블 Insert 활용
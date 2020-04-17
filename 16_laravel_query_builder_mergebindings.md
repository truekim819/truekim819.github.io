## [Laravel] eloquent - mergeBindings() 

회사에서 첫 개발을 맡았다. 

첫 개발은 간단한 이벤트 개발로 , 매니저님의 코드리뷰로 오늘 하루도 탈탈 털렸다. 

그 와중에 오늘 알게 된 것을 적어놓아야지 란 생각이 들어서 깃 블로그를 열었다. 

### Row Query

내가 쿼리빌더 , 또는 eloquent 모델로 변경해야 하는 쿼리는 아래와 같은 형태였다. 

```html
SELECT count(*)
FROM (
         SELECT no,
                SUM(price)
         FROM orders
         WHERE dt >= ‘’
           AND dt < ‘’
         GROUP BY no
         HAVING SUM(price) >= 500000
     ) as a
```

### 첫번째 쿼리문 작성 

일단 라라벨 docs에서 확인 가능한 eloquent orm 함수들을 이용해 작성했다. 

```markdown
Order::selectRaw(‘no’, ‘sum(price) as total_sum’)
->where(‘dt’,’>=‘, $startDate)
-> where(‘dt’,’<‘,$endDate)
->groupBy(‘no’)
->get()
->where(‘total_sum’ ,’>=‘,500000)
->count();
```
`매니저님 코멘트 : get() 수행 후 다시 where 절과 count를 가져오는 것은 부적절한 것 같다.
대상의 raw 수가 적으면 크게 상관 없겠지만, 쿼리량이 많아지게 되면 , 
해당 건에 대한 리스트를 가져오는 시간이 너무 오래 걸리게 된다. 
sub query를 쓰거나 join으로 해결 하거나 정 안 되면 raw 쿼리를 섞는게 좋을거 같다. `

### 두번쩨 쿼리문 작성 

매니저님의 코멘트에 따라 최대한 get() 이후에 다른 작업을 하지 않을려고 조물조물 해보았다. 
- groupBy 다음에 havingRaw()를 이용해 , get() 으로 리스트를 가져오기 전에 조건을 가져왔다. 
- 다만 쿼리 후에 , php 함수인 count([array]) 함수를 사용해 쿼리 사이즈를 계산했다. 

```markdown
$orders = Order::selectRaw(‘no’, ‘sum(price) as total_sum’)
->where(‘dt’,’>=‘, $startDate)
-> where(‘dt’,’<‘,$endDate)
->groupBy(‘no’)
->havingRaw(‘total_sum >= ?’, [500000])
->get()

Count($count);
```
`매니저님 코멘트 : get() 이후에 count([array]) 하는 부분에서 여전히 연산 부하가 생길거 같다. 
selectRaw가 반드시 필요한가 ? 더 좋은 함수가 있는지 찾아봐야 한다. `

### Last Query

이번엔 매니저님이 같이 찾아주셨다. 그  중 가장 베스트 쿼리함수를 찾았는데 
그게 바로 *mergeBindings()* [= Get a base query builder instance]. 이다. 
번역 하자면 쿼리빌더 인스턴스를 베이스로 가져오는 것이다. 

```markdown
#sub query를 먼저 작성한다. 매니저님의 말씀을 참고해 쿼리의 불필요한 부분을 제거하고 최대한 간략하게 만들었다.  
$subQuery = Order:: where(‘dt’,’>=‘, $startDate)
-> where(‘dt’,’<‘,$endDate)
->groupBy(‘no’)
->havingRaw(` SUM(price) >= ?’, [500000]);

#query builder에서 toSql() 로 sub query를 받아 ,mergeBindings를 이용해  
$orders = DB::table( DB::row( ‘({$subQuery->toSql()}) as a’)
->mergeBindings( $subQuery->toBase() )
->count()
```

위와 같이 Eloquent로 작성한 쿼리를 아래에서 쿼리 빌더로 가져와 둘의 쿼리를 조합 할 수 있도록 해줍니다. 


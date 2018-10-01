---
layout: post
title:  "함수형 프로그래밍이란 무엇인가?"
subtitle:   "What is Functional Programming?"
categories: dev
tags: scala
comments: true
---
```
[스칼라로 배우는 함수형 프로그래밍] 책을 읽고 정리한 내용입니다
```


##### 함수형 프로그래밍이란 무엇인가?

<u>프로그램을 "순수함수"들로만, 다시 말해서 부수효과가 없는 함수들로만 작성한다는것</u>이다.<br>
부수효과가 있는 함수라는 것은, 수행하는 기능에 해당하는 결과를 돌려주는것 이외에 다른 일도 수행하는 함수를 말한다.

> **부수효과의 예**<br>
> - 변수 또는 자료구조를 수정한다<br>
> - 객체의 필드를 설정한다<br>
> - 예외를 던지거나 오류를 내면서 실행을 중단한다<br>
> - 콘솔에 출력하거나 사용자의 입력을 읽는다<br>
> - 파일에 기록하거나 파일에서 읽어들인다<br>
> - 화면에 뭔가를 그린다

> **순수함수들로 프로그램을 작성하면** <br>
> - "모듈성"이 증가 : 테스트, 재사용, 병렬화, 일반화, 분석이 쉽다. <br>
> - 버그가 생길 여지가 훨씬 적다.


---


예를 하나 보자.

```scala
class Cafe {
  def buyCoffee(cc: CreditCard): Coffee = {
    val cup = new Coffee()
    cc.charge(cup.price)
    cup
  }
}
```
```
카페에서 커피를 사겠다. 신용카드를 들고 갈거고, 목적은 커피 한컵을 얻는것.
커피를 컵에 담는다.
신용카드로 내 커피의 가격만큼 요금을 청구하고
컵을 준다
```


순수한 함수인지 살펴보자.<br>
목적은 커피 한컵을 얻는것인데, 단순히 그 일만 하지 않고 다른 일도 한다.<br>
그 다른 일, 즉 요금을 청구하는 일이 부수효과이다.<br>
부수효과가 섞여있어서 순수하다고 할수 없다. <br>
이 상태에서는 커피 한컵을 주는 코드가 잘 동작하는지만 테스트하기가 어렵다. 신용카드 회사에 연결해서 카드 대금을 청구해야 하기 때문이다.<br>
카드로 요금 계산하는 일은 여기서 하지 말자. 다른 곳으로 빼자.<br>

또 객체지향적인 시각으로 보면, CreditCard 에 실제 신용카드 회사와 결제 연동하는 방법이나, 결제한 내역을 로그로 남기는 방법 등을 집어 넣는것은 적합하지 않다.<br>
Payment 객체를 따로 만들어 처리하는게 좋겠다.

또한 커피 12잔을 주문한다고 했을때, 신용카드 청구가 12번이나 된다는 문제도 있다<br>
청구 대금을 누적하는 buyCoffees 라는 함수를 작성해보자. 그 안에서 buyCoffee를 재사용하자.<br>
그러기 위해서 buyCoffee 함수에서는 Coffee와 함께 Charge 청구건을 돌려주게 하자.


--- 


변경한 코드는 다음과 같다.

```scala
class Cafe {
  def buyCoffee(cc: CreditCard): (Coffee, Charge) = {
    val cup = new Coffee()
    (cup, Charge(cc, cup.price))
  }
  
  def buyCoffees(cc: CreditCard, n: Int): (List[Coffee], Charge) = {
    val purchases: List[(Coffee, Charge)] = List.fill(n)(buyCoffee(cc))    
    val (coffees, charges) = 
    purchases.unzip(coffees, charges.reduce((c1, c2) => c1.combine(c2)))
  }
}
```
```
카페에서 커피를 사겠다. 신용카드를 들고 갈거고, 목적은 커피 한컵과 그에 따른 청구서를 받는것.
커피를 컵에 담는다.
커피가 담긴 컵과, 들고간 신용카드로 내 커피의 가격만큼 지불한다는 내용의 청구서를 준다.

카페에서 커피들을 사겠다. 신용카드를 들고 n개 사겠다 말한다, 목적은 커피들과 청구서 하나를 받는것.
커피를 사는 함수의 결과를 n개 복사한다. 
[(커피1,청구서1)(커피2,청구서2)..(커피n,청구서n)]
unzip으로 커피들을 하나의 리스트로 모으고, 청구서를 결합하여 청구서 하나로 만든다. 
([커피1,커피2,..커피n],결합청구서)
```

List.fill(n)(x) : x의 복사본 n개로 이루어진 List를 생성
unzip : 쌍들의 목록 => 목록들의 쌍
reduce : 청구건 두개 c1, c2를 결합하는 과정을 반복적으로 수행 => 하나의 청구건을 반환


```scala
case class Charge(cc: CreditCard, amount: Double) {
  def combine(other: Charge): Charge = 
    if(cc == other.cc)
      Charge(cc, amount + other.amount)
    else 
      throw new Exception("Can't combine charges to different cards")
}
```
```
청구서는 어떤 신용카드로 얼마를 지불할 것인지가 쓰여있다.
청구서를 합치는 함수(combine)
  청구서가 다른청구서(other)의 신용카드와 
  같은 신용카드라면, 지불할 금액을 합쳐서 하나의 청구서를 만들어준다.
  다른 신용카드라면 합칠수 없다고 알려준다.
```


---


여기서 Charge를 일급(first-class)값으로 만들면, 같은 카드에 대한 청구건들을 하나의 List[Charge]로 취합하는 함수를 작성할수 있다.<br>

[일급값이란?](http://blog.doortts.com/m/135) (출처:http://blog.doortts.com/m/135) <br>
함수가 일급 객체가 되려면 함수 자체를 파라미터로 넘기고, 결과값으로 함수가 오는 것이 가능해야 한다. 
```
어떤 객체(일반명사)가 있을때 다음 조건을 만족하면 일급 객체로 간주한다.

1. 변수나 데이터 구조안에 담을 수 있다.
2. 파라미터로 전달 할 수 있다.
3. 반환값(return value)으로 사용할 수 있다.
4. 할당에 사용된 이름과 관계없이 고유한 구별이 가능하다.
```


같은 카드에 대한 청구건들을 취합하는 함수는 다음과 같다.

```scala
def coalesce(charges: List[Charge]): List[Charge] = 
  charges.groupBy(_.cc).values.map(_.reduce(_ combine _)).toList
```
```
같은 카드끼리 청구서를 합쳐보자. 청구서 리스트를 받을 것이고, 목적은 카드별로 청구서를 합쳐 받는것.
청구서를 
  카드로 그룹화 한다(groupBy) 
  반복적으로 돌면서(map) 
  청구서 두개를 결합하여(combine) 하나로 합친다(reduce)
  카드별 청구서를 리스트로 만든다(toList)
```

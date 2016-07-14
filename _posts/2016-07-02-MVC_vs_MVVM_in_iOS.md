---
layout: post
title:  "MVC vs MVVM in iOS"
date:   2016-06-23 10:49:00
categories: iOS
---

#### MVC

iOS앱의 구조를 설계할 때에는 다음 구조처럼 ViewController가 Model 및 View를 소유하고 View에 Model의 데이터를 나타내기 위한 중재자로서 ViewController가 존재하는 형태이다. 

![CocoaMVC](https://www.objc.io/images/issue-13/mvvm1-16d81619.png)


실제 프로젝트를 진행하다보면 ViewController가 Model의 변경을 이미 알고 있기 때문에 자신이 직접 Model을 변경한 뒤에 View를 업데이트하도록 개발한다. 결국 (애플이 의도했던 형태의 MVC 구현은) 아래와 같이 '서로 강결합된 초대형 View-ViewController 덩어리'를 양산하게 된다. 게다가 ViewController는 Model-Controller 사이에서 다뤄야할 비즈니스로직들 사이에 View 코드가 껴들기 쉽고, 이로 인해 유닛테스트 코드를 만들기에도 까다로운 문제가 함께 발생한다.


#### MVVM

위에서 언급한 문제를 해결하기 위해 MVVM 패턴이 대안이 될 수 있다. MVVM 패턴을 iOS앱에 적용시켜보면 다음과 같다.

![MVVM in iOS](https://www.objc.io/images/issue-13/mvvm-b27768df.png)


위 도식에서 보면 ViewModel이라는 새로운 Model이 나타나는데 쉽게 설명하면 **View를 위한 Model**이다. ViewModel은 다음과 같은 역할을 담당한다.

* Model을 소유하거나 갱신함.
* View에 데이터와 사용자 액션을 바인딩함.


이를 적용하면 다음과 같은 장점이 있다.

* View-ViewController 코드 덩어리의 크기가 작아지며, 각 모듈별로 역할/책임이 명확해진다.
* **사용자 액션-모델 업데이트-뷰 갱신 흐름이 명확하다.**
* MVVM은 기존의 MVC 코드를 크게 수정하지 않고도 사용할 수 있다.
* 모델을 조작하는 코드가 보다 테스트에 용이해진다.

보통은 MVVM 구현을 위해 [ReactiveCocoa](https://github.com/ReactiveCocoa/ReactiveCocoa) 또는 [RxSwift](https://github.com/ReactiveX/RxSwift)를 많이 이용하는 편이지만, 이를 이용하지 않고도 KVO 등 언어와 프레임워크의 기본기능만으로도 어느정도 MVVM 모양새는 구현할 수 있다. (이들 라이브러리를 쓰면 디버깅 시 스택 트레이스를 보기 어려워진다는 문제도 있다.) 나 같은 경우는 사용자 액션을 모니터링하거나 정의하기 위해 [BlocksKit](https://github.com/zwaldowski/BlocksKit)을 유용하게 사용했다.
# [CH 8-4] R-Tree 인덱스

아마도 MySQL의 공간 인덱스(Spatial Index)라는 말을 한 번쯤 들어본 적이 있을 것이다. 공간 인덱스는 R-Tree 인덱스 알고리즘을 이용해 2차원의 데이터를 인덱싱하고 검색하는 목적의 인덱스다.
기본적인 내부 메커니즘은 B-Tree와 흡사하다. B-Tree는 인덱스를 구성하는 칼럼의 값이 1차원의 스칼라 값인 반면, R-Tree 인덱스는 2차원의 공간 개념 값이라는 것이다.

최근 GPS나 지도 서비스를 내장하는 스마트 폰이 대중화되면서 SNS 서비스가 GIS와 GPS에 기반을 둔 서비스로 확장되고 있다.
이러한 위치 기반의 서비스를 구현하는 방법은 여러 가지가 있겠지만 MySQL의 공간 확장(Spatial Extension)을 이용하면 간단하게 이러한 기능을 구현할 수 있다.
MySQL의 공간 확장에는 다음과 같이 크게 세 가지 기능이 포함돼 있다.

- 공간 데이터를 저장할 수 있는 데이터 타입
- 공간 데이터의 검색을 위한 공간 인덱스(R-Tree 알고리즘)
- 공간 데이터의 연산 함수(거리 또는 포함 관계의 처리)

이번 절에서는 공간 인덱스를 이해하는 데 필요한 기본적인 내용과 R-Tree 알고리즘을 살펴보겠다.

---
<br/>

## (1) 구조 및 특성
MySQL은 공간 정보의 저장 및 검색을 위해 여러 가지 기하학적 도형(Geometry) 정보를 관리할 수 있는 데이터 타입을 제공한다. 대표적으로 MySQL에서 지원하는 데이터 타입은 아래와 같다.

#### [그림 8.19] GEOMETRY 데이터 타입
<img src="https://github.com/user-attachments/assets/e6f816e1-d37f-4123-ac32-dfd252a2e8e3" width="380"/><br/>

마지막에 있는 GEOMETRY 타입은 나머지 3개 타입의 슈퍼 타입으로, POINT와 LINE, POLIGON 객체를 모두 저장할 수 있다.

공간 정보의 검색을 위한 R-Tree 알고리즘을 이해하려면 MBR이라는 개념을 알고 있어야 한다.
아래 그림은 위에서 예시로 든 도형들의 MBR을 보여주는데, MBR이란 "Minimum Bounding Rectangle"의 약자로 해당 도형을 감싸는 최소 크기의 사각형을 의미한다.
이 사각형들의 포함 관계를 B-Tree 형태로 구현한 인덱스가 R-Tree 인덱스다.

#### [그림 8.20] 최소 경계 상자(MBR, Minimum Bounding Rectangle)
<img src="https://github.com/user-attachments/assets/c2f2b5d9-a50e-4bb8-802a-7b76abb6bdb8" width="400"/><br/>

간단히 R-Tree의 구조를 살펴보자. 아래와 같은 도형(공간 데이터)이 있다고 해보자.

#### [그림 8.21] 공간(Spatial) 데이터
<img src="https://github.com/user-attachments/assets/5d7f8f1f-8784-4c46-927d-f18c4cd8597c" width="350"/><br/>

여기에는 표시되지 않았지만 단순히 X좌표와 Y좌표만 있는 포인트 데이터 또한 하나의 도형 객체가 될 수 있다.
이러한 도형이 저장됐을 때 만들어지는 인덱스의 구조를 이해하려면 우선 이 도형들의 MBR이 어떻게 되는지 알아볼 필요가 있다. 아래 그림은 이 도형들의 MBR을 3개의 레벨로 나눠서 그려본 것이다.

- 최상위 레벨: R1, R2
- 차상위 레벨: R3, R4, R5, R6
- 최하위 레벨: R7 \~ R14

#### [그림 8.22] 공간(Spatial) 데이터의 MBR
<img src="https://github.com/user-attachments/assets/196458e3-e900-4c92-a3d1-54b35022fb86" width="380"/><br/>

최하위 레벨의 MBR(각 도형을 제일 안쪽에서 둘러싼 점선 상자)은 각 도형 데이터의 MBR을 의미한다. 그리고 차상위 레벨의 MBR은 중간 크기의 MBR(도형 객체의 그룹)이다.
위 그림의 경우 최상위 MBR은 R-Tree의 루트 노드에 저장되는 정보이며, 차상위 그룹 MBR은 R-Tree의 브랜치 노드가 된다.
마지막으로 각 도형의 객체는 리프 노드에 저장되므로 아래와 같이 R-Tree 인덱스의 내부를 표현할 수 있다.

#### [그림 8.23] 공간(R-Tree, Spatial) 인덱스 구조
<img src="https://github.com/user-attachments/assets/b7530aa4-7d45-4320-9f89-d4a3c42b57ce" width="530"/><br/>
<br/>

## (2) R-Tree 인덱스의 용도
R-Tree는 앞에서 언급한 MBR 정보를 이용해 B-Tree 형태로 인덱스를 구축하므로 Rectangle의 'R'과 B-Tree의 'Tree'를 섞어서 R-Tree라는 이름이 붙여졌으며, 공간(Spatial) 인덱스라고도 한다.
일반적으로는 WGS84(GPS) 기준의 위도, 경도 좌표 저장에 주로 사용된다.
하지만 위도, 경도 좌표뿐 아니라 CAD/CAM 소프트웨어 또는 회로 디자인 등과 같이 좌표 시스템에 기반을 둔 정보에 대해서는 모두 적용할 수 있다.

R-Tree는 각 도형(더 정확히는 도형의 MBR)의 포함 관계를 이용해 만들어진 인덱스다.
따라서 ST_Contains() 또는 ST_Within() 등과 같은 포함 관계를 비교하는 함수로 검색을 수행하는 경우에만 인덱스를 이용할 수 있다.
대표적으로는 '현재 사용자의 위치로부터 반경 5km 이내의 음식점 검색' 등과 같은 검색에 사용할 수 있다.
현재 출시되는 버전의 MySQL에서는 거리를 비교하는 ST_Distance()와 ST_Distance_Sphere() 함수는 공간 인덱스를 효율적으로 사용하지 못하기 때문에 공간 인덱스를 사용할 수 있는 ST_Contains()
또는 ST_Within()을 이용해 거리 기반의 검색을 해야 한다.

#### [그림 8.24] 특정 지점을 기준으로 사각 박스 이내의 위치를 검색
<img src="https://github.com/user-attachments/assets/6f4037f0-4b3c-4efd-80e4-95842076ff49" width="330"/><br/>

위 그림에서 가운데 위치한 'P'가 기준점이다.
기준점으로부터 반경 5km 이내의 점(위치)들을 검색하려면 우선 사각 점선의 상자에 포함되는(ST_Contains() 또는 ST_Within() 함수 이용) 점들을 검색하면 된다.
여기서 ST_Contains()나 ST_Within() 연산은 사각형 박스와 같은 다각형(Polygon)으로만 연산할 수 있으므로 반경 5km를 그리는 원을 포함하는 최소 사각형(MBR)으로 포함 관계 비교를 수행한다.
점 'P6'은 기준점 P로부터 반경 5km 이상 떨어져 있지만 최소 사각형 내에는 포함된다. P6을 빼고 결과를 조회하려면 조금 더 복잡한 비교가 필요하다.
P6을 결과에 포함해도 무방하다면 다음 쿼리와 같이 ST_Contains()나 ST_Within() 비교만 수행하는 것이 좋다.

```
-- // ST_Contains() 또는 ST_Within()을 이용해 "사각 상자"에 포함된 좌표 Px만 검색

mysql> SELECT * FROM tb_location
       WHERE ST_Contains(사각 상자, px);

mysql> SELECT * FROM tb_location
       WHERE ST_Within(px, 사각 상자);
```

ST_Contains() 함수와 ST_Within() 함수는 거의 동일한 비교를 수행하지만 두 함수의 파라미터는 반대로 사용해야 한다.
ST_Contains() 함수는 첫 번째 파라미터로 포함 경계를 가진 도형을 명시하고 두 번째 파라미터로 포함되는 도형(또는 점 좌표)을 명시해야 한다.
하지만 ST_Within() 함수는 첫 번째 파라미터로 포함되는 도형(또는 점 좌표)을 명시하고 두 번째 파라미터로 포함 경계를 가진 도형을 명시해야 한다.

P6을 반드시 제거해야 한다면 다음과 같이 ST_Contains() 비교의 결과에 대해 ST_Distance_Sphere() 함수를 이용해 다시 한번 필터링해야 한다.

```
mysql> SELECT * FROM tb_location
       WHERE ST_Contains(사각상자, px) -- // 공간 좌표 Px가 사각 상자에 포함되는지 비교
             AND ST_Distance_Sphere(p, px)<=5*1000 /* 5km */;
```
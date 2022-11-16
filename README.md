# 블록체인 프로그래밍 프로젝트
-----
### 목적 

어떤 물품의 소유권 이전 현황을 블록체인에 기록 하여 특정 물품의 매매시 현재의 소유자, 전체 소유권 이전 히스토리, 실거래 가격 변동 추이 등
신뢰할 수 잇는 정보를 거래 당사자 양측이 확인 할 수 있는 시스템을 구현

### P2P 네트워크

myBlockChain을 위한 P2P 네트워크는 다수의 full 노드들과 다수의 사용자 노드들로 구성된다.
P2P 네트워크의 위상(topology)은 파일 topology.dat에 기술된 것처럼 정해진다. 
파일의 예는 다음과 같으며, 여기서 Fi는 full 노드, Ui는 사용자 노드, Ui-Fj는 Ui와 Fj사이의 링크, Fi-Fj는 Fi와 Fj 사 이의 링크를 의미한다.

```
    % cat topology.dat
    {
    node F0, F1, F2, F3, F4, F5
    node U0, U1, U2, U3
    link U0-F1, U1-F2, U2-F4, U3-F0
    link F0-F1, F2-F1, F2-F4, F3-F4, F4-F0, F4-F1, F3-F1, F2-F5, F4-F5 }
    %
```
<img  width="600" src="https://user-images.githubusercontent.com/82711279/202229550-0bd835e1-310b-424a-a6f3-c5406775faab.jpg" >

각 full 노드는 수신한 트랜잭션을 검증하는데, 검증 성공(valid라 판단)하면 저 장한 후, 이를 이웃 노드(들)에 보내고, 각 이웃 노드는 이를 다시 이웃(들)에 전파 하는 식으로 전체 네트워크에 전파되게 한다.
Full 노드는 자신이 판단하는 longest chain의 마지막 블록에 연결할 블록을 채굴하는데, 이 과정에서 채굴에 먼저 성공하기 위해 다른 full 노드들과 경쟁한다. 
Full 노드는 채굴 성공 시, 채굴 블록을 즉 시 이웃 노드(들)에 전달해 이 블록이 전체 네트워크로 전파되게 한다.

### 트랜잭션 구성 및 발생

이 서비스에서 필요한 트랜잭션은 단 한 가지 종류로 판매자(seller)가 구매자(buyer)에게 물품의 소유권을 넘기는 거래 내용을 기록한 것이다. 
트랜잭션은 trID: <input, output, identifier, modelNo, manufactured date, price, trading date, others>로 구성되는데, input은 특정 물품의 판매자(현재 해당 물건의 소유자)의 public key이며, output은 이 물품 구매자의 public key이다. 또한 identifier(특정 물품을 지칭하는 ID(예를 들어, 물품에 부착된 bar-code나 QR- code 통해 얻는 값)), modelNo(물품의 모델명), manufactured date(물품의 제조일)은 이 물품의 모 든 과거 및 현재와 미래의 판매 트랜잭션들에서 값이 변할 수 없는 immutable이며, price(판매된 가격), trading date(거래일)와 others는 mutable이다. others 필드는 이 물품에 대한 설명으로 어떤 내용이든 쓸 수 있다.
trID는 트랜잭션의 ID로서 트랜잭션 전체(서명 제외)에 대한 해시 함수 적용 결과이다.
트랜잭션은 판매자와 구매자의 확인 후, 판매자의 트랜잭션 전체에 대한 서명이 추가된 상태로 P2P 네트워크에 전파된다. 이 서명은 판매자의 public key로 검증할 수 있다.

### 블록 구조

블록은 헤더와 나머지 부분으로 구성되는데, 헤더에는 blockNo(myBlockChain 상의 블록 순서로서 genesis block은 blockNo로 0, 다음 블록은 1, ... 의 No를 가짐), prevHash(myBlockChain 상 직전 블록에 대한 hash pointer), nonce(hash puzzle 풀 때, 이를 변경시키며 target number보다 블록 hash 결과가 작아질 때까 지 시도함) 및 Merkle-root(트랜잭션들을 leaf로 가지는 Merkle-tree의 root에 해당하는 hash 값)으로 구성되며, 나머지 부분은 이 블록에 포함될 트랜잭션들을 leaf 들로 하는 Merkle-tree이다(다만 Bitcoin과는 다르게 coinbase 트랜잭션은 없음).

### 채굴 및 mhBlockChain 형성

각 full 노드는 채굴을 위해 블록에 포함될 트랜잭션들을 모두 검증하는데, 검증을 위한 과정은 다음과 같다.
트랜잭션 T에 대하여, (1) T를 통해 판매하려는 물품의 최종 소유자(즉 합의된 마지막 판매의 구매자)가 T의 input(즉 판매자)과 같은 지, 
(2) T의 immutable 필드들 값이 합의된 마지막 판매의 그것들과 일치하는지, 
(3) 서명이 T의 input 주소인 public key로 검증한 결과 T 전체(서명 제외한)의 서명이 맞는지, 확인한다. 그리고 검증을 통과한 트랜잭션들을 선택해 Merkle-tree를 구성하고, 블록 헤더의 nonce 값을 차례로 변경하면서 target number보다 작아질 때까지 반복한다. 
채굴에 성공한 full 노드는 채굴한 블록을 즉시 P2P 네트워크의 다른 노드들에게 전파한다(사용자 노드는 도착한 채굴 블록을 무시하도록 하거나, 아예 Fj->Ui방향의 통신을 차단함). 채굴된 블록을 수신한 full 노드는 이 블록을 검증한 후, 이 블록을 반영한 갱신된 myBlockChain에 기반하여 새로운 블록의 채굴을 시도한다.
Bitcoin과 달리, 이러한 채굴 과정의 난도(difficulty)는 불변이라 가정하고 target number 역시 고정된 값을 사용한다. 이때 target number는 블록 채굴에 소요되는 평균 시간이 (10초~15초) 정도가 되도록 자유롭게 정한다.
myBlockChain은 Bitcoin의 longest chain rule을 따라 각 full 노드가 현재 합의된 chain이 무엇인지를 독자적으로 판단하고, 이를 바탕으로 다음 채굴될 블록을 어느 블록에 연결할 것인지 결정하게 한다. 따라서 Bitcoin 블록체인과 마찬가지로 일정 기간 full 노드 사이의 합의된 체인에 대한 의견 불일치가 있을 수 있지만, 궁극적 으로 모든 full 노드들 사이에 합의된, 가장 긴 myBlockChain이 존재할 것이다.

### 마스터 process를 통한 동작 확인

구현된 프로그램이 요구 사항을 모두 만족시키는지 확인하기 위해, 질의를 받고 그에 대한 응답을 구하여 보여주는 역할을 하는 마스터 process를 생성한다. 
이 process는 myBlockChain 및 모든 full 노드들에 접근하여 필요한 데이터를 추출하 거나 요청하여 받을 수 있다. Full 노드를 구현한 process는 마스터 process의 데 이터 요청에 대하여 즉각적으로 응하여 해당 데이터를 제공해야 한다. 마스터 process를 통해 확인할 수 있는 동작의 종류는 다음과 같다.

(1) snapshot myBlockChain ALL(또는 특정 Fi)
ALL이 지정되면 현재 시점의 각 full 노드가 판단하는 myBlockChain을, 특정 Fi
가 지정되면 해당 Fi가 판단하는 myBlockChain을, 각각 다음과 같이 출력한다.

<img width="600" src="https://user-images.githubusercontent.com/82711279/202236872-ab310224-f0c4-4181-8e64-95abaef418f6.png">

(2) snapshot trPool \<Fi>
Full 노드 Fi가 현재 유지하고 있는 트랜잭션 풀(채굴 시 블록에 포함될 수 있는 트랜잭션들의 집합)의 내용을 다음과 같이 출력한다.

<img width="600" src="https://user-images.githubusercontent.com/82711279/202237410-f611d546-40dd-4c69-b16d-e63c837ec0a3.png">

(3) verifyLastTr \<Fi>
Full 노드 Fi가 가장 최근의 블록 채굴 시 포함한 마지막 트랜잭션의 검증 결과
(즉, Fi가 가장 최근 시도한 채굴 시(성공 여부 무관) 사용한 블록에 마지막으로 포함된(Merkle tree의 rightmost leaf에 해당하는) 트랜잭션에 대한 검증 내용 및 결과)를 다음과 같이 출력한다.

<img width="600" src="https://user-images.githubusercontent.com/82711279/202238241-77d18557-1562-4898-a2f5-07d864927bce.png">
<img width="600" src="https://user-images.githubusercontent.com/82711279/202238301-2c684fa4-0955-4875-8b38-ba4baaa4c338.png">

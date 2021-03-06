---
title: イベントソーシングの概要
date: "2020-10-31"
---

・6.1.2 イベントソーシングの概要
イベントソーシングは、イベントを中心に据えてビジネスロジックを実装し、アグリゲートを永続化する手法。
アグリゲートは、一連のイベントでデータベースに格納される。
個々のイベントはアグリゲートの状態変更を表す。
アグリゲートのビジネスロジックは、これらのイベントを生成、消費するための要件を中心として構成されます。
この仕組みの説明


イベントソーシングは、イベントを使ってアグリゲートを永続化する

従来の永続化) 
アグリゲートをテーブルに、フィールドを列に、インスタンスを行にマッピングしていた。

イベントソーシング ) 
ドメインイベントの概念を基礎として、それとは全く異なる形でアグリゲートを永続化していて、それをイベントストアと呼ぶ。

例) 
Orderテーブルに個々のOrderを格納するのではなく、
イベントソーシングは、EVENTSテーブルの1つ以上の行というかたちでOrderアグリゲートを永続化する。
アプリケーションは、
アグリゲートを作成・更新した時にアグリゲートが生成したイベントをEVENTSテーブルに入れる。
アグリゲートをロードするときは、データベースから読み取った一連のイベントを順番に適用します。
 1 ) アグリゲートのイベントをロードする
 2 ) デフォルトコンストラクタでアグリゲートのインスタンを作る
 3 ) apply()を呼び出して、各イベントの適用を繰り返す
詳細は図6.2を参照

処理 ) アグリゲートクラスのインスタンスを作る。
アグリゲートのapplyEvent()メソッドでイベントを反復的に適用してアグリゲートを復元する。(fold、reduceといった畳み込み操作)

ORMフレームワークは、1つ以上のSELECT文を実行してオブジェクトをロードし、現在の永続化状態を読み取り、
デフォルトコンストラクタでオブジェクトのインスタンスを作りリフレクションでインスタンスを初期化します。
イベントソーシングでは、インメモリ状態の再構築のためにイベントを使うところが異なるだけ

・イベントは状態の変化を表す

イベントは、アグリゲートIDのような最小限のデータを持つだけにすることも、イベントエンリッチメントに基づき、
多くのコンシューマで役に立つデータを含めることもできる。
例えば、オーダーサービスは、オーダーが作られた時にOrderCreatedイベントを生成することができます。
イベントをパブリッシュするかどうか、そのイベントにどのような情報を入れるかは、コンシューマのニーズによって決まります。
イベントソーシングでは、イベントとその構造を決めるのは、主としてアグリゲートになります。

イベントソーシングを使うと、イベントはオプションではなくなります。
アグリゲートの状態のすべての変化は、ドメインイベントによって表現されます。
アグリゲートの状態が変化した時には、必ずイベントを生成しなければならない。
イベントはアグリゲートが状態遷移するために必要なデータを含んでなければならない。
アグリゲートの状態は、アグリゲートオブジェクトのフィールドの値から構成される。



・アグリゲートのメソッドはイベントに関するすべての情報を提供する
ビジネスロジックは、アグリゲートルートのコマンドメソッドを呼び出してアグリゲート更新リクエストを起動する。
従来 : 
コマンドメソッドは一般に引数をチェックしてから、アグリゲートの1つ以上のフィールドを更新する。
イベントソーシングは
イベントを生成するために実行される。
アグリゲートのコマンドメソッドを実行すると、状態の変更を表す一連のイベントが生成される。
これらのイベントをデータベースに永続化される一方で、アグリゲートの適用されてアグリゲートの状態を更新します。
イベントを生成し、それを適用するという要件のために、ビジネスロジックは機械的なものですが、全面的な書き換えが必要になる。
イベントソーシングは、コマンドメソッドを2つ以上のメソッドに書き換えるリファクタリングを伴う。

第一のメソッドは、引数としてはリクエストを表現するコマンドオブジェクトを取り、どのような状態変更が必要かを判断します。
そのために引数をチェックし、状態変更を表すイベントのリストを返します。
アグリゲートの状態は更新しません。
このメソッドは一般的にコマンドが実行不能な時に例外を投げる


・イベントソーシングベースのOrderアグリゲート
イベントソーシングバージョンのOrderアグリゲートは、イベントを生成します。
ビジネスロジックがイベントを生成するコマンドとそれらのイベントを適用して状態を更新するコードによって実装されている

アグリゲートのidがアグリゲートに保存されないこと。


Order アグリゲートのフィールドとインスタンスを生成するメソッド
Orderアグリゲートを修正するprocess、apply()メソッド

各メソッドは1個のprocessメソッドと1個以上のapply()メソッドに置き換えられる。
reviseOrderメソッドはprocess(ReviseOrder)とapply(OrderRevisionProposed)、confirmRevision()メソッドはprocessとapplyに置き換えられています。

public class Order { 
 private OrderState state;
 private Long consumerId;
 private Long restaurantId;
 private OrderLineItems orderLineItems;
 private DeliveryINfomation deliveryInformation;
 private PaymentInformation paymentInformation;
 private Money orderMinumum;

 public Order(){}

 public List<Event> process(CreateOrderCommand command) {
 	return events(new OrderCreatedEvent(command.getOrderDetails()));
 }

 public void apply(OrderCreatedEvent event){
   OrderDetails orderDetails = event.getOrderDetails();
   this.orderLineItems = new OrderLineItems(orderDetails.getLineItems());
   this.orderMinumum = orderDetails.getOrderMinimum();
   this.state = APPROVAL_PENDING;
 }

}

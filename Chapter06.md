# Chapter 06

* 各マイクロサービスが独自に持つテータストア (RDB/NoSQL)にデータを永続化させる
* データ登録・更新・削除のためのAPI追加と、参照系の既存APIもデータストアからデータを取得するようにする

1. Adding a persistence layer to the core microservices
2. Writing automated tests that focus on persistence
3. Using the persistence layer in the service layer
4. Extending the composite service API
5. Adding databases to the Docker Compose landscape
6. Manual testing of the new APIs and the persistence layer
7. Updating the automated tests of the microservice landscape

## はじめに：利用技術の説明

### Spring Data

* データストアに対するアクセスを抽象化するライブラリ
* RDBだけでなく各種データストアに対応（MongoDB, Redis, Neo4j等）

> Spring Data の使命は、基礎となるデータストアの特殊な特性を保持したまま、データアクセス用の使い慣れた一貫した Spring ベースのプログラミングモデルを提供することです。  
> https://spring.pleiades.io/projects/spring-data#overview  
core concept
![](Chapter%2006/spring_data_overview_small.jpg)core concept

* データアクセスのための各種インタフェースを提供する

### Spring Data JPA

* JPAは仕様
	* JPA(Java Persistence API)は、リレーショナルデータベースで管理されているレコードを、Javaオブジェクトにマッピングする方法と、
マッピングされたJavaオブジェクトに対して行われた操作を、リレーショナルデータベースのレコードに反映するための仕組みをJavaのAPI仕様として定義したものである。
* 実装はHibernateのようなJPAプロバイダによって提供される

### Mapstruct

> MapStruct is a code generator that greatly simplifies the implementation of mappings between Java bean types based on a convention over configuration approach. The generated mapping code uses plain method invocations and thus is fast, type-safe and easy to understand.  
> https://mapstruct.org/  

* Spring DataのエンティティオブジェクトとAPIモデルクラスの間のmappingを省力化するために導入

## Adding a persistence layer to the core microservices
Entity Classを追加する。

主要なアノテーション

|アノテーション|概要|
|---|---|
|@Id|DB上のpk（サロゲートキー）に対応
|@Version| 楽観ロック用
|@Table| RDBの場合、テーブルとのマッピング。デフォルトはクラス名と自動マッピングだが、明示的にマッピング対象の指定も可能
|@GeneratedValue|idの採番ロジックを指定。TABLE : 永続層(DB)のテーブルを使用してIDを採番する。SEQUENCE : 永続層(DB)のシーケンスオブジェクトを使用してIDを採番する。IDENTITY : 永続層(DB)のidentity列を使用してIDを採番する。AUTO : 永続層(DB)に最も適切な採番方法を選択してIDを採番する。※ MongoDBではサポートされていない
|@Document|MongoDB用のTableアノテーション的なもの

APIのEntityとの違いは id と version を持つ点。 id はエンティティの一意性を担保するキーで、DBのサロゲートキーに当たる。APIを通じて公開しない。
versionは楽観ロックのために利用される。更新しようとしているEntityクラスのインスタンスのversionが格納済みの値と異なる場合Spring Dataがこれを阻止する。 ビジネス上の一意キー（ナチュラルキー）には unique
indexを付与するアノテーションをつける。

実際のMongoDBの場合のEntityクラスのコードは以下の様になる 通常 JPAのEntityクラスには `@Entity` annotationを付与するが、 MongoDBでは `@Document` annotation
を付与する。

#### RDBの場合
[ReviewEntity.java](microservices/review-service/src/main/java/se/magnus/microservices/core/review/persistence/ReviewEntity.java)

#### MongoDBの場合
[ProductEntity.java](microservices/product-service/src/main/java/se/magnus/microservices/core/product/persistence/ProductEntity.java)

複合キーの場合は以下のように定義する

```java
@CompoundIndex(name = "prod-rec-id", unique = true, def = "{'productId': 1, 'recommendationId' : 1}")
```

## Repository

### 基本的な操作
Spring DataではRepositoryを作成する方法として3つの方法があるが、本書では 1. の方法にて実装を進める。

1. Spring Data提供のインタフェースを継承する
2. 必要なメソッドのみ定義したインタフェースを継承する(Repositoryインタフェースのメソッドの中から、必要なメソッドのみ定義したプロジェクト用の共通インタフェースを作成し、作成した共通インタフェースを継承する)
3. Spring Dataから提供されているインタフェースの継承は行わない（`@org.springframework.data.repository.RepositoryDefinition`を利用する）

継承に使えるインタフェースは以下のものがある（本書では使っていないが、JpaRepository を継承しておけばよいのではという気がする）

|interface|概要|
|---|---|
|org.springframework.data.repository.CrudRepository|汎用的なCRUD操作を行うメソッドを提供しているRepositoryインタフェース。
|org.springframework.data.repository.PagingAndSortingRepository|CrudRepository のfindAllメソッドにページネーション機能とソート機能を追加したRepositoryインタフェース。
|org.springframework.data.jpa.repository.JpaRepository|JPAの仕様に依存するメソッドを提供しているRepositoryインタフェース。PagingAndSortingRepository を継承しているため、 PagingAndSortingRepository および CrudRepository のメソッドも使用する事ができる。

例

```java
public interface ProductRepository extends PagingAndSortingRepository
        <ProductEntity, String> {
    Optional<ProductEntity> findByProductId(int productId);
}
```

```java
public interface RecommendationRepository extends CrudRepository
        <RecommendationEntity, String> {
    List<RecommendationEntity> findByProductId(int productId);
}
```

### 基本的な操作の拡張
インタフェースに定義されている汎用的なCRUD操作を行うためのインタフェースだけでは、実際のアプリケーションを構築する事は難しいため、これを拡張することができる。
Spring Dataは、以下の方法をサポートする。

1. 命名規約ベースのメソッド名で指定する （参照系クエリのみ）
2. @Query アノテーションで指定する
3. プロパティファイルにNamed queryとして指定する

Spring Dataは、メソッドのシグネチャの命名規則に基づいて、追加のクエリメソッドを定義することをサポートしている。 例えば以下の様に拡張できる

```java
public interface ReviewRepository extends CrudRepository<ReviewEntity, Integer> {
    @Transactional(readOnly = true)
    List<ReviewEntity> findByProductId(int productId);
}
```

## Writing automated tests that focus on persistence
永続層にフォーカスした自動テスト（単体テスト）を構成する。
永続化テストを記述する際には、テストの開始時にデータベースを起動し、テストの完了時にデータベースを削除したい、ただし不要なMW（Nettyなど）は起動したくない
こうした要求を叶えるクラスレベルのアノテーションとして以下が準備されている。これらを @SpringBootTest の代わりに利用する

* @DataMongoTest
	* テストの開始時にMongoDBデータベースを起動する
* @DataJpaTest
	* テストの開始時にSQLデータベースを起動する
	* デフォルトでは
	* デフォルトでは、他のテストへの悪影響を最小限にするために、SQLデータベースへの更新をロールバックするようにテストを設定します。
今回のケースでは、この動作によって一部のテストが失敗するため、クラスレベルのアノテーション@Transactional(propagation = NOT_SUPPORTED)で自動ロールバックを無効にしている

上記の仕組みの中に、コンテナ化されたミドルウェアを組み込むため、 Testcontainers を使う
@Testcontainers アノテーション と、@Container アノテーション

このアプローチを何も考えず採用すると、テストクラスごとに @Container で指定したコンテナの再作成が行われてしまう。 例えば、MySQLのCreate/Deleteで10数秒かかり × クラス数 それが発生することとなる。
対策として [Singleton Container Pattern](https://www.testcontainers.org/test_framework_integration/manual_lifecycle_control/#singleton-containers) を利用する。
これは共通の基底クラスに利用するコンテナを定義するもの。

* MySQL
	* 基底クラス：
[MySqlTestBase.java](microservices/review-service/src/test/java/se/magnus/microservices/core/review/MySqlTestBase.java)
	* テストクラス：
[PersistenceTests.java](microservices/review-service/src/test/java/se/magnus/microservices/core/review/PersistenceTests.java)

* MongoDB
	* 基底クラス
[MongoDbTestBase.java](microservices/product-service/src/test/java/se/magnus/microservices/core/product/MongoDbTestBase.java)
	* テストクラス
[PersistenceTests.java](microservices/product-service/src/test/java/se/magnus/microservices/core/product/PersistenceTests.java)

以下の様に、AutoConfigureTestDatabaseを無効化しておかないと組み込みのDB（H2）を起動しようとしてしまうので注意

```java
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
```

## Using the persistence layer in the service layer

1. (Logging the database connection URL)
2. Adding new APIs
3. Calling the persistence layer from the service layer
4. Declaring a Java bean mapper
5. (Updating the service tests)

### 2. Adding new APIs
以下の様に作成・削除のAPIを追加する。冪等にできるAPIは冪等になるようにすべし

[ProductService.java](api/src/main/java/se/magnus/api/core/product/ProductService.java)

```java
@PostMapping(
        value = "/product",
        consumes = "application/json",
        produces = "application/json")
Product createProduct(@RequestBody Product body);
@DeleteMapping(value = "/product/{productId}")
void deleteProduct(@PathVariable int productId);
```

### 3. Calling the persistence layer from the service layer
ServiceクラスにてConstructor InjectionでmapperクラスをDIする。 必要なメソッドを定義していく。

[ProductServiceImpl.java](microservices/product-service/src/main/java/se/magnus/microservices/core/product/services/ProductServiceImpl.java)

#### Create
ポイントは、 repository の save メソッドを呼び出し管理対象としている点 と、mapperを用いたAPIモデルクラスとエンティティクラスの間でのJava beansの詰め替えを行っている点

#### Read
特筆すべき点はないが、Optional が返るので、対象が存在しない場合 orElseThrow で例外を発生させる

#### Delete
一旦削除対象を検索した上で、存在していれば削除、存在していなければ何もしない（エラーにもしない）

### 4. Declaring a Java bean mapper
レイヤ間のデータ授受の際には、マッピングが必要となる
よくあるのはこんなコードだが、単調でつまらない、ミスりやすい、可読性悪い と言った辛みがある

```java
Product entityToApi(ProductEntity entity) {
    return Product
            .builder()
            .productId(entity.getProductId())
            .name(entity.getName())
            .weight(entity.getWeight())
            .build();
}
```

これを解消するため、Mappingを自動で行ってくれるライブラリを活用する。
本書では`Mapstruct`を使う。 類似のものには、`JMapper`,`Orika`,`ModelMapper`,`Dozer` 等がある。
Mapstructは、作成されたマッピングコードを目で直接確認できることやパフォーマンスの点で優れていると言われているそう。

参考：
* [Performance of Java Mapping Frameworks](https://www.baeldung.com/java-performance-mapping-frameworks)

本書ではインタフェースに２つのメソッド、entityToApi / apiToEntity を定義している。
自動的に同盟のフィールド同士がマッピングされるが、アノテーションで指定することで無視したり別名にマッピングしたりと言った指定もできる。

ビルド時に、ProductMapperインタフェースからProductMapperImplが生成される。

[ProductMapper.java](microservices/product-service/src/main/java/se/magnus/microservices/core/product/services/ProductMapper.java)

```java
@Mappings({
    @Mapping(target = "serviceAddress", ignore = true)
    Product entityToApi(ProductEntity entity);
})
```

[ProductMapperImpl.java](microservices/product-service/build/generated/sources/annotationProcessor/java/main/se/magnus/microservices/core/product/services/ProductMapperImpl.java)

### 5. (Updating the service tests)
（割愛：テストも更新する）

## Extending the composite service API
コンポジットAPIを拡張する

1. Adding new operations in the composite service API
2. Adding methods in the integration layer
3. Implementing the new composite API operations
4. Updating the composite service tests

### 1. Adding new operations in the composite service API
Controllerにメソッドを追加する。OpenAPI Specification のためのアノテーションも付与する。

### 2. Adding methods in the integration layer
単純にRestTemplateを用いて対応するAPIを呼び出す

[ProductCompositeIntegration.java](microservices/product-composite-service/src/main/java/se/magnus/microservices/composite/product/services/ProductCompositeIntegration.java)

### 3. Implementing the new composite API operations
コンポジットのcreateメソッドは、集約されたproductオブジェクトを product, recommendation, review の個別オブジェクトに分割し、対応するcreateメソッドを呼び出します。

コンポジットのdeleteメソッドは、単純に統合レイヤーの3つのdeleteメソッドを呼び出し、基礎となるデータベースの対応するエンティティを削除します。

[ProductCompositeServiceImpl.java](microservices/product-composite-service/src/main/java/se/magnus/microservices/composite/product/services/ProductCompositeServiceImpl.java)

注意：現状ではHappy path飲みを考慮した設計・実装となっている。Edge caseについては後の章で取り扱う

### 4. Updating the composite service tests
（割愛：もちろんテストも追加する）

## Adding databases to the Docker Compose landscape
（割愛：Docker composeにMongoとMySQLを追加する）

> Note: It is strongly recommended to set the spring.jpa.hibernate.ddl-auto property to none or validate in a production environment—this prevents Spring Data JPA from manipulating the structure of the SQL tables. For more information, see https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#howto-database-initialization .  

## Manual testing of the new APIs and the persistence layer
swagger-ui からテストする

## Updating the automated tests of the microservice landscape
自動テスト

### 個人的に思ったこと

* テストデータの準備・DBデータ照合を、開発中のメソッドにやらせるよりは DB Unit + Database Rider など使ったほうがよいと思う
* 自動テストスクリプトに docker-compose up/down が組み込まれているが、Gradleに統合したほうがスマートな気がする
	* 自動テストは現状素朴な感じなので今後の進化に期待
* （これは好き好きか）先にテストを書いたほうが気持ち良いかもしれない

## Appendix

* [Spring Data Commons - Reference Documentation](https://docs.spring.io/spring-data/data-commons/docs/current/reference/html/#reference)
* [6.3. データベースアクセス（JPA編） — TERASOLUNA Server Framework for Java (5.x) Development Guideline 5.7.0.RELEASE documentation](http://terasolunaorg.github.io/guideline/current/ja/ArchitectureInDetail/DataAccessDetail/DataAccessJpa.html)
* [Java ORマッパー選定のポイント #jsug](https://www.slideshare.net/masatoshitada7/java-or-jsug)

### デザインパターン

* Data Mapper
* Repository パターン

> A Repository mediates between the domain and data mapping layers, acting like an in-memory domain object collection. Client objects construct query specifications declaratively and submit them to Repository for satisfaction. Objects can be added to and removed from the Repository, as they can from a simple collection of objects, and the mapping code encapsulated by the Repository will carry out the appropriate operations behind the scenes. Conceptually, a Repository encapsulates the set of objects persisted in a data store and the operations performed over them, providing a more object-oriented view of the persistence layer. Repository also supports the objective of achieving a clean separation and one-way dependency between the domain and data mapping layers.  
> https://martinfowler.com/eaaCatalog/repository.html  
https://www.oreilly.com/library/view/spring-5-design/9781788299459/
https://www.oreilly.com/library/view/spring-5-design/9781788299459/f96c8c51-3ff2-4b82-870b-bd2c3ece2964.xhtml

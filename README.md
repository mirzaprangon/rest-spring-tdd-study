# rest-spring-tdd-study
Learn about spring library, spring boot and Test Driven Development process.

この repositoryは codespacesで開けます。Forkして自由にいじって下さい。
# `GET` api
## Initialization
[Spring initializr](https://start.spring.io/) あるいは VScodeでspring projectを作成します。オプションは以下の通り
- Project: Gradle - Groovy
- Language: Java
- SpringBoot: Latest 3.0X

そして、program的に以下の
- Group: example
- Artifact: cashcard
- Name: CashCard
- Description: CashCard service for Family Cash Cards
- Packaging: Jar
- Java: 17
Dependencies:
- Spring web

作成された projectを open. そして以下を実行。
```bash
./gradlew build
```
これで setup完了。

## API Contract
API contractとは、APIプロバイダーとAPI利用者の間の正式な合意であり、お互いのやり取り方法を明確に伝えるものです。このcontractは、APIプロバイダーとAPI利用者がどのようにやり取りするか、データ交換がどのように見えるか、成功と失敗の場合の通信方法を定義します。

私たちは作るAPIの contractは以下の通りに作ります。

```
Request
  URI: /cashcards/{id}
  HTTP Verb: GET
  Body: None

Response:
  HTTP Status:
    200 OK if the user is authorized and the Cash Card was successfully retrieved
    403 UNAUTHORIZED if the user is unauthenticated or unauthorized
    404 NOT FOUND if the user is authenticated and authorized but the Cash Card cannot be found
  Response Body Type: JSON
  Example Response Body:
    {
      "id": 99,
      "amount": 123.45
    }
```
**[JSONとは？](https://www.json.org/json-ja.html)**

## TDD
ソフトウェア開発チームが自動テストスイートを作成して、リグレッションに対処することは一般的です。多くの場合、これらのテストはアプリケーションコードの後に書かれます。代わりに、私たち別のアプローチを取ります。アプリケーションコードを実装する前にテストを書きます。これは Test Driven Developmenr（TDD）と呼ばれます。

なぜ TDD を使うのか？望ましい機能を実装する前期待される動作をテストすることで、プログラムがすでに行なっている動作ではなく、プログラムが行うべき動作をテストしそれに対応するプログラムを設計・製造します。

さらに、Red, Green, Refactor development loopを使います。

テストのため Junitを使います。テストコードを`src/test/java/<package>`ない書きます。早速始めましょう。まずは API contractのresponseの JSON形式のテストをします。

## Serialization and Deserialization: Writing first test
1. `src/test/java/example/cashcard` `CashCardJsonTest.java` fileの中以下のコードを書きます。
    ```java
    package example.cashcard;

    import org.junit.jupiter.api.Test;

    import static org.assertj.core.api.Assertions.assertThat;

    public class CashCardJsonTest {

    @Test
        public void myFirstTest() {
            assertThat(1).isEqualTo(42);
        }
    }
    ```
2. Testを実行
    ```bash
    ./gradlew test
    ```
3. コードを修正してもう一回 test実行しましょう。
    ```java
    assertThat(42).isEqualTo(42);
    ```
4. `CashCardJsonTest.java` の内容を以下にしてテストする。
    ```java
    import java.io.IOException;

    import static org.assertj.core.api.Assertions.assertThat;

    @JsonTest
    public class CashCardJsonTest {

        @Autowired
        private JacksonTester<CashCard> json;

        @Test
        public void cashCardSerializationTest() throws IOException {
            CashCard cashCard = new CashCard(99L, 123.45);
            assertThat(json.write(cashCard)).isStrictlyEqualToJson("expected.json");
            assertThat(json.write(cashCard)).hasJsonPathNumberValue("@.id");
            assertThat(json.write(cashCard)).extractingJsonPathNumberValue("@.id")
                    .isEqualTo(99);
            assertThat(json.write(cashCard)).hasJsonPathNumberValue("@.amount");
            assertThat(json.write(cashCard)).extractingJsonPathNumberValue("@.amount")
                 .isEqualTo(123.45);
        }
    }
    ```
    
5. `src/main/java/example/cashcard/CashCard.java` に CashCard classを作成する。Javaの recordを使います。
    ```java
    package example.cashcard;

    public record CashCard(Long id, Double amount) {
    }
    ```
6. `src/test/resources/example/cashcard/expected.json` ファイルを作る。中身は 以下`{}`。
```json
{
  "id": 99,
  "amount": 123.45
}
```
7. `src/test/java/example/cashcard/CashCardJsonTest.java` に以下のテストを追加してテストを実行。
    ```java
    @Test
    public void cashCardDeserializationTest() throws IOException {
       String expected = """
               {
                   "id":99,
                   "amount":123.45
               }
               """;
       assertThat(json.parse(expected))
               .isEqualTo(new CashCard(1000L, 67.89));
       assertThat(json.parseObject(expected).id()).isEqualTo(1000);
       assertThat(json.parseObject(expected).amount()).isEqualTo(67.89);
    }
    ```
8. Assertを修正してテスト
    ```java
    assertThat(json.parse(expected))
            .isEqualTo(new CashCard(99L, 123.45));
    assertThat(json.parseObject(expected).id()).isEqualTo(99);
    assertThat(json.parseObject(expected).amount()).isEqualTo(123.45);
    ```
これでAPI contractの JSON形式のテストしました。

## GET method
1. `src/test/java/example/cashcard/CashCardApplicationTests.java` を以下で updatesしてテストを実行。
    ```java
    package example.cashcard;

    import com.jayway.jsonpath.DocumentContext;
    import com.jayway.jsonpath.JsonPath;
    import org.junit.jupiter.api.Test;
    import org.springframework.beans.factory.annotation.Autowired;
    import org.springframework.boot.test.context.SpringBootTest;
    import org.springframework.boot.test.web.client.TestRestTemplate;
    import org.springframework.http.HttpStatus;
    import org.springframework.http.ResponseEntity;

    import static org.assertj.core.api.Assertions.assertThat;

    @SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
    class CashCardApplicationTests {
        @Autowired
        TestRestTemplate restTemplate;

        @Test
        void shouldReturnACashCardWhenDataIsSaved() {
            ResponseEntity<String> response = restTemplate.getForEntity("/cashcards/99", String.class);

            assertThat(response.getStatusCode()).isEqualTo(HttpStatus.OK);
        }
    }
    ```
    `@Autowired TestRestTemplate restTemplate;` で spring dependency injectionをしています。これで springが必要な classの objectを用意してくれます。
2. Controller classを作成してテストを実行。
```java
package example.cashcard;

import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

public class CashCardController {
   public ResponseEntity<String> findById() {
      return ResponseEntity.ok("{}");
   }
}
```
3. Controllerを以下で updateする。
    ```java
    @RestController
    @RequestMapping("/cashcards")
    public class CashCardController {

       @GetMapping("/{requestedId}")
       public ResponseEntity<String> findById() {
             return ResponseEntity.ok("{}");
       }
    }
    ```
    -  `@RestController`: Tells spring this is capable of handling HTTP request.
    -  `@RequestMapping("/cashcards")`: Which address request will this handle.
    -  `@GetMapping("/{requestedId}")`: Marks method as a handler method. The URI will be `/cashcards/{requestedId}`
4. TestをUpdateして実行。
    ```java
    assertThat(response.getStatusCode()).isEqualTo(HttpStatus.OK);

    DocumentContext documentContext = JsonPath.parse(response.getBody());
    Number id = documentContext.read("$.id");
    assertThat(id).isEqualTo(99);
    ```
    `DocumentContext documentContext = JsonPath.parse(response.getBody());` で response を json aware objectに変換。便利な色々 method使える。
5. Handler methodを update. CashCardを作成してそれを返しましょう。
    ```java
    @GetMapping("/{requestedId}")
    public ResponseEntity<CashCard> findById() {
       CashCard cashCard = new CashCard(1000L, 0.0);
       return ResponseEntity.ok(cashCard);
    }
    ```
6. Handler method `new CashCard(99L, 0.0);`. Tet pass! でも、amountはゼロ。。。
7. Testに amountを追加してテスト。
    ```java
    DocumentContext documentContext = JsonPath.parse(response.getBody());
    Number id = documentContext.read("$.id");
    assertThat(id).isEqualTo(99);

    Double amount = documentContext.read("$.amount");
    assertThat(amount).isEqualTo(123.45);
    ```
8. Methodをupdate. `new CashCard(99L, 123.45);`. Test pass!
9. 次 Path variableでテストしましょう。以下のテストを追加して実行。
    ```java
    @Test
    void shouldNotReturnACashCardWithAnUnknownId() {
      ResponseEntity<String> response = restTemplate.getForEntity("/cashcards/1000", String.class);

      assertThat(response.getStatusCode()).isEqualTo(HttpStatus.NOT_FOUND);
      assertThat(response.getBody()).isBlank();
    }
    ```
    `1000` ID のcashcardがないので 404返すはず。
10. Handler methodをupdate。
    ```java
    @GetMapping("/{requestedId}")
    public ResponseEntity<CashCard> findById(@PathVariable Long requestedId) {
        if (requestedId.equals(99L)) {
            CashCard cashCard = new CashCard(99L, 123.45);
            return ResponseEntity.ok(cashCard);
        } else {
            return ResponseEntity.notFound().build();
        }
    }
    ```
    - `@PathVariable`: Springが URIの `requestedId`名所と method の `requestedId` 名所が同じのでこの変数に入れてくれます。
    これで test成功！

## DB からデータを取る
今まで hard coded データしか返していない。でも、我々は実やりたいのは本当のDBからデータを返す。次 DB使うようにしましょう。

DBは embedded, in memory databaseを使います。Embededなので javaの dependencyになります。In memory DB H2を使います。Dependencyを使いするだけで springが auto cnfigureしてくれるので使えます。

今まで書いたコードを見ましょう。Handler methodを修正しなければならない。けれどテストコードはそのまま使える。

1. `build.gradle` に H2の dependencyを追加。
```
dependencies {
   implementation 'org.springframework.boot:spring-boot-starter-web'
   testImplementation 'org.springframework.boot:spring-boot-starter-test'

   // Add the two dependencies below
   implementation 'org.springframework.data:spring-data-jdbc'
   testImplementation 'com.h2database:h2'
}
```
H2 memoryと [Spring Data JDBC](https://spring.io/projects/spring-data-jdbc) を追加しました。`./gradlew test` を一回実行して

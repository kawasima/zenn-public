---
title: "ドメインモデル実装におけるトリレンマに対するSoutherの回答"
type: "tech" # tech: 技術記事 / idea: アイデア
emoji: "🦹🏼"
topics: ["Souther"]
published: true
---

[ドメインモデル貧血症はなぜ生まれるのか — Decision パターンで DDD トリレンマを解く](https://zenn.dev/tellernovel_inc/articles/0193eb68cabb6e) は [Domain model purity vs. domain model completeness (DDD Trilemma)](https://enterprisecraftsmanship.com/posts/domain-model-purity-completeness/)を非常に分かりやすく解説し、かつGoでこの循環を解消する方法として、domainの関数が「許可」「拒否」「まだ終わっていない、これを取ってきてほしい」の3種類を返すDecisionパターンを示している良い記事だと思いました。

@[card](https://zenn.dev/tellernovel_inc/articles/0193eb68cabb6e)

ただ、Decisionパターンはやはり実装上の課題をドメインモデルに持ち込んでしまうので、外部I/Oをdomainの外に置ける一方、決定木の途中状態まで型として表すため、通常の関数より設計要素が増えるのが何とかしたいところです。

## ドメインモデル実装におけるトリレンマ

ドメインモデルに実装におけるトリレンマとは、たいていの業務ドメインで「完全性」「純粋性」「性能」のすべてを同時に満たすのが難しい、という問題です。ここでは、それぞれを次の意味で使います。

- 完全性：業務上の判断がdomainの中に漏れなく書かれている
- 純粋性：外部状態やI/Oに依存せず、入力した値から結果が決まる
- 性能：判断に必要なデータだけを取得する

| 選択肢 | 守れるもの | 諦めるもの | なぜそうなるか |
| --- | --- | --- | --- |
| A. domain が repository を呼ぶ | 完全性・性能 | 純粋性 | 判断は domain に残り、必要なデータだけ取得できる。ただし、判断の途中で外部の状態を取得するため、同じ入力だけから結果が決まる処理ではなくなる |
| B. 先に全部取ってから domain に渡す | 完全性・純粋性 | 性能 | 判断は純粋なまま domain に残せる。ただし、結局使わないデータも先に取得することになる |
| C. 呼び出し側で分岐する | 純粋性・性能 | 完全性 | 必要なデータだけ取得でき、domain も純粋に保てる。ただし、肝心の業務ルールが domain の外へ漏れる |

## Southerにおけるトリレンマの扱い

[Souther](https://souther-lang.org/)は、ドメインモデルを記述することに特化したプログラミング言語です。

概要と基本構文については、[Souther - ドメインモデルをスラスラ書けることを追い求めた最果てのJVM言語](https://zenn.dev/kawasima/articles/souther-essentials)を参照してください。

@[card](https://zenn.dev/kawasima/articles/souther-essentials)

引用記事のドメインモデルをSoutherで書き起こすと次のようになります。

```elm
// 小説投稿サービスで、あるストーリーを閲覧者が読めるかどうかを判定します。
// - 作者本人はいつでも読める
// - 公開範囲が全体公開なら誰でも読める。フォロワー限定ならフォローしていないと読めない
// - 有料作品は VIP 会員なら読める
// - VIP でなくても、その話を単話購入していれば読める

data UserId = String
    invariant String.length(value) > 0

data StoryId = String
    invariant String.length(value) > 0

data Audience = Everyone | FollowersOnly
data Pricing = Free | Paid

data Story =
    { id: StoryId
    , author: UserId
    , audience: Audience
    , pricing: Pricing
    }

data RefusalReason = FollowerRequired | PurchaseRequired
data Refused =
    { reason: RefusalReason
    }

behavior followership : (reader: UserId, author: UserId) -> Follower | NotFollower

behavior membership : (reader: UserId) -> Vip | Regular

behavior purchase : (reader: UserId, story: StoryId) -> Purchased | NotPurchased

behavior read : (story: Story, reader: UserId) -> Allowed | Refused
    depends on followership, membership, purchase
    constructs Refused

let read (story, reader, followership, membership, purchase) = {
    guard story.author /= reader else Allowed

    let audienceAllows = match story.audience with
        | Everyone -> true
        | FollowersOnly
            -> match followership(reader, story.author) with
                | Follower -> true
                | NotFollower -> false

    guard audienceAllows else Refused { reason = FollowerRequired }

    match story.pricing with
        | Free -> Allowed
        | Paid
            -> match membership(reader) with
                | Vip -> Allowed
                | Regular
                    -> match purchase(reader, story.id) with
                        | Purchased -> Allowed
                        | NotPurchased -> Refused { reason = PurchaseRequired }
}
```

このモデルで確認したいのは、構文の短さではありません。まず、Decisionパターンとの違いを押さえておきます。

Decisionパターンは、判断の途中で必要になったデータを中間状態として返し、呼び出し側に取得してもらいます。Southerは必要なデータの取得をbehaviorの呼び出しとしてモデル内に書き、その実装だけを外から与えます。`depends on`で受け取ったbehaviorをその場で呼べるため、判断の途中で呼び出し側へ制御を戻す必要がありません。

この違いを踏まえて、次の順に見ていきます。

1. 閲覧可否を決める分岐を`read`に残し、業務ルールを一か所で読めるようにする
2. DBから事実を取得する処理をJava側に置き、接続失敗やタイムアウトを業務上の結果へ混ぜない
3. `example`に書いた仕様をモデルとJava実装の両方へ適用し、ケースの不足や実装とのずれを検出する

Southerが優先するのは、業務上の判断をモデルに漏れなく書く「完全性」と、判断に必要なデータだけを取得する「性能」です。ただし、ここで諦める純粋性の範囲を分けて考える必要があります。

Southerで記述した`let read`の判断手順は、依存するbehaviorの答えも入力として扱えば、その入力から結果が決まります。この意味では、モデル内の判断手順は純粋です。一方、実行時にJava実装をbehaviorへバインドすると、呼び出し全体はDBやキャッシュに触れます。Southerは、実行全体まで純粋であることは求めません。

したがって、ここで問い直したいのは「純粋性に価値があるか」ではなく、ドメインモデルに必要なのはどちらの純粋性なのか、という点です。

純粋性の利点として最もよく挙げられるのは、テストのしやすさです。外部I/Oに依存せず、同じ入力に対して同じ出力が得られるのであれば、振る舞いを局所的に検証しやすい。これは確かに有用です。

ただし、ドメインモデルにとってより重要なのは、純粋であることそのものではありません。ある振る舞いがどのような入力を受け取り、どのような出力を返し得るのか。さらに、その出力が入力に対してどのような意味を持つのか。これらが明示されていることだと私は考えます。

後続の処理が内部実装を知らなくても、契約だけを頼りに安全に呼び出し、ほかの振る舞いと合成できることの方が重要です。

たとえば、単に

```elm
UserId -> List<Follower>
```

という型が分かるだけでは不十分です。その `Follower` が、引数で与えた `UserId` に対する follower であることまで分からなければ、後続の処理はその値を安心して利用できません。

Southerでは、たとえば次のように書きます。

```elm
behavior myFollower : (userId: UserId) -> List<Follower>
    ensures userId = value.followeeId
```

`value`は、このbehaviorが返す`Follower`を表します。戻り値がリストの場合は、その各要素について`userId = value.followeeId`が成り立つことを要求します。シグネチャが値の形を表すのに対し、`ensures`は入力と出力の間に必要な関係を表します。

ここで重要なのは、この振る舞いが純粋か非純粋かではありません。引数で渡した`UserId`と、返された`Follower`がどのような関係にあるのかまで、契約としてモデル上に現れていることです。外部実装をこの契約や具体的な`example`に照らして検証する方法は、後半で扱います。

純粋性は、このような契約を安定して扱いやすくするための有力な性質ではありますが、合成可能性を生み出している本体は純粋性ではありません。振る舞いの意味が契約として明示されていることです。

上記モデルには`followership`、`membership`、`purchase`、`read`の4つのbehaviorがあります。ここで区別したいのは「純粋かどうか」ではなく、「どこまでがモデルとして書かれていて、どこから先が外部の実装に委ねられているか」です。

`read`には`let`で判断の手順が書かれています。一方、`followership`、`membership`、`purchase`には宣言だけがあり、実行時には外部で定義した実装をバインドします。Southerでは、実装がモデルの内側にあるか外側にあるかにかかわらず、いずれもドメインの振る舞いとして扱います。

- 純粋かどうかによってドメインの振る舞いであるかを区別しない。
- どちらの場合でも、振る舞いはドメインのデータ入出力だけで組み立てられ、意味が契約として読める形になっているべき。

Southerで保ちたいのはこの2点です。

ここで、何を純粋と呼んでいるのかを分けておきます。Southerのモデルとして書かれた`let read`の判断手順は、依存するbehaviorも含めて入力が与えられれば、その入力から結果が決まります。一方、実行時に`followership`や`membership`へJavaの実装をバインドすると、呼び出し全体はDBやキャッシュに触れるため、純粋な計算ではありません。

Southerは、この2つを混ぜません。モデルの中に実装を持つbehaviorと、外部から実装を注入するbehaviorを宣言上で区別し、外界に触れる箇所を追跡できるようにします。そのうえで、`read`のシグネチャや分岐にはDBや通信の都合を持ち込ませません。外部I/Oがあっても業務上の入力と結果だけで振る舞いを組み立てられることが、Southerで残したい合成可能性です。

## 業務上の結果と実行時エラーを分ける

Southerの言語モデルには、業務上の結果とは別に投げられる例外がありません。起こりうる業務上の帰結は、behaviorの戻り値に含めます。

一方、behaviorへバインドしたJava実装では、DB接続の失敗や通信のタイムアウトが発生します。Southerはそれらを`Follower`や`NotFollower`のようなドメインの値へ変換しません。Java側の例外はJava側へ伝わり、behaviorの評価は完了しなかったものとして扱います。つまり、なくしているのは実行時エラーではなく、業務上の結果と技術的な失敗を同じ戻り値へ混ぜることです。

今回の `read` が返しうる結果は、次の2つだけです。

```elm
behavior read : (story: Story, reader: UserId) -> Allowed | Refused

data RefusalReason = FollowerRequired | PurchaseRequired

data Refused = {
    reason: RefusalReason
}
```

読めるなら `Allowed`、読めないなら `Refused` です。

さらに、読めない理由も業務上意味のあるものだけを型にしています。フォロワー限定なのにフォローしていなければ `FollowerRequired`、有料作品なのにVIPでも購入済みでもなければ `PurchaseRequired` です。

フォロワー限定の作品をフォローしていない人が開くことも、有料作品を未購入の人が開くことも、業務として普通に起こりうることです。であれば、それは例外ではなく `read` が取りうる正当な結果として最初から型に現れているべきという考えです。

もしこれを例外で表現すると、たとえば表面上は

```elm
(Story, UserId) -> Allowed
```

のように見えていても、実際には裏側で `FollowerRequiredException` や `PurchaseRequiredException` が発生することになります。そうなると、`read` が実際に何を起こしうるのかはシグネチャだけでは分かりません。

```elm
Allowed | Refused
```

と書いてある以上、業務上起こりうる結果はこの中にあります。呼び出し側もそのどちらかを扱うことになります。

これは先ほど書いた「振る舞いの入力と出力が明示されていること」の続きでもあります。

Southerが重視しているのは、単に処理が例外を投げないことではなく、どういう入力を受け取りどういう結果を返しうるのかを振る舞いの型の外に逃がさないことです。

一方で、`read` が依存している次の3つは少し性質が違います。

```elm
behavior followership : (reader: UserId, author: UserId) -> Follower | NotFollower

behavior membership : (reader: UserId) -> Vip | Regular

behavior purchase : (reader: UserId, story: StoryId) -> Purchased | NotPurchased
```

これらはDBやキャッシュなど、ドメインモデルの外から取得される事実です。

たとえば `followership` の実装がDBを読んでいるとして、そのDBが落ちることは当然あります。しかし、そのときに `DatabaseUnavailable` や `QueryTimeout` のような結果を `Follower | NotFollower` に混ぜることはしません。

なぜなら、それらは「reader と author のフォロー関係が何であるか」という業務上の答えではないからです。

`followership` がモデルの内側へ持ち込むのは、あくまで

```elm
Follower | NotFollower
```

というドメイン上の事実だけです。

同じように、`membership` が持ち込むのは `Vip | Regular`、`purchase` が持ち込むのは `Purchased | NotPurchased` だけです。

DBからどう取得するか、失敗したらリトライするのか、タイムアウトをどう扱うのか、といった話は、外部実装をバインドする側に残します。

たとえば`followership`のJava実装でDB接続に失敗した場合、その例外をSoutherが`Follower`や`NotFollower`へ変換することはありません。例外はそのままJava側へ伝わり、トランザクションのロールバック、リトライ、HTTP 5xxへの変換といった実行環境の仕組みで扱います。このとき`read`が「フォローしていない」と判断するわけではなく、業務上の判断そのものが完了していません。

つまりSoutherがやっているのは、エラーをなくすことではありません。behaviorの戻り値を見れば業務上起こりうる帰結を把握でき、技術的な失敗はプラットフォーム側の仕組みで処理できるように、両者の経路を分けています。外部実装の失敗を隠さず、それでもドメインの型をインフラ都合で広げずに済むのが、この非対称な境界の利点です。

この区別は、振る舞いを合成するときに効いてきます。

`read` の中では、

```elm
match followership(reader, story.author) with
  | Follower -> ...
  | NotFollower -> ...
```

と書けばよく、そこに「DB取得に失敗した場合」という第3のケースは存在しません。

`membership` や `purchase` についても同じです。

そのため `read` が扱っているのは最初から最後まで、

- 作者本人か
- 公開範囲は何か
- フォローしているか
- 無料か有料か
- VIPか
- 購入済みか

という業務の言語だけになります。

Southerに例外がないことの意義は、単に例外処理を書かなくてよいことではありません。

業務上起こりうることは結果として明示し、業務ではない失敗はドメインの振る舞いへ持ち込まない。Southerに例外がないのは、この区別を保つためです。

## ドメインの外とのつながり

```java
public final class JdbcFollowership extends Followership {
    @Override
    public FollowershipResult apply(UserId reader, UserId author) {
        return dsl.fetchExists(FOLLOWS,
                FOLLOWS.FOLLOWER_ID.eq(UserId.encoder().encode(reader)),
                FOLLOWS.AUTHOR_ID.eq(UserId.encoder().encode(author)))
            ? Follower()
            : NotFollower();
    }
}
```

`behavior`がSoutherで実装されていない場合は、`Followership`クラスが生成されます。このクラスを継承して、外部実装とのバインディングを書きます。この例ではencoderしか出てきませんが、ドメインモデルへのデータマッピングに必要なencoderとdecoderはSoutherが生成します。そのため、テーブルやクエリ結果と1対1で対応する型を別に用意する必要はありません。

## ドメインモデルの正しさの検証

Southerが特に力を入れているのが、ドメインモデルの検証です。behaviorごとに`example`を定義し、その中に複数のケースを記述して、入力と出力の関係を検証できます。

```elm
example read
    | "全体公開の無料作品はフォローしていなくても読める"
        : (storyOf(Everyone, Free), フォロワーでない人)
        -> Allowed
    | "フォロワー限定の無料作品はフォロワーなら読める"
        : (storyOf(FollowersOnly, Free), フォロワー)
        -> Allowed
    | "フォロワー限定の無料作品はフォローしていないと断られる"
        : (storyOf(FollowersOnly, Free), フォロワーでない人)
        -> Refused { reason = FollowerRequired }
```

`example`の各行は、ほかの言語ならxUnitのテストコードとして書く1ケースに相当します。Southerでは、モデル内で実装されたbehaviorについて、これらのケースをコンパイル時に検証します。別のテストランナーでモデルのテストを実行する必要はありません。

Southerは、記述したケースで十分かどうかも判定します。足りない場合は、追加すべきケースの候補を生成します。同値クラスや境界値、実装中の分岐や条件の組み合わせなどから、ドメインモデルの検証に必要なケースを探します。この仕組みについては、別の記事で詳しく説明する予定です。

ケース充足度（adequacy）が`satisfied`になるまでケースの追加とモデルの修正を繰り返すことで、仕様の考慮漏れを減らせます。

```elm
// behavior read についてケース充足度レポート。
  read                     implemented   rows 0    pending 0
    signature   not measured (no row names this behavior)
    partition   axes 2   equivalence partitions 0/0   (2 not measured: no row names this behavior)
      · no line: comparison@84:24 — it relates two positions rather than dividing one, about `reader`
      · no line: comparison@84:24 — it relates two positions rather than dividing one, about `story.author`

4 behaviors: 1 implemented, 0 unimplemented, 3 injected; 0 rows waiting for a `let`.
adequacy: undetermined

// 足すべきケース
// generated by `souther examples --generate`: 2 rows to fill what nothing covers.
// Replace each `<?>` with what the system actually answers.
example read
    | "story.audience=FollowersOnly"
        : (
            Story {
                id = StoryId("x"),
                author = UserId("x"),
                audience = FollowersOnly,
                pricing = Free
            },
            UserId("x")
        )
        -> <?>
    | "story.pricing=Paid"
        : (
            Story { id = StoryId("x"), author = UserId("x"), audience = Everyone, pricing = Paid },
            UserId("x")
        )
        -> <?>
```

これは、振る舞いが全域的に定義されていることを、コンパイル時の検証に利用する仕組みです。全域性については別の大きな話になるため、ここでは割愛します。

では、実行時に外部の実装を呼び出すbehaviorはどう検証するのでしょうか。上記の`read`は、`followership`、`membership`、`purchase`に依存しています。そこで、これらのbehaviorには`fake`を用意します。

モックを使ったテストに似ていますが、単に適当な戻り値を返すものではありません。`example`を充足させるために必要なパターンを返せるように定義します。

```elm
let storyOf (audience: Audience, pricing: Pricing): Story =
    Story {
        id = StoryId("story-1"),
        author = UserId("author-1"),
        audience = audience,
        pricing = pricing
    }

let フォロワー: UserId = UserId("reader-1")
let フォロワーでない人: UserId = UserId("stranger")
let VIP会員: UserId = UserId("vip-reader")

fake followership
    | (UserId("reader-1"), UserId("author-1"))   -> Follower
    | (UserId("vip-reader"), UserId("author-1")) -> Follower
    | (UserId("buyer"), UserId("author-1"))      -> Follower
    | _                                          -> NotFollower

fake membership
    | (UserId("vip-reader")) -> Vip
    | _                      -> Regular

fake purchase
    | (UserId("buyer"), StoryId("story-1")) -> Purchased
    | _                                     -> NotPurchased

example read
    | "フォロワー限定の有料作品を VIP のフォロワーが開くと読める"
        : (storyOf(FollowersOnly, Paid), VIP会員)
        -> Allowed
    | "フォロワー限定の有料作品をフォロワーが未購入で開くと購入を求められる"
        : (storyOf(FollowersOnly, Paid), フォロワー)
        -> Refused { reason = PurchaseRequired }
    | "フォロワー限定の有料作品をフォローしていない人が開くとフォローを求められる"
        : (storyOf(FollowersOnly, Paid), フォロワーでない人)
        -> Refused { reason = FollowerRequired }
```

## 外部実装を同じ仕様で検証する

外部から注入するbehaviorは、実際にDBやキャッシュからドメイン上の事実を取得します。モデル内の判断だけを検証しても、この実装が間違っていれば`read`は正しい結果を返せません。

そこでSoutherは、モデルに書いた`example`を外部実装のテストにも利用します。検証には二つの方法があります。

### exampleを外部実装に適用する (Contract Testing)

一つ目は、behaviorに書いた`example`を、fakeではなく実際のJava実装へバインドして実行する方法です。次の例では、`membership`の実装として`JooqMembership`をバインドし、そのbehaviorに対する各行をJUnitから評価しています。`read`全体を評価する場合は、依存する`followership`、`membership`、`purchase`の実装をすべてバインドします。

```java
   BoundExamples examples = SoutherExamples.of(MODEL).bind(new JooqMembership(dsl));

   assertAll(examples.rows().stream().map(row -> () -> {
       RowEvaluation evaluation = examples.evaluate(row);
       assertThat(evaluation.held())
               .describedAs("%s: %s", row.shown(), evaluation.shown(Locale.JAPANESE))
               .isTrue();
   }));
 
```

### fakeと実装を比較する (Differential Testing)

二つ目は、`fake`に書いた答えと外部実装の答えを比較する差分テストです。`fake`と実装へ同じ入力を与え、同じドメイン上の値が返るかを確認します。

```java
   BoundExamples examples = SoutherExamples.of(MODEL).bind(new JooqMembership(dsl));

   assertAll(examples.standinEntries().stream().map(entry -> () -> {
       StandinObservation observation = examples.observe(entry);
       assertThat(observation)
               .describedAs("%s(%s): fake は %s と書いているが実装の答えが違う — %s",
                       entry.behavior(), String.join(", ", entry.shownInputs()),
                       entry.shownStated(), observation)
               .isInstanceOf(StandinObservation.AsStated.class);
   }));

```

adequacyが`satisfied`であれば、`fake`にはドメインモデルの検証に必要な入力パターンがそろっています。差分テストでは、Injected behaviorの実装が同じ入力に対して`fake`と同じ出力を返すかを検証します。

ただし、adequacyが保証するのはケースの充足度であり、`fake`に書いた期待値が業務として正しいことではありません。どの入力に対して何が起きるべきかを決めるのは、あくまで人間です。これは通常のテストでも同じで、テストフレームワークが期待値の正しさまで決めてくれるわけではありません。

## 検証の仕組みをどう使い分けるか

ここまでに登場した仕組みは、それぞれ異なる対象を記述、検証します。

| 仕組み | 記述、検証するもの |
| --- | --- |
| behaviorのシグネチャ | 入力と、結果として現れうる型 |
| `ensures` | 入力と出力の間に常に必要な関係 (事後条件) |
| `example` | 具体的な入力に対して期待する結果 |
| adequacy | exampleが実装の分岐や入力領域をどこまでカバーしているか |
| `fake` | Injected behaviorについて、モデルが必要とする入力と答え |
| Java側のテスト | 外部実装がexampleやfakeと同じ結果を返すか |

Southerが業務判断そのものを決めるわけではありません。人間が決めた答えをモデルに置き、その答えだけでは検証できていない入力領域と、モデルと外部実装のずれを機械的に見つけます。

## まとめ

Southerはトリレンマを消すわけではありません。純粋性を「モデル内の判断手順」と「外部I/Oを含む実行全体」に分け、前者を保ちながら後者には要求しない、という選択をします。

- 完全性は、閲覧可否を決める分岐を`read`に残すことで守る
- 性能は、判断の途中で必要になったbehaviorだけを呼び出すことで守る
- モデル内の純粋性は、外部I/Oの実装をInjected behaviorとして分離することで守る
- 業務上の結果と技術的な失敗は、戻り値とJava例外の別経路で扱う
- モデルと外部実装の整合は、同じ`example`と`fake`を使って検証する

そのために必要なのは、呼び出し全体を純粋な関数にすることではありません。behaviorの入出力とその意味を契約としてモデルに残し、外部I/Oを伴うbehaviorも同じ仕様で検証できることです。Southerでは、この条件を保ったままbehaviorを組み合わせられる状態を、ドメインモデルの合成可能性として扱います。

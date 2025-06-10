# Spring Security

カスタマイズしやすい ~~（AOP はめんどい）~~ 、そして強力な認証とアクセス制御のフレームワーク<br>
アクセス可能なページやログイン url、パスワードの暗号化手法などは基本的に Config に書く<br>
とりあえず今回だけ乗り切りたい人は[#認可](#認可)を読んで<br>

## Config

今回は config パッケージに SecurityConfig.java を置いて設定を記述する（この置き場所と名前が人によって違うとかあんまないと思う）

### アノテーション

- **@Configuration**<br>
  Spring にコンフィグファイルであることを認識させる

- **@EnableWebSecurity**<br>
  Spring Security のコンフィグなのでそれを有効化

- **@EnableMethodSecurity**<br>
  メソッドごとのセキュリティを有効化（後述する[#認可](#認可)にあるアノテーションが使えるようになる）

- **@Bean**<br>
  Spring に Bean として登録することを示す<br>
  Bean は Spring が管理するオブジェクトで、Spring が起動時に読み込んで設定を永続化する<br>

### メソッド

- **PasswordEncoder passwordEncoder()**<br>
  パスワードの暗号化手法を指定する
  今回は BCryptPasswordEncoder を使う ~~← 何でこれにしたか特に理由はない~~

- **SecurityFilterChain securityFilterChain(HttpSecurity http)**<br>
  HttpSecurity を使って、どの URL にどのようなアクセス制御を行うかを設定する<br>
  ログインやログアウトの URL もここで設定する<br>
  url が複雑なのでここでは大まかに設定する<br>
  メソッドごとにアクセス制御を行う場合は、`@PreAuthorize` や `@Secured`（古い書き方らしいから今回は使わない） アノテーションを使う<br>
  デフォルトでは username でログインするので、email でログインする設定も今回は行う

## User 管理の Service クラス

Spring Security では、ユーザ情報を保持する UserDetails クラスと、それを取得、管理するための UserDetailsService インターフェースを実装する必要がある

- **CustomerPrincipal**<br>
  UserDetails インターフェースを実装したクラス<br>
  一番安直なのは UserDetailsImpl ってクラス名だけど、今回は DB に合わせて CustomerPrincipal と名付ける<br>
  ユーザの詳細情報を取得するためのメソッドを実装する<br>
  フィールドに Customer エンティティを持ち、実装は customer のフィールドを返すようにする

- **CustomerService**<br>
  UserDetailsService インターフェースを実装したクラス<br>
  同様に UserDetailsServiceImpl ってクラス名になりがちだけど、DB に合わせて CustomerService と名付ける<br>
  ユーザ自体の取得や登録を行う Service クラス<br>
  Customer の更新処理をサービスに書きたい場合はここに追記してください

## 認可

認可は、クラスやメソッドごとにアクセス制御を行うためのアノテーションを使ったり、html の要素にアクセス制御を行うための属性を追加することで実現する

### Java のアノテーション

- **@PreAuthorize**<br>
  メソッドの実行前に、指定した条件を満たすかどうかをチェックする<br>
  例えば、`@PreAuthorize("hasRole('ADMIN')")` と書くと、ADMIN ロールを持つユーザのみがそのメソッドを実行できるようになる<br>
  `@PreAuthorize("isAuthenticated()")` と書くと、認証済みのユーザのみがそのメソッドを実行できるようになる<br>
  `@PreAuthorize("hasRole('ADMIN') or hasRole('USER')")` のように複数のロールを指定することも可能<br>
  `@PreAuthorize("not hasRole('ADMIN')")` のように、特定のロールを持たないユーザに対してアクセスを許可することもできる<br>
  `@PreAuthorize("isAnonymous()")` と書くと、認証されていないユーザのみがそのメソッドを実行できるようになる。多分使わん、使う場面わかんね

- **@Secured**<br>
  古い書き方のアノテーションで、`@Secured("ROLE_ADMIN")` のように書く<br>
  今は `@PreAuthorize` を使うことが推奨されているので、あまり使わない。~~使うな~~

### Thymeleaf

thymeleaf の基本操作が分かっている前提で書いてますごめんね 👅

- **sec:authorize**<br>
  Thymeleaf の属性で、要素の表示/非表示を制御する<br>
  `sec:authorize="hasRole('ADMIN')"` と書くと、ADMIN ロールを持つユーザのみがその要素を表示できるようになる<br>
  `sec:authorize="isAuthenticated()"` と書くと、認証済みのユーザのみがその要素を表示できるようになる<br>
  以降 PreAuthorize と同じように使えるから言わなくてもわかれ

- **sec:authentication**<br>
  Thymeleaf の属性で、現在の認証情報を取得する<br>
  `sec:authentication="name"` と書くと、現在のユーザの名前を取得できる<br>
  他にもあるけどたぶん使わん

- **#authentication**<br>
  Thymeleaf のオブジェクトで、現在の認証情報を取得する<br>
  これらはログインしてないと null になるので、`sec:authorize="<ログイン状況>"`等の div で囲む必要がある<br>
  これで **th:if** や **th:text** などで使える<br>
  `#authentication.principal` と書くと、現在のユーザの詳細情報を取得できる<br>
  `#authentication.principal.username` と書くと、現在のユーザのユーザ名を取得できる<br>
  `#authentication.principal.role` と書くと、現在のユーザのロールを取得できる<br>
  CustomerPrincipal が読めてる人はなんとなく分かったかもしれないけど、`#authentication.principal` は CustomerPrincipal のインスタンスなので、Customer エンティティのフィールドを取得することもできる<br>
  `#authentication.principal.customer.email` と書くと、現在のユーザのメールアドレスを取得できる<br>
  <br>

# おまけ

## csrf について

CSRF（Cross-Site Request Forgery）対策として、Spring Security はデフォルトで CSRF トークンを有効にしている<br>
CSRF とは、悪意のあるサイトが、ユーザがログインしている別のサイトに対して不正なリクエストを送信する攻撃手法のこと<br>
CSRF トークンは、フォーム送信時に自動的に生成され、リクエストヘッダーに含まれる<br>
このトークンを照合することで、リクエストがそのサイトからのものであることを確認する<br>
CSRF トークンは、 `th:action` を指定したフォームに自動的に追加される<br>
指定しない場合は、`<input type="hidden" name="${_csrf.parameterName}" value="${_csrf.token}"/>` を追加する必要がある（送信先が二種類あるようなフォーム以外で使うことはない）<br>

## Lombok について（Spring Security とは関係ない）

Lombok は、Java のコードを簡潔に書くためのライブラリで、**Getter** や **Setter**、**コンストラクタ**などを自動生成することができる<br>
Lombok を使うことで、書かなくてもわかる処理が簡略化され、コードの可読性が向上する<br>
Lombok を使うためには、`@Data`, `@Getter`, `@Setter`, `@NoArgsConstructor`, `@AllArgsConstructor` などのアノテーションをクラスに付与する<br>
`@Getter`と`@Setter` はフィールドに付与することで、生成する Getter と Setter をフィールドごとに指定することもできる<br>
`@Data` は `@Getter`, `@Setter`, `@ToString`, `@EqualsAndHashCode`, `@RequiredArgsConstructor` をまとめて付与したもの<br>
急に出てきた知らんアノテーション 3 つについては勝手に調べろ

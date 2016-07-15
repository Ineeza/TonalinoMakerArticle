## RESTって何？
REST(Representational State Transfer)とは、Web APIの開発において標準的なアーキテクチャです。
RESTとは、ステートレスなクライアントとサーバーの関係を意味します。それはサーバーから取得されいるクライアントのコンテキストがない他の多くのアプローチとは異なるという意味です。
それとは逆に、RESTの場合、各リクエストはユーザの認証のために必要なすべての情報を持つ必要があります。
そしてどんなセッションデータをしっかりと送信しなければいけません。

RESTを使うメリットは、既に存在しているHTTPアーキテクチャにHTTPリクエストメソッドを適用できる点です。
これらの操作は、以下のような操作から構成されています。

    GET - サーバーから基本的なデータ取得するときに使われます。
    PUT - サーバーに既に存在しているオブジェクトの変更をするときに使われます。
    POST - サーバーに新しくオブジェクトを作成するときに使われます。
    DELETE - サーバーから既存のオブジェクトを取り除くときに使われます。

これらの操作を利用するURIエンドポイントを作ることで、RESTful APIはすぐに出来上がります。

## APIって何？
この意味でAPI(Application Programming Interface)とは、外部のプログラムから公開されているメソッドへアクセス
して操作できるものです。APIの共通の使い方として、実際にサイトを訪問することなく、データだけをアプリケーションから取得することができます。（例えば, 人気レシピ.comからケーキのレシピのデータを取得するようなことができます。）
このデータの取得が行われるには、特定の外部アプリケーションのリクエストを許可したAPIを公開して、外部アプリケーションの内側からユーザーにデータを返すようにすれば良いでしょう。
Web上では、RESTful URIを使用して、よく実装されています。
先ほどのケーキのAPIの例では、以下のようなURIを含むことができるでしょう。
```
人気レシピ.com/api/v1/recipe/cake
```
このURIをRESTfulエンドポイントといいます。もしこのURIに対してGETリクエストを送信すると、アプリケーションが持つ最新のケーキのレシピのリストを表示させることができるでしょう。もし代わりに/cake/141のようなリクエストをするならば、特定のケーキの詳細を取得することができるでしょう。これら両方は、推測しやすいアプリケーションとの情報のやり取り方法をつくる、理に適ったエンドポイントの例です。

## RESTful APIをつくってみよう！
ここで作ろうとしているAPIは2つのクラスで構成されます。1つ目は、URIをパースしてレスポンスを返す抽象クラスです。
2つ目は、APIためのエンドポイントから構成される具象クラスです。このように要素を分解して考えることができるように、まずどのRESTful APIの基礎として使える、かつアプリケーション自体のユニークなすべてのコードから分離した場所に置ける抽象クラスを用意します。

しかしながら、それらのクラスを書き始める前に、準備すべきことがあります。

## .htaccessファイルを書こう。
.htaccessファイルは、Webサーバーが.htaccessが置いてあるディレクトリに対してくるリクエストをどのように処理するかを記述されている設定ファイルです。APIが含むすべてのエンドポイントに対して新しくPHPファイルを作成したくないので、このような設定ファイルを書きます。（メンテナンスするのが難くなるという他の理由もあります。）
APIにくるすべてのリクエストが意図されているコントローラーに向けて到達して、特定のエンドポイントに対応したコードが実行されること。そのことを念頭に置いて、.htaccessファイルを作りましょう。
```
<IfModule mod_rewrite.c>
RewriteEngine On
RewriteCond %{REQUEST_FILENAME} !-f
RewriteCond %{REQUEST_FILENAME} !-d
RewriteRule api/v1/(.*)$ api/v1/api.php?request=$1 [QSA,NC,L]
</IfModule>
```

#### この設定ファイルは何してるの？
この設定ファイルについて少し言及します。まずここで最初にすべてを内包している部分は、mod_rewrite.cファイルが存在していることを確認するための記述です。もしそのApacheのモジュールが有効ならば、次の記述に進みます。RewriteEngineを有効にしてから、次の２つのルールを追加します。これらのルールはリクエストされたURIが既に存在しているファイル名やディレクトリ名と一致していなければ書換えを行います。

次の行は、実際のRewriteRuleの宣言です。これは、api/v1/に存在しないファイルやディレクリに対するどのようなリクエストであっても、api/v1/index.phpに送られるべきという記述です。(.＊)マークはキャプチャと呼ばれ、デリミタ（$1）を通してリクエストの変数としてMyAPI.phpに送られます。最後の行では、幾つかのフラグをどのように書き換えるかを設定しています。
最初に[QSA]は[QSA]はQueryStringAppendの略で、最新の作成されたURIに対してクエリ文字列を付与します。2つめの[NC]はNot Case sensitiveの略で、大文字・小文字を区別しないという意味です。最後に[L]はmod_rewriteにこれ以上追加ルールを適用しないための標識です。

## 抽象クラスを構築しよう
.htaccessファイルがあるところで、さっそく抽象クラスをつくりましょう。既に説明したように、このクラスは今回のAPIが使うことになるすべての独自エンドポイントに対して動作します。そのためには、私達のすべてのリクエストを取得可能で、URIのStringからエンドポイントを捉え、HTTPメソッド（GET,POST,PUT,DELETE）を見つけ出し、ヘッダーやURIのデータを組み合わせます。
それが済んだら、抽象クラスは実際に実行されるために具象クラスのメソッドにリクエスト情報を渡します。その次に、HTTPレスポンスを成形する処理を行う抽象クラスをクライアントに返します。

まず最初にクラスとプロパティ、コンストラクタを宣言します。

```
abstract class API
{
    /**
     * Property: method
     * The HTTP method this request was made in, either GET, POST, PUT or DELETE
     */
    protected $method = '';
    /**
     * Property: endpoint
     * The Model requested in the URI. eg: /files
     */
    protected $endpoint = '';
    /**
     * Property: verb
     * An optional additional descriptor about the endpoint, used for things that can
     * not be handled by the basic methods. eg: /files/process
     */
    protected $verb = '';
    /**
     * Property: args
     * Any additional URI components after the endpoint and verb have been removed, in our
     * case, an integer ID for the resource. eg: /<endpoint>/<verb>/<arg0>/<arg1>
     * or /<endpoint>/<arg0>
     */
    protected $args = Array();
    /**
     * Property: file
     * Stores the input of the PUT request
     */
     protected $file = Null;

    /**
     * Constructor: __construct
     * Allow for CORS, assemble and pre-process the data
     */
    public function __construct($request) {
        header("Access-Control-Allow-Orgin: *");
        header("Access-Control-Allow-Methods: *");
        header("Content-Type: application/json");

        $this->args = explode('/', rtrim($request, '/'));
        $this->endpoint = array_shift($this->args);
        if (array_key_exists(0, $this->args) && !is_numeric($this->args[0])) {
            $this->verb = array_shift($this->args);
        }

        $this->method = $_SERVER['REQUEST_METHOD'];
        if ($this->method == 'POST' && array_key_exists('HTTP_X_HTTP_METHOD', $_SERVER)) {
            if ($_SERVER['HTTP_X_HTTP_METHOD'] == 'DELETE') {
                $this->method = 'DELETE';
            } else if ($_SERVER['HTTP_X_HTTP_METHOD'] == 'PUT') {
                $this->method = 'PUT';
            } else {
                throw new Exception("Unexpected Header");
            }
        }

        switch($this->method) {
        case 'DELETE':
        case 'POST':
            $this->request = $this->_cleanInputs($_POST);
            break;
        case 'GET':
            $this->request = $this->_cleanInputs($_GET);
            break;
        case 'PUT':
            $this->request = $this->_cleanInputs($_GET);
            $this->file = file_get_contents("php://input");
            break;
        default:
            $this->_response('Invalid Method', 405);
            break;
        }
    }
}
```

この抽象クラスを宣言してから、このクラスの中にPHPで具象クラスを書くことはできません。そのため、幾つかのprotectedクラスメンバをつくります。protectedメンバだけが対象のクラスとその子クラスにアクセスできます（メンバを定義したクラスにだけアクセスできるprivate変数のような振る舞いではありません）。

## CORSについて
APIの主な前提として、クライアントは1つのドメインだけでなく複数のドメインからAPIに対して、データを送受信することです。
そこには、攻撃者が模倣したページをつくり、あちこちで送信されたデータを盗まれてしまうセキュリティリスクを伴います。
故に、この通称CORS（Cross-Origin Resource Sharing）と呼ばれる機能を必要なページにのみ明示的に有効にされる必要があります。CORSをもっと詳しく知りたい方は、<a href="http://enable-cors.org/">Enable CORS</a>を御覧ください。

今回の私達がつくるAPIで、CORSが有効になっているかどうかを確認する必要があります。そしてまず最初に__constructメソッドにカスタムヘッダーを追加します。最初の2行はおまじないだと思っても構いません。まず私たちはこのページを通過するどんな出発地からのリクエストも許可してから、次にどんなHTTPメソッドも許可します。

これで驚くほど単純だけど、とっても重要なCORSリクエストの許可する準備は完了です。今回のスクリプトがクライアントのどんなことを尋ねているのかを理解する良いきっかけになりましたね。
.htaccessファイルから私達のスクリプトに送信される$requestを取得して（クライアントがリクエストした独自のURIを含んでいるのを覚えていますか？）、そして私達が必要なコンポーネントに分割すること。
取得可能なエンドポイントの最初の要素から、スラッシュ毎に展開された時点で、もし配列の次の枠が動詞に当てはまるなら、他のどんな項目も$argsとして利用されます。

HTTPメソッドはこのリクエストの目的を表します。GETリクエストは、簡単に見つけることができますが、DELETEとPUTリクエストはHTTP_X_METHODヘッダを通してPOSTリクエストの内部に隠されています。メソッドが選ばれた時点で、妥当なデータソースがパースされ、実行される前に安全に除去されます。

## 抽象クラスを完成させよう。
今のところ、具象クラスの中でメソッドを呼び出す関数が不足しています、そのあとレスポンスを返す処理をしましょう。抽象クラスの残りの部分は次の通りです。
```
abstract class API
{
    ...

    public function processAPI() {
        if (method_exists($this, $this->endpoint)) {
            return $this->_response($this->{$this->endpoint}($this->args));
        }
        return $this->_response("No Endpoint: $this->endpoint", 404);
    }

    private function _response($data, $status = 200) {
        header("HTTP/1.1 " . $status . " " . $this->_requestStatus($status));
        return json_encode($data);
    }

    private function _cleanInputs($data) {
        $clean_input = Array();
        if (is_array($data)) {
            foreach ($data as $k => $v) {
                $clean_input[$k] = $this->_cleanInputs($v);
            }
        } else {
            $clean_input = trim(strip_tags($data));
        }
        return $clean_input;
    }

    private function _requestStatus($code) {
        $status = array(  
            200 => 'OK',
            404 => 'Not Found',   
            405 => 'Method Not Allowed',
            500 => 'Internal Server Error',
        );
        return ($status[$code])?$status[$code]:$status[500];
    }
```
ここで特出すべき関数は、processAPI()メソッドです。これは、APIで全体に公開されているメソッドで、その役割はクライアントがリクエストしたエンドポイントのために具象クラスがメソッドを実装するかどうかを決定することです。もしそうであれば、そのメソッドを呼び、もしくは404レスポンスを返します。残りの新しいコードは、単純にすべての可能なHTTPコードと入力サニタイザの配列マップです。

以上で、抽象クラスについての話は終わりです。ようやく、具体的な実装の時間です。

## 実際にAPIをつくってみよう！
はじめの方に話したCORSについて思い出してみてください。どのようにセキュリティの脆弱性は起こりますか？私たちは、アクセス元からユニークなAPIキー入力することで、可能な限り強固にAPIをつくりたいと思います。これは、既に登録済みで許可されている外部のホストだけが生成されたユニークAPIキーとドメイン名のペアを使ってAPIサービスの接続ができます。この例の目的は、抽象化されたAPIキーを実証するコードについて、後ほど省いて説明するためです。ちなみに、私達のAPIはユーザを確認するためにすべてのリクエストに対してユニークトークンを要求します。

```
require_once 'API.class.php';
class MyAPI extends API
{
    protected $User;

    public function __construct($request, $origin) {
        parent::__construct($request);

        // Abstracted out for example
        $APIKey = new Models\APIKey();
        $User = new Models\User();

        if (!array_key_exists('apiKey', $this->request)) {
            throw new Exception('No API Key provided');
        } else if (!$APIKey->verifyKey($this->request['apiKey'], $origin)) {
            throw new Exception('Invalid API Key');
        } else if (array_key_exists('token', $this->request) &&
             !$User->get('token', $this->request['token'])) {

            throw new Exception('Invalid User Token');
        }

        $this->User = $User;
    }

    /**
     * Example of an Endpoint
     */
     protected function example() {
        if ($this->method == 'GET') {
            return "Your name is " . $this->User->name;
        } else {
            return "Only accepts GET requests";
        }
     }
 }
 ```

 ## APIを使ってみよう！
 具象クラスをつくることは、とっても単純です。好みのエンドポイントをそれぞれ追加して、単純にエンドポイントの名前と一致する関数をMyAPIクラスに新しく追加するだけです。その際、$methodや$verb,$argsなどの変数をエンドポイント内に枝上のパスにつかうことができます。

実際にAPIを実装するためには、.htaccessファイルがすべてのリクエストを転送するPHPファイルを作成する必要があります。今回の例では、api.phpと名づけました。

```
// Requests from the same server don't have a HTTP_ORIGIN header
if (!array_key_exists('HTTP_ORIGIN', $_SERVER)) {
    $_SERVER['HTTP_ORIGIN'] = $_SERVER['SERVER_NAME'];
}

try {
    $API = new MyAPI($_REQUEST['request'], $_SERVER['HTTP_ORIGIN']);
    echo $API->processAPI();
} catch (Exception $e) {
    echo json_encode(Array('error' => $e->getMessage()));
}
```
ブラウザ上で/api/v1/exampleを見てみてください（ユーザとシステムが準備しているトークンを持って）、きっと何かエンドポイントからその出力を見れるでしょう。

## 終わりに
これでPHPでRestfulなAPIをつくる説明は終わりです。実際にやってみると、本当にたくさんの内容がありましたね。


参考記事
<a href="http://coreymaynard.com/blog/creating-a-restful-api-with-php/">Creating a RESTful API with PHP</a>

****
# Sunrise2017 Database Strategy
# VOYAGE GROUP
# システム本部　三浦裕典(@hironomiu)
****

# 第0章　はじめに

## 講義を行うにあたって

### 講義テーマ
本講義では「大規模なサービス」を展開するフェーズでは「データベース(以降DB)」に対する知見の重要度が格段にあがることを踏まえDB、DBサーバを中心とした講義を行います。

### 大規模サービスとは
「大規模なサービス」とは「100万人規模のユーザが利用するWebサービス」と言うBtoCモデルのWebサービスとします。今回はVOYAGE GROUPで展開している様々な「大規模サービス」で使用しているDB運用のノウハウを中心に講義を行います。

### 必要な技能
講義はDBの中でもRDBMSを中心とした知識があることを前提に進めます。出来るだけ特定のプロダクトの機能にフォーカスはあてず広くDB技術を俯瞰する形で進めます。また最近のDBのトレンドも踏まえ、様々なRDBMSやNoSQLの技術についても適時講義では紹介していきます。


# 第1章 DBサーバの性能計測、サイジング、I/O特性

サービスの成長に伴いDBサーバも、サービスのトラフィック量に合わせた拡張などの対策は必須となります。DBサーバは一定の規模を超えたところからボトルネックが顕在化しやすく対策もステージ毎に多岐に渡ります。対策を取る上でDBサーバの性能評価、サービス稼働中の稼働情報の取得、稼働情報からの考察は対策の根拠となるためとても重要です。稼働情報の取得、考察から将来を予測した内容と実際のサービスが成長した稼動実績で比較し乖離が発生することは多々ありますが事前に計測し考察から予測と実際の稼働実績を照らしあわせる活動を継続的に行うことにより柔軟に将来のサービスの拡張に対して備えることができ将来予測の確度も上がります。DBサーバの稼働情報の内容について考察したり将来予測をする上ではDBサーバにて利用されるミドルウェアや特徴を理解していなければ稼働情報の内容を正確に評価し将来予測できません。この章では性能計測、サイジング、特徴などから将来予測について学びます。
			
					
## 性能計測
DBサーバの性能を計測、評価するには大きくわけて二つの観点で行います。

### ベンチマークテスト
hdparmやddなどのコマンドを用いたread/writeの性能計測やツールを用いたCPU、メモリ、Diskの性能計測します。サービス稼働後ある程度の時間の経過後にサーバ増強をする場合に同じH/Wが好ましいが調達することが出来ない場合もあります。その場合は過去のベンチマークテストから最新のラインナップを選択する際の判断材料に用いる場合もあります。

### 負荷テスト
後述するDBサーバ(もしくはDB)への負荷をかけOSレベルでの負荷計測、DB上でどれだけの処理（SQL数、トランザクション数、同時実行数、その他）が捌けるかを計測します。その際にDBから出力される統計情報（DBの様々な稼動情報、主にSQLの実行数）とOSレベルでの計測情報の両方を付き合わせて負荷状況を観測することが大事です。そこから将来ボトルネック、リソース不足となる箇所を洗い出していきます。OSレベルでの負荷計測ではCPU使用率、I/O状況を複数のコマンドを用いて計測していきます。(Sunriseではtop,ps,dstatを用います)

- デバイス、ハードウェアなどの稼働情報取得コマンド例

```
$ sar
$ vmstat
$ iostat
$ dstat			
```

- ミドルウェア、プロセス、スレッドなどの稼働情報取得コマンド例

```
$ top
$ perftop
$ ps
```	

- サーバのリソース情報取得コマンド例

```
$ cat /etc/procinfo
$ free
```

## 負荷テストの内容				
DBサーバで性能計測する場合、実際の業務を想定したシナリオを負荷テストのシナリオとして作成し実施するか、RDBMSでは業界団体TPCによって策定された負荷テスト（主にTPC-C）を用いて期待する時間当たりのトランザクション処理量を負荷として掛けるのが一般的です。

### Think
負荷テストを行う環境などについて考えてみましょう

- ローカルホスト内でテストが完結する場合
- ネットワーク経由から負荷が掛けられる場合
  - 動作してるプロダクト(Web,App,DBが同じ環境、別々の環境)			
					
## DBサーバでのCPUの役割とSizing
DBサーバではCPUは実行計画の決定、データのソート、DBへの接続の確立、暗号化、ハッシュ化など様々な処理を行います。一般的にDBサーバの負荷の特徴は永続領域(HDD、SSDなど)へのRead/WriteによるDBサーバの処理性能が頭打ち(I/Oバウンド)になりやすく、CPUを利用し尽くす(CPUバウンド)状況はあまりありません。
	
### CPUサイジング		
仮に1つのSQLを処理するのにCPU時間で0.1秒かかるとします。

- 1秒間でSQLが何回実行できるでしょう？
- CPU1個で1秒あたり10回処理できるとした場合、2CPU、2コアの場合、1秒間で処理できるSQL数はいくつでしょう？
- SELECT、INSERT、UPDATE、DELETEの全てを1SQL0.1秒で見積もるのは妥当でしょうか？
- SQL文はでどのような内部動作で0.1秒掛かるでしょう？
- 実際の1SQLがどのくらいCPU時間を利用しているかOS、DBでの計測情報を使ってどのように導きだせるか考えてみましょう。

### 講義時の備忘録用キーワード
ソート、圧縮、オンメモリ、OS、プロセス、スレッド、コンテキストスイッチ、コネクションプール
					
### まとめ
今回はRDBMSで考えましたがRDBMSに限らずDBMS（SQL、NoSQL）のアーキテクチャを理解した深度に比例してCPUサイジングも確度は上がると思います。	
					
					
## DBサーバでのメモリの役割とサイジング
メモリはDBサーバの重要なリソースです。DBサーバではOSで利用するメモリ(ページキャッシュ、バッファキャッシュなど)＋若干の余裕以外全てDBの利用するメモリとして適用するのが一般的です。

メモリに限りませんが、DBサーバは安定したアクセス性能、障害の際問題の切り分けの容易性なども考慮し出来るだけ他のプロダクトと併用せず、DBサーバ単体でサーバ運用するのが一般的です。

### 共有メモリ、プロセス固有又はスレッド固有メモリ
一般的なRDBMSではインスタンス内の全てのスレッドで共有するメモリ、プロセスもしくはスレッド固有のメモリの大きく2種類のメモリがあります。

#### 共有メモリ
テーブルデータ、インデックスデータ、SQLの実行結果、プロダクトによってはSQLの実行計画などをキャッシュし各セッションから利用するために確保されるメモリ領域です。

#### プロセス固有、スレッド固有メモリ
DBへの接続単位もしくはセッション単位で処理する際に確保されるメモリ領域です。	
### メモリサイジング
DBサーバの物理メモリを30Gbyte、サービス開始からデータ量は線形に増加し1ヶ月あたり750Mbyte、インデックスは250Mbyte増加し、プロセス固有（スレッド固有）は一律10Mbyte必要とします。

- 同時接続数を100とした場合、プロセス固有（スレッド固有）メモリはどのくらい必要でしょう。
  - どのくらいの同時接続数を賄うことが可能でしょう。

- ヒット率による性能予測。データを問合せメモリ上にデータがある場合ヒットとし無い場合はDiskに問い合わせに行くとする。メモリから応答するコストを1、Diskを100とし、ヒット率100％、95％、80％でコストについて考えてみましょう。
  - メモリ上にデータがあることのメリットを考えてみましょう。
 
### 講義時の備忘録用キーワード
ヒット率、I/O性能差（メモリとストレージのI/O差は一般的には10万～1000万と言われている）、LRU、プロセス(スレッド)固有、共有メモリ、圧縮
					
### まとめ
確保したメモリ内に、アプリケーションなどから要求されるデータが収まることが大事な戦略となります。メモリを有効活用する方法を様々な視点から考えてみましょう。

					
## DBサーバでのDiskの役割とサイジング
Disk(ストレージ)は永続的なデータを保存するために利用します。Diskはメモリと比較しRead/Write共に性能が低いためI/Oのネックになる箇所でもあります。メモリに出来るだけ、要求頻度の高いデータをキャッシュしDiskアクセスを減らす工夫が重要になります。

これまでのDBサーバのボトルネックはこのDiskとのI/Oをいかに効率的に行うかが重要でした。また、キャパシティプランの視点でもDisk容量が溢れた場合、新規データが格納できないため問題となります。今後もDBサーバにおいてはDiskに対しては様々な角度で対策をする必要があります。

### Diskサイジング	
テーブル users　ID(int)、NAME(varchar 10byte)、PROFILE(varchar 255byte)、PROFILE_IMAGE(blob 平均10kbyte)でサイジングしてみましょう

- 100万人が会員登録した場合、テーブル users はどのくらいの容量になるでしょう？

同様にメッセージアプリを想定し、ユーザがメッセージを投稿するテーブル messages　ID(int)、USER_ID(int)、MESSAGE(varchar 255byte)、POST_DATETIME(datetime)でサイジングしてみましょう

- 1日に50万ユーザが100メッセージを投稿すると、どのくらいの容量になるか。
  - 1年後はどのくらいの容量になるか。 

(最近は重要度は高くないが)データを格納するRDBMSの領域特性を理解し(ブロック、ページ、エクステント、セグメント)サイジングの精度を上げましょう。

(最近は重要度は高くないが)DBサーバの満たすべき機能(障害耐性、復旧時間)やサービスレベルなどによる冗長化などによるサイズ増についても考えてみましょう
      	
## I/O特性
ここではRDBMSを前提としたDBサーバの挙動をIOバウンド(対義語 CPUバウンド)の観点から考えてみましょう。

IOバウンドとは、CPU性能を使い切る前に、Disk読み込み、書込みの処理量が頭打ちとなり実行できる処理(SQL)も頭打ちに達します。

- I/Oバウンドにて処理が頭打ちになる状況を具体的に考えてみましょう
- もしDBサーバがCPUバウンドに達する状況が存在する場合、どのような状況があるか考えてみましょう。
				
### 講義時の備忘録用キーワード
BBWC、塩漬け、iowait、RAID、SSD、ioDrive、圧縮、WAL、格納効率、フラグメンテーション
					
### まとめ
今回はRDBMSで考えましたがRDBMSに限らず永続性を保障するDBMS（SQL、NoSQL）についてはどれでもDiskI/Oが性能の一番ネックとなる箇所に違いはありません。OS、選択したDBMSの機能を用いてI/Oの最適化や最小化は常に意識していきましょう。		
					
## 1章まとめ
DBサーバの性能を取得する方法、取得した情報からDBサーバの状況を把握、考察し、サイジングが妥当か、将来の必要となるリソースについて予測ができるようになりましょう。		
# 2章 DBサーバのシステム要件、運用保守

DBサーバの性能計測などから賄える処理量やサイジングなどの情報が得られます。その情報などからサービスの稼働状況を考慮し、ある程度近い将来に対する、拡張などの計画の策定が行えると思います。しかしサービスを継続させ続けるには単純に各種サイジング内に処理が賄えるだけでは不十分です。そこでこの章ではサービスを継続させ続ける上で性能観点以外を中心に必要な観点について触れて行きたいと思います。

## RAS
以下の項目をDBサーバに照らし合わせて考えて行きましょう。

- 信頼性（Reliability）、可用性（availability）、保守性（Serviceability）

信頼性はハードウェアやソフトウェアが故障する頻度が少なく故障している期間が短いこと、可用性は利用者が使用できること、保守性は保守しやすいことを指す。（Wiki）

### 可用率(稼働率)(例 99.9%(スリーナイン)99.999%(ファイブナイン))
主にサーバ、ネットワークのH/Wの選定をする上での指標やクラウドサービスを選定する際にも指標として使用します。

- 可用率と年間ダウンタイム

|可用率|24時間/日|8時間/日|	
|:-:|:-:|:-:|	
|90%|876時間(36.5日)|291.2時間(12.13日)|
|95%|438時間(18.25日)|145.6時間(6.07日)|
|99%|87.6時間(3.65日)|29.12時間(1.21日)|
|99.90%|8.76時間|2.91時間|
|99.99%|52.56分|17.47分|
|99.999%|5.256分|1.747分|
|99.9999%|31.536秒|10.483秒	|

### 単一障害点(SPOF)
サービス(事業)継続を考慮すると、単一障害点(SPOF)での障害によるサービス停止は出来る限り避けたいところです。そこで単一障害点(SPOF)の対策について考えてみましょう。

単一障害点(SPOF)とはその単一箇所が障害となると、システム全体が障害となるような箇所を指す。（Wiki)

例えば、DBサーバの電源が突然落ちり、CPU破損のようなH/Wの障害でDBサーバの機能が止まりサービスが止まるケースなどは単一障害点(SPOF)による障害です。掛けられる予算やリソースの中でサービスの継続性も考慮し、サーバ構成、アプリケーション構成などを作成することはとても大事です。  

※機能が止まる部分については一瞬、一定時間と前段の可用率なども含め考えましょう。

単一障害点(SPOF)の回避をするには様々なレイヤでの冗長化が一般的です。ここからは冗長化により単一障害点(SPOF)を対策しサービスを継続し続ける方法について考えていきましょう。今回はDBサーバのみに絞っていきます。

#### サーバ
サーバはHAクラスタ技術により冗長化するのが一般的です。LinuxでメジャーなHAクラスタのHeartbeatではローカルDisk間をブロックデバイスレベルでの同期を行うDRBDとの組合わせが一般的です。商用ではSANストレージを用いた共有Disk構成も広く利用されています。HA構成例ではACTIVE-STANDBY構成を例に上げていますが、ACTIVE-ACTIVE構成も要件により利用されます。
![テキスト](./png/ha.png)

#### Disk	(ストレージ)
DiskはRAID技術により冗長するのが一般的です。又、RAIDコントローラーを搭載したSANやNASなどの共有ストレージを併用することも多くあります。
![テキスト](./png/disk.png)

#### Filesystem

ビッグデータに特化したFileSystem(例 HDFS)ではデータノード上にファイル単位での冗長化(デフォルト3多重)を行う仕組みがあります。

#### DB

DBではサーバ、Disk、Filesystemレベルでの複製だけでは対策が不十分な場合、複製データを作る仕組みを用いて対策します。

  - MySQL（レプリケーション）  
  大規模なDBサーバを構築する上でこの機能について熟知しておくことは必須です。マスタースレーブ構成やマルチマスター構成など柔軟性もあります。但し完全な(Atomicな)同期を保障していないので非同期なデータ運用が前提となります。

![テキスト](./png/db.png)

#### その他

Mysqlクラスタやmemcachedのレプリケーション、オンメモリでサーバ間のDBを同期し可用性とプロダクトによっては性能向上も担う。DBプロダクトにより性能は大きく左右されるRDBMSの場合は線形に性能はスケールすることは無く、NoSQLはこの機能を強みに持つプロダクトが多いです。

最後にアプリケーションの作りこみによる冗長化（複数へ書込み）や単一障害点（SPOF）があることを認識した上でコストなどのトレードオフを踏まえて冗長化をしないと言う選択肢もあります。

## キャパシティプランニング
DBサーバに対して想定以上のデータが格納されDiskFULLとなったり、想定以上のアクセスにより処理が滞留、応答の遅延が発生することもあります。このような状況に対して対処ができずDBサーバの機能を継続することが出来ないことは容認できるでしょうか。またサービスの視点から無停止で対処が必要かRASの項で学んだ可用率の範囲内で対処を行うか段階を設けた視点が重要です。以下のケースについて考えてみましょう。

### 同時実行数が増加した場合
明らかなボトルネック(例 ORマッパーによる不必要な複数要求、アプリケーションのバグ)が見当たらず応答性能(レスポンスタイム、スループットなどを指標とする)の劣化が顕著と判断した場合にサーバの増設（スケールアウト）、もしくはサーバ内の増強が可能でリソースの枯渇している対象が判明している場合は対象のCPU、メモリ、ストレージの増強（スケールアップ）を行う。

### 格納できるDisk容量の限界	
Disk容量の拡張による性能劣化が起きないと判断した場合、Diskの追加を行う。拡張が不可能、又は拡張により性能劣化の可能性があると判断した場合サーバの増設（スケールアウト）を行う。	

### 緩やかな参照、書込み量の増加による処理速度の遅延
参照、書込み量がじょじょに増加し続けI/Oネックが顕著に現れた場合（I/Oバウンドが更に偏った場合）メモリの増設を行う。メモリの増設が行えない、又はメモリの増設だけでは改善しないと判断した場合サーバの増設（スケールアウト）を行う。

## 障害（復旧が必要な状況）
ここまでで単一障害点の対策や拡張性による対策を洗い出してきました。しかし障害が起きない可能性がゼロになったわけではありません。ここまでを踏まえて何かしらの障害により復旧が必要になった場合にどのような対策が行えるか考えていきましょう。

### クラッシュリカバリ（自動）
RDBMSのトランザクションの機能を用いた回復方法です。サーバやDBシステムが予期せぬダウンが発生した場合に次回起動時にDB上のデータの状態を正常に戻す機能です。

### バックアップリストア（手動）
DBMSの機能やfilesystemのスナップショット機能、その両方を組合わせたりすることでDBシステムを別に退避したバックアップから戻す方法です。

![テキスト](./png/backup.png)

- データの特性から復旧をせず再作成で行うケースもあると思います。どのようなケースでしょうか。
- 手動、自動に限らず復旧が必要となる状況を想定してみましょう。（例　NoSQLの系切り替え、RDBMSの系切り替え）
- スケールアップにてDBを拡張した場合、それまでの復旧方法は通用するか考えてみましょう。
													
## 2章まとめ
サービスを継続する上で回避するべきリスク、対処するべき施策、その結果を踏まえたH/W、DB、アプリケーションの構成を自分なりに考えてみましょう。															
# 第3章 サービスの成長とあわせたDBの拡張戦略

サービスの成長に合わせDBサーバもサービスの成長に合わせた拡張戦略が求められます。そこで求められる戦略とは何でしょう。この章ではサービスの特性による稼動傾向の特徴を見つけ出し今後サービスが成長した際にDBサーバではどういう対応が必要か、もしくはどのように備えて行くかについて考えていきます。又、第1章、第2章で学んだ知識を用いてどういう施策が継続的に必要かについても考えていきます。

## 拡張戦略
サイトが成長した際にDBサーバで取れる戦略にどのようなものがあるでしょうか。ここでは大きく二つの戦略スケールアップ、スケールアウトについて学んでいきます。

### スケールアップ
サーバの性能を機器の追加（CPU、メモリ、Disk）、もしくは更に高機能なサーバに置き換えることでサーバ単体で処理性能を向上させる手法です。現在のIT業界ではオンプレミスな環境ではあまり現実的ではありませんがクラウド環境ではスケールアップの選択枝は広く見られます。オンプレミスの環境に関してはサーバのグレードと価格が同じ形で比例しておらず高機能化に伴い価格は指数的に上がり更にスケールできる限界性能に達しやすいため一定のスケールアップまでは選択肢に入りますが爆発的なスケールを求められる場合早期に破綻します。メリットとしてはスケールアップ時にアプリケーションの改修コストはほとんど掛からない、もしくは全く掛からないケースが多いことでしょう。注意点としては安価なスケールアップ（CPU,メモリの追加）が可能で効果が見込める場合はこの限りではなく、後述するスケールアップの前に検討する価値は十分にあります。

![テキスト](./png/scaleup.png)

### スケールアウト
同一機能を有したサーバの追加(垂直分散)や機能ごとにサーバ分割(水平分散)をしていきシステム全体で処理性能を向上させる手法です。現在のIT業界ではスケールアウト戦略が基本となっています。理論上は同一機能を有したサーバを追加し続ければスケールできる処理限界には達しないと言えます。	スケールアウト戦略を取れるようにするには予めスケールアウトした際にやりとりが発生するアプリケーション、サーバ等が接続参照する先をスケール前と変わらずシームレスに行えることが重要です。DBサーバについてもスケールアウトが取れるDB構造にしておくことが将来の拡張戦略で大事となります。

![テキスト](./png/scaleout.png)

#### 垂直分散
機能（Webサーバ、アプリケーションサーバ、DBサーバ、オプティマイザ、DB writer/reader）もしくはエンティティごとに分割する。

![テキスト](./png/3章垂直分散.png)

#### 水平分散	
同一機能を有したサーバを並べる(Webサーバ)、データを任意のキーの単位で分割する。分割単位のキーで閉じた処理を行う。(4章sharding で学びます。)

![テキスト](./png/3章水平分散.png)

#### 垂直、水平の混合(読み込み、書き込みの特性により分散、拡張)
オリジナルと複製(レプリカ)と言う構成とし複製の数を増やしていく。基本的にはオリジナルに書込みオリジナルから複製(レプリカ)にデータを伝播させ複製(レプリカ)を参照専用とする。(2章で触れたmysql レプリケーションがこれにあたります。4章で改めて学びます。)

- 注意：垂直、水平による解説はイメージをしやすくするために用いています。

### Think
- サービスを停止せず稼働させながら拡張したい場合どのような拡張戦略が適しているでしょうか。逆にサービスを停止して拡張する場合サービスを停止するデメリットの代わりにどのようなメリットがあるでしょうか。

## サービスの稼働状況
私たちが提供するサービスは人を相手に行うサービスです。サービスを運営するにあたってサービスの稼動状況はサービスの内容により特徴を帯びてきます。ここではサービス全体からDBサーバにフォーカスをあてて稼動(負荷)状況についてユーザの行動の傾向とシステム側で行う日々の運用の両面から考えていきましょう。

### トップとボトム
サービスを稼働すると様々な要因からリソースに利用状況が山の頂上（トップ）と麓（ボトム）として見られることが多いです。

以下はVOYAGE GROUP内のサービスのDBサーバのロードアベレージの推移グラフ例です。2週間弱と短い期間のグラフですがグラフの形は1日単位で繰り返しているところに特徴を見出だせます。今回例に出したDBのロードアベレージのトップの部分は「日次のバックアップの時間帯」です。

![テキスト](./png/la.png)

このグラフの場合トップはロードアベレージ３近くに、ボトムは1未満となっています。この稼動状況に対する評価も当然必要ですが日々の稼動状況を監視し続けることによりいち早く傾向の違いを見出すことやトップとボトムを見つけ出し処理限界について意識を置くことも重要です。

#### Think

- なぜ「日次のバックアップの時間帯」の負荷は高くなるのでしょう。
  - 負荷を軽減するにはどうしたら良いでしょう。
- その他の時間帯の負荷をどう評価(妥当、低い、高い)するか考えてみましょう。
  
### 通年を通した性能観測と性能予測
サービスを運営する際に1年の中で特異な稼動状況を表すときがあります。その中でも通年を通じて毎年同じ傾向になるものもあります。例えば母の日限定でアクセスがアップするサイトのような特異日が限定されるケースです。期間を1年、四季、1ヶ月、1週間、1日でどのような傾向が見られるか考えてみましょう。またその際にユーザの行動に依存するものと、運用する側の都合で発生するシステム依存の二つの観点で考えていきましょう。		

#### ユーザのアクセスが集中するケース
例えば有名なニュースサイトに露出した、TVで取り上げられた時などは一時的にアクセスが集中します。人気商品の発売開始時もアクセスが集中します(この場合はさらに同じアイテムにアクセスが集中します)

#### システムによるアクセス(処理や負荷)が集中するケース
例えば1日の売り上げを集計する処理(バッチ処理)、前述したバックアップ処理などシステムの都合によりアクセス(処理や負荷)が集中するケースがあります。

#### キーワード
月初バッチ、バックアップ、サッカーのゴールシーン、スラッシュドット現象、バルス！！

## オンライン処理、バッチ処理
サービスの傾向やしシステム上必要な処理についても触れました。DBサーバの稼動について別な切り口、オンライン処理とバッチ処理について触れていきましょう。

### オンライン処理(OLTP)
ユーザとのやりとりは基本これに該当します。目標のレスポンスタイムを重視し快適なサービスを提供することが至上命題となります。目標のレスポンスタイムを維持、もしくは更に向上させるにはどういう方法については第4章で行います。

### バッチ処理
システムにより全てもしくは一部のデータを用いて集計等の結果を作成する処理を指します。オンライン処理で重視されたレスポンスタイムではなくスループットを重視するのが一般的です。

#### Think

- レスポンスタイム、スループットの違いについて考えてみましょう。
- オンライン処理にてDBサーバに対するレスポンスタイムの内訳について考えてみましょう。
- バッチ処理によるDBサーバへの影響について考えてみましょう。
  - 処理時間は何に依存するか考えてみてください。
  - バッチ処理が動くことによりオンライン処理にはどのように影響があるか考えてみてください。
- オンライン処理、バッチ処理を行いつづけ拡張戦略を行う場合、どのような拡張戦略が良いか考えてみましょう。

## 3章まとめ
サービスの成長とあわせDBサーバの成長戦略はどのようなものがあるでしょうか。また成長戦略を考える際に意識する部分はどういうところでしょうか。又サービスの成長を踏まえスタートアップ時に予め織り込める要素についても考えてみましょう。		

# 第4章 DBサーバのI/O戦略
DBサーバの特徴は一般的にI/Oバウンドで性能限界に達しやすく、I/Oがボトルネックとして支配的です。その点を踏まえI/Oバウンドに対する対策をI/Oの最適化、最小化、局所化、拡張の観点でDBMSの機能を用いたアプローチ、H/WレベルでのDBの分割(スケールアウト)、アプリケーションの作りこみによるアプローチ等、様々な実践的なアプローチについて、この章では学びます。

## INDEX　探索経路の最適化（最小化）
RDBMSではテーブル探索の機能としてB+tree,B*tree索引は(ほぼ)必ず実装されています。INDEXはRDBMSでのI/O戦略を行う上で、とても重要な機能です。

![テキスト](./png/4章btree.png)

B+tree索引の構造は簡単にあらわすと、このような図になります。二分木と違い、ブロックデバイスを前提に、格納効率の最大化をしており、探索時の計算量はO(logFN*log2F)となります。図中ではINDEXから探索しデータにアクセス(ルックアップ)されるるまでに5ページ(ブロック)のアクセスで実現されています。

例として、データが100万件で容量を1Gbyteのテーブルがあるとします。全データを走査し1件のデータを求めた場合、計算量はO(n)になります。この場合100万分の1(もしくはn)件のデータを1Gbyte全てアクセスして導き出します。いわゆるFull Scanの状態です。INDEXでこのテーブルを探索した場合、root,branch,leafを探索しleafから紐づくデータのアクセスで完了します。図中では5ページ(ブロック)のアクセスで導き出せています。この場合、root,branch,leaf,データ部のページサイズを8Kbyteとした場合、40Kbyteのアクセスで探索が完了します。

この例での100万件から1件(もしくはn件)のデータが導きだされるケースでFull ScanとINDEX探索では1Gbyte対40byteのデータ容量の走査の差が性能差となって現れます。

### Think
このB+tree構造から性能的に有利な探索条件について考えてみましょう。比較対象はFull Scan、計算量O(1)となるKey Valueで考えてみましょう。

- 一意検索
- 非一意検索
- 範囲検索
- 構造の特徴を用いたその他用法	
				

## 複製（レプリカ）の作成(垂直分散)
Webサイトでは参照と挿入(更新,削除)などの処理では比較的参照処理が多くなる傾向があります。その参照処理に対して2章で学んだレプリケーション機能を用いて参照専用の複製したDBを作成しDBサーバシステム全体で参照時の性能向上をはかる際に用います。

挿入(更新,削除)機能をMasterに、参照機能をSlaveに分けるので読み、書きで垂直分散(垂直分割)するケースに該当します。

レプリカの発展系で下図のマスタースレーブと呼ばれる構成以外にマルチマスターと言う相互から更新ができるものもありますが、扱いが難しく一般的ではないため今回は触れません。

![テキスト](./png/4章レプリカ.png)

### Think
Master,Slaveの複製機能を用いたメリットやアプリケーション側で注意すべき点などについて考えてみましょう。

- 参照性能はどのようにスケールするでしょう？	
- アプリケーションから参照先はどのように制御するべきでしょう？
- オンライン、バッチ処理の両方の観点でI/O性能はどう向上するでしょう  

	

## 圧縮
I/Oバウンドになる理由としてメモリ上に展開できない永続化されたデータの参照や挿入更新削除などをDiskにアクセスすることでread/writeがDBサーバ上で一番ボトルネックとして顕在化することは触れました。

これを解消するために一度にDiskから転送できるデータ量を増やそうと言う考えで圧縮を使用します。圧縮、解凍(が必要なタイプには)によりCPUリソースは無圧縮時より使用されますが、I/OバウンドによるCPUの余裕から(かつ複数コア化、高性能化)十分トレードオフを踏また実践投入が検討出来るレベルになっています。

圧縮はH/Wレベルで行っているものやRDBMSプロダクトの機能と多様です。MySQLでは圧縮オプションがありデータを1/2に圧縮することがVOYAGE GROUPではサービスにて実践され実証されています。

![テキスト](./png/4章圧縮.png)

### Think
圧縮機能をオンライン処理、バッチ処理の二つで考えてみましょう。

- オンライン処理時の圧縮のメリット、デメリット
- バッチ処理時の圧縮のメリット、デメリット
				
## パーティショニング
RDBでは表(テーブル)でデータは表現されます。この表に対し、RDBMSの機能で一定のルール(レンジ、ハッシュなど)で内部的に分割管理しデータを格納する方法をパーティショニングと呼びます。

SQLなどでDBに問い合わせる際は表指定だけで問い合わせることが出来、表面上は通常の表のアクセスと変わりはありません。（特定のRDBMSでは内部分割した特定のパーティションを指定できるものもあります。）INDEXについてもテーブルと同様にパーティション管理が行えるRDBMSも存在します。以下の例は日付(年次)をレンジとしたパーティション分割した表です。

![テキスト](./png/4章パーティション.png)

### Think
パーティショニングによりI/O性能を向上させることができる状況について考えてみましょう。

- データの挿入、削除
- パーティションキーを含めた検索（パーティションプルーニング）
- 索引の衝突				

## sharding（物理パーティション）
データを任意のルールで分割する技術です。書籍等によっては物理パーティションと呼ばれるものもあります。例えばVOYAGE GROUPではユーザを軸に分割することを多用しています。DBMSによってはパーティショニングとの違いが無い場合もありますが、本講義ではshardingは開発者が任意のルールで、かつルールに沿って各物理サーバにデータ配置されたデータのみで一定水準のサービスを行うことが出来ると言う定義で進めます。

![テキスト](./png/4章sharding.png)

### Think
shardingによるメリット、デメリットについて考えてみましょう。

- ユーザ全体であるデータの平均値や最大値、最小値を求める場合どのようなアプリケーション設計が必要でしょうか。
- 複製（レプリカ）と同様にアプリケーションから参照先はどのように制御するべきでしょうか。

## キャッシュ
主にRDBMSを選択した際に取る戦略となります。SQLを用いてRDBMSに都度問い合わせるコストが重い場合、KeyValueで問合せ可能なキャッシュとして予めデータを配置しSQLのコストを軽減する戦略です。

## その他
最近ではSSDも一般的になり、過去に比べメモリとDiskのI/O性能の差が縮まってきています。更に高速なストレージ製品なども登場していることもあり、費用対効果が見込める場合、高速なストレージ製品の導入によりI/Oネックを解消するケースもあります。

またI/O戦略の文脈でサーバ、ノード、ストレージなどを全体でI/O量の平準化させる戦略と、局所局所で対策するケースにわかれます。ここ最近は前者の平準化の戦略をベースに組み立てることがセオリーです。

### 余談 列指向データベース
ここまでは主にオンライン処理(プラス業務上同じデータベースでの処理が望ましいバッチ処理)のI/O戦略について触れていきましたが、近年サービスにて生み出された全て(大量)のデータを用いたデータ解析のアプローチも増えてきています。この場合自前で大量のデータを様々なディメンションで解析が行える解析基盤を持つかGoogle BigQuery、Amazon Redshiftなどのクラウド基盤を用いるなどのアプローチもあります。どちらを用いるにせよ一定のサービス規模を超えた場合は業務処理とは切り離した解析基盤を用意することが望ましい領域です。
			
				
## 4章まとめ
この章では様々なI/O負荷に対する戦略に触れてきました。一つの戦略を選択するのでなくこれらの戦略を組み合わせていくことでI/O負荷をコントロールし安定したサービスの稼働と継続して将来の拡張戦略に繋げていくことが重要です。

サービスが成長するにつれI/O負荷に対して都度対策することは避けれませんが、サービス開始時に低コストで予め仕込める戦略は無いか考えてみてください。
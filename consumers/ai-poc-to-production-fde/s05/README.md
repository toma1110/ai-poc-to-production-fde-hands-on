# ログを人とAIで調べる

共有注文アプリの同じ実行ログを、まず自分で調べ、その後にAmazon Bedrockの回答と比べます。AIには一度のログ検索だけを許し、今回の実行記録にあるJSONログ1件だけを返します。ログにない原因や影響は、分からないまま区別します。

AIの回答は調査の補助です。運用上の判断や障害原因の確定には使いません。

## 始める前に

共有アプリの[固定版README](https://github.com/toma1110/aws-observability-sample-app/blob/v1.0.2/README.md)を先頭から読み、そこに書かれた前提確認、AWSへの配置、注文の実行、後片付けを行います。このREADMEは共有アプリの手順を置き換えません。

講座で案内されたこの演習リポジトリの固定コミットから、S05の資料を取得します。共有アプリを配置したCloudShellで、共有アプリのリポジトリ直下から次を実行します。

```bash
git clone https://github.com/toma1110/ai-poc-to-production-fde-hands-on.git ../ai-poc-to-production-fde-hands-on
EXERCISE_COMMIT='講座のリソースURLに含まれる40桁のコミットID'
git -C ../ai-poc-to-production-fde-hands-on archive "$EXERCISE_COMMIT" consumers/ai-poc-to-production-fde/s05 | tar -x -C .
test -f consumers/ai-poc-to-production-fde/s05/investigate.py
git rev-parse HEAD
```

最後に表示される共有アプリのコミットが `3a3599121537bb83a9b7afd42162a8635aeee59b`（`v1.0.2`）と一致することを確認します。

BedrockでAI調査を行う場合は、AWSアカウントで`amazon.nova-lite-v1:0`への推論が許可され、`ap-northeast-1`のBedrock Runtimeを呼べる必要があります。権限がない場合は、自分で権限を広げず、管理者に確認してください。AI調査をしない場合も、先にログを人が読んで記録するところまでは試せます。

推論は最大350出力トークンです。このS05資料フォルダーでの実行は、異なるrequest IDを使う通常調査4件まで（各2回、合計8回）と、架空ログの動作確認1回（合計2回）に分かれ、合計10回を上限にします。request IDごとの2回目は同じtool resultを使った最終回答です。一度使ったrequest IDや動作確認枠は再実行できません。APIエラーや想定と異なるtool要求が返った場合も、その回数は消費され、自動再試行しません。上限に達した場合やエラーが出た場合は追加実行を止めて状態を確認してください。`api-usage.json`を削除・初期化して上限を戻さないでください。

## 実行結果を準備する

固定版アプリを実行した同じCloudShellで、共有アプリのリポジトリ直下に移動します。次のファイルとフォルダーがあることを確認してください。

```text
run-evidence.txt
consumers/ai-poc-to-production-fde/s05/
```

`run-evidence.txt`は、共有アプリの`run_scenarios.py`が今回実際に出力したものを使います。別の実行のログを混ぜたり、JSON例を実行記録として作ったりしないでください。

## 先に自分で調べる

S05の資料フォルダーへ移動します。

```bash
cd consumers/ai-poc-to-production-fde/s05
```

`run-evidence.txt`からJSONログを一つ選び、表示された`requestId`を使ってS05の資料フォルダーで次を実行します。

```bash
export REQUEST_ID='run-evidence.txtに表示されたrequestId'
python3 investigate.py --show-log --request-id "$REQUEST_ID"
```

`REQUEST_ID`には今回の`run-evidence.txt`に表示された実際の値を入れてください。画面にはそのIDの`request_complete` JSONログ1件だけが表示されます。HTTP応答や注文入力を根拠に加えず、そのJSONに記録された事実と、ログだけでは分からないことを先に記録します。アプリを実行する時点では操作名や入力条件を知るため、完全に情報を伏せた比較ではありません。その前提を記録し、原因についての判断に入力条件など他の情報を使った場合は「追加で確認した証拠」として分けてください。

人の調査にかかった時間は、調べ始める前と終わった後に、実際の時刻とタイムゾーンを記録してください。たとえば`date --iso-8601=seconds`はCloudShellの時刻とタイムゾーンを表示します。時刻や結論を後から推測して埋めないでください。

テンプレートをコピーして編集します。

```bash
cp human-investigation.template.json human-investigation.json
nano human-investigation.json
```

`request_id`と`evidence_request_ids`には表示された実際の`requestId`を入れ、`query`にはそのJSONだけに答える問いを書きます。`started_at`と`completed_at`にはタイムゾーン付きの実時刻を入れ、`findings`と`uncertainty`には自分の調査結果と残る不明点を書きます。Bedrockを呼ぶ前にこの記録を保存してください。人の調査内容やシナリオ名はBedrockへ送信されず、AI toolには同じJSONログ1件と同じ問いだけが渡ります。

### 教材用の想定例

任意の想定例は、実測した人の調査の代わりにはなりません。まず自分でログを調べ、実際の回答と時刻を`human-investigation.json`へ保存し、その実測記録を使ってAI調査を行ってください。教材用の例を確認するときは、別名の`hypothetical-investigation.example.json`を直接表示します。この例を`human-investigation.json`へコピーすると実測記録が置き換わり、続けてAI調査を実行した場合も想定例が使われるため、コピーしないでください。`investigate.py`には別ファイルを指定する引数はありません。

```bash
python3 -m json.tool hypothetical-investigation.example.json
```

このコマンドは別名の例を表示するだけで、調査CLIの入力や実測記録を変更しません。`investigate.py --validate-only`は現在の`human-investigation.json`と実行ログの対応を確認するコマンドです。実測記録を保存した後にこれを実行し、AI調査も同じ`actual`記録を使ってください。例の`started_at`/`completed_at`は`null`です。架空の日時や実測時間欄を作らず、想定所要時間は`illustrative_duration`へ「教材用の想定」と明記して分けます。例では「3分程度」としています。想定時間と実測値から短縮率を出すことはできません。

記録が現在の実行ログと対応するかを確認します。このコマンドはBedrockを呼びません。

```bash
python3 investigate.py --validate-only
```

`requestId`が違う場合は停止し、人の記録と実行ログを確認してください。

## AIの調査を行う

人の記録を保存し、対応する実行ログを確認した後に限り、次を実行します。

```bash
python3 investigate.py
```

AIは必須の`search_run_logs` toolを一度呼び、選んだ`requestId`のログだけを受け取ります。ファイルを開いたりAWSを操作したりするtoolはありません。AIが別のtoolや別の入力を要求したときは停止します。

回答は`ai-investigation-<requestId>.json`に保存されます。このファイルには、人の記録、対象ログ、AI回答、各Converse応答の利用量、API応答時間が含まれます。API応答時間は人が回答を確認・修正した時間ではありません。

AIの回答を確認するときは、実際に確認を始めた時刻と終えた時刻を記録し、回答を残した点・修正した点、追加で見た証拠、不確実なまま残った点を記入します。

```bash
cp ai-review.template.json ai-review.json
nano ai-review.json
```

この確認記録と結果JSONは自分の学習記録です。公開リポジトリへ追加せず、必要な比較情報を後段の演習に使う場合は、自分の実測記録として保管してください。実ログや実アカウント情報を公開しないでください。

所要時間は次の3つを分けて記録します。実測modeの人の調査時間は`human-investigation.json`の開始・完了時刻、BedrockのAPI応答時間は`ai-investigation-*.json`、AI回答を人が確認・修正した時間は`ai-review.json`の開始・完了時刻から得ます。想定modeには人の調査時間がありません。API応答時間だけを人の作業時間や短縮時間と読み替えず、想定例と実測値から短縮率を出しません。必要なら「AI支援の経過時間」をAPI応答時間と人の確認時間の合計として併記できますが、どの区間を足したか明記し、準備時間を含まない経過時間として扱います。

## Converseのtool-useだけを試す場合

人の比較記録や共有アプリの実行ログを使わず、架空ログでtool-useを一度だけ確認するには、次を使います。

```bash
python3 investigate.py --smoke-test-only
```

このモードは`human-investigation.json`と`run-evidence.txt`を読みません。2回のConverse呼出しだけを使い、結果を`smoke-test-only.json`へ分けて保存します。これは動作確認用の架空ログで、共有アプリの実行結果や人とAIの比較データには含めません。動作確認はこのS05資料フォルダーで1回だけ実行できます。実行後は同じローカル呼出し予算を消費するため、`api-usage.json`の残りを確認してください。

## うまくいかない場合

- `No recognized completed app JSON logs`が出たら、共有アプリが今回出力した`run-evidence.txt`を確認します。
- 選んだログが見つからない場合は、別runの値で補わず、記録したrunとrequest IDを確認します。
- AWSの権限エラー、model IDエラー、tool要求の不一致が出たら、繰り返し実行せずエラーと利用量ファイルを確認します。
- 通常調査8回、動作確認2回、request IDごとの2回、または動作確認の1回制限に達した場合は停止します。`api-usage.json`を消して上限を戻さないでください。

## 記録を保存して後段へ進む

S06の評価やS09の提案で使う記録は、共有アプリを削除する前にCloudShellの外へ保存します。S05の資料フォルダーから共有アプリのリポジトリ直下へ戻り、今回の実行用フォルダーとZIPを一意なUTC日時で作ります。既存の同名フォルダーまたはZIPがあれば上書きせず停止し、別の日時でやり直します。

```bash
cd ../../..
pwd
REQUEST_ID='今回AI調査に使った実際のrequestId'
RUN_ID="$(date -u +%Y%m%dT%H%M%SZ)"
RUN_ROOT="$HOME/s05-records"
RUN_DIR="$RUN_ROOT/$RUN_ID"
ZIP_PATH="$RUN_ROOT/$RUN_ID.zip"
if [ -e "$RUN_DIR" ] || [ -e "$ZIP_PATH" ]; then
  echo "同名の保存先があるため停止しました。既存の内容は上書きしないでください。"
elif mkdir -p "$RUN_ROOT" && mkdir "$RUN_DIR"; then
  if cp run-evidence.txt "$RUN_DIR/" && cp consumers/ai-poc-to-production-fde/s05/human-investigation.json \
    "$RUN_DIR/"; then
    ARCHIVE_FILES=(run-evidence.txt human-investigation.json)
    COPY_OK=true
    for FILE_NAME in "ai-investigation-$REQUEST_ID.json" ai-review.json api-usage.json smoke-test-only.json; do
      SOURCE="consumers/ai-poc-to-production-fde/s05/$FILE_NAME"
      if [ -f "$SOURCE" ]; then
        if cp "$SOURCE" "$RUN_DIR/"; then
          ARCHIVE_FILES+=("$FILE_NAME")
        else
          COPY_OK=false
        fi
      fi
    done
    if [ "$COPY_OK" = true ]; then
      (cd "$RUN_DIR" && python3 -m zipfile -c "$ZIP_PATH" "${ARCHIVE_FILES[@]}") && echo "保存先を作成しました: $RUN_ID"
    else
      echo "今回の記録をすべて保存できなかったため停止しました。"
    fi
  else
    echo "今回の記録を保存できなかったため停止しました。"
  fi
else
  echo "保存先を安全に新規作成できなかったため停止しました。"
fi
```

`REQUEST_ID`の値は説明文ではなく今回AI調査に使った実際のrequest IDに置き換えます。人の調査だけを記録した場合も、`run-evidence.txt`と`human-investigation.json`を保存できます。AI調査や架空ログの動作確認を実行した場合は、作成された結果と利用量のファイルも保存され、実測記録と区別されます。`保存先を作成しました`と表示された場合だけ続けます。同名フォルダーまたはZIPがある、保存に失敗したという表示が出た場合は停止し、`RUN_ID`を別の現在時刻に変えてこの手順を最初からやり直してください。既存の保存先は再利用しません。`RUN_ID`、`RUN_DIR`、`ZIP_PATH`を確認し、保存ファイル名とサイズ、各JSONのrequest ID・実行値・利用量が今回の記録と一致することを確かめます。ZIPの中身は`python3 -m zipfile -l "$ZIP_PATH"`で一覧表示し、`run-evidence.txt`と今回作成された各JSONが入っていることを確認してください。

CloudShellで`realpath "$ZIP_PATH"`を実行し、表示された絶対パスをコピーします。**Actions → Download file** のファイル名欄には、`$HOME`のような変数ではなく、コピーした実際の絶対パスを入力して受講者PCへダウンロードします。PCの `ドキュメント\FDE-course-records\S05\<RUN_ID>` など自分の学習記録用フォルダーへ展開し、`run-evidence.txt`と各JSONが開けて同じrequest IDを含むことを確認します。S06とS09ではこの保存版を参照します。S05資料の取得先やCloudShell上の一時作業フォルダーは記録の保管先にしないでください。

AI調査が未完了で後から同じ実行を再開する場合は、CloudShellの同じ作業場所とS05資料フォルダーを使い、残っている`api-usage.json`を保ったまま続けます。すでに上限に達した枠や一度使ったrequest IDを再実行しません。`api-usage.json`を削除・初期化して同じ実行の回数を戻すこともできません。再開前に、残り回数と保存済みの人の記録を確認してください。

## 後片付け

S05資料を使ったログ調査とBedrock呼出しは追加の永続AWSリソースを作成しません。Bedrock Runtimeの呼出し料金は発生します。共有アプリのLambdaとCloudWatch Logsは共有アプリのstackが存在する間残るため、別料金が発生します。演習後は共有アプリREADMEの手順でstackを削除し、CloudWatch Logsや実行中のリソースが残っていないことを確認してください。

PCへ保存したファイルを開いて内容を確認するまでは、CloudShell上の記録を削除しないでください。後段のS06評価とS09提案で使い終えた後、CloudShellで今回の実行記録を削除できます。まだ同じS05実行を再開する可能性がある間は、`api-usage.json`と作業ファイルを残し、同じ実行をやり直さないでください。`api-usage.json`は同じ実行の回数を戻すために削除・初期化しないでください。

```bash
REQUEST_ID='今回AI調査に使った実際のrequestId'
RUN_ID='上の保存で表示された実際のRUN_ID'
RUN_ROOT="$HOME/s05-records"
RUN_DIR="$RUN_ROOT/$RUN_ID"
ZIP_PATH="$RUN_ROOT/$RUN_ID.zip"
if [ -d "$RUN_DIR" ] && [ -f "$ZIP_PATH" ]; then
  if ls -la "$RUN_DIR" && python3 -m zipfile -l "$ZIP_PATH"; then
    rm -f "$RUN_DIR/run-evidence.txt" \
      "$RUN_DIR/human-investigation.json" \
      "$RUN_DIR/ai-investigation-$REQUEST_ID.json" \
      "$RUN_DIR/ai-review.json" \
      "$RUN_DIR/api-usage.json" \
      "$RUN_DIR/smoke-test-only.json" &&
    rmdir "$RUN_DIR" &&
    rm -f "$ZIP_PATH" &&
    rm -f consumers/ai-poc-to-production-fde/s05/human-investigation.json \
      "consumers/ai-poc-to-production-fde/s05/ai-investigation-$REQUEST_ID.json" \
      consumers/ai-poc-to-production-fde/s05/ai-review.json \
      consumers/ai-poc-to-production-fde/s05/smoke-test-only.json \
      consumers/ai-poc-to-production-fde/s05/api-usage.json \
      run-evidence.txt &&
    ls consumers/ai-poc-to-production-fde/s05 &&
    test ! -e run-evidence.txt
  else
    echo "保存内容を確認できないため停止しました。"
  fi
else
  echo "指定した専用フォルダーまたはZIPが見つからないため停止しました。"
fi
```

`REQUEST_ID`と`RUN_ID`には今回の実際の値を入れ、まず`pwd`で共有アプリのリポジトリ直下にいることを確かめてください。S06とS09で必要な記録をPC側で使い終えた後に限り、この手順を実行します。ZIP一覧と`RUN_DIR`の中身を確認し、今回の保存物だけであることを確かめてから実行してください。最初に`RUN_DIR`内の既知のファイルだけを個別に削除し、`rmdir`で空になったことを確認します。`rmdir`に失敗した場合は中身を調べ、残ったファイルを確認せず削除しないでください。続いてZIPとS05の今回の実行記録を個別に削除し、`ls`と`test`で残存がないことを確認します。ここでの`api-usage.json`削除は後片付けだけに使い、利用回数を戻す目的では行いません。共有アプリのstackとAWSリソースは、別途、共有アプリREADMEの手順で削除・残存確認してください。

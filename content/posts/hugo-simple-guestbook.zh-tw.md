---
title: "Hugo 靜態部落格簡易留言板實作"
date: 2026-04-24T23:06:00+08:00
lastmod: 2026-09-15T11:49:00+08:00
author: "黃宏勝"
categories:
  - 科技
tags:
  - Hugo
  - 靜態網站
  - 留言板
  - GitHub Gist
  - Google Apps Script
  - Cloudflare Turnstile
  - Akismet
  - Google 表單
keywords: 
  - 科技
  - Hugo
  - 靜態網站
  - 留言板
  - GitHub Gist
  - Google Apps Script
  - Cloudflare Turnstile
  - Akismet
  - Google 表單
summary: "由於利用 Cloudflare Worker 架設 Hugo 靜態部落格沒有後台伺服器可以儲存，所以我們利用 Google Apps Script、GitHub Gist 以及 Google 表單來建立一個簡易的留言系統"
description: "由於利用 Cloudflare Worker 架設 Hugo 靜態部落格沒有後台伺服器可以儲存，所以我們利用 Google Apps Script、GitHub Gist 以及 Google 表單來建立一個簡易的留言系統"
slug: "hugo-simple-guestbook"
featureimage: "https://media.secologies.com/Hugo%20%E9%9D%9C%E6%85%8B%E9%83%A8%E8%90%BD%E6%A0%BC%E7%B0%A1%E6%98%93%E7%95%99%E8%A8%80%E6%9D%BF%E5%AF%A6%E4%BD%9C%E5%B0%81%E9%9D%A2.webp"
---
![留言板封面](https://media.secologies.com/Hugo%20%E9%9D%9C%E6%85%8B%E9%83%A8%E8%90%BD%E6%A0%BC%E7%B0%A1%E6%98%93%E7%95%99%E8%A8%80%E6%9D%BF%E5%AF%A6%E4%BD%9C%E5%B0%81%E9%9D%A2.webp)

## 動機
對於要讓讀者在部落格裡留言或者只能寄信討論問題的選擇一直環繞在我的腦海中，以前寄信問過 JN[^1] 我們到底需不需要，也看過 Wiwi[^2] 對於留言區的看法，所以我對於留言板的看法一直拿捏不定

最早我用 Hugo 內建的 Disqus 討論系統，但這必須引入外部的 JavaScript 模組進來，所以會拖慢部落格的載入速度，甚至 Ivon[^3] 提到 Disqus 有可能會看頁面的流量來決定開廣告，我想對於讀者來說體驗會非常糟

之後又看到了像是 ikuka[^4] 還有 YangBear[^5] 等人對於留言與寄信之間的差異看法，其中試著降低讀者的交流門檻也許不失為一個好想法，就像 Alex[^6] 說的一樣，當成一個小實驗來做做看其實不會有什麼損失對吧？

所以我又再次思考要怎麼做這個留言板，這次我希望不再倚賴 Disqus 系統，參考 JN 之前探索過的路途，Giscus 強制使用者登入 Github 帳號對讀者也不太友善，再來就是一些自架的服務，像是 Isso, Remark42, Cusdis 還有目前有些人正在用的 Artalk

但最大的問題應該是我的部落格是部署成 Cloudflare Pages 的靜態網站，底層沒有任何機器給我 ssh 進去，這導致我沒辦法利用 Artalk 的 Docker 檔無痛安裝設定

這讓我必須思考其他路徑，而我之前看到皮皮[^7]有針對這種情況做一種不需要後端伺服器機器也可以實現的簡易留言簿，LQ7[^8] 跟他的助理(?)也都同意這個方法確實可行，而且照皮皮的版本二，還可以提升效率，對於靜態網站來說簡直是摸蜊仔兼洗褲

除此之外，我再加上 Cloudflare Turnstile 做流量驗證以及 Akismet 防釣魚留言來多幾道保障，故此文想記錄一下流程

## 架構

{{< mermaid >}}
stateDiagram-v2
    s1 : 讀者
    s2 : Github Gist JSON
    s3 : Google Apps Script
    s4 : Cloudflare Turnstile
    s5 : Akismet
    s6 : Google 表單 
    s7 : 留言失敗

    s1 --> s2: 1. 送出留言內容
    s2 --> s3: 2. POSTs method 到 Google 中繼服務
    s3 --> s4: 3. 確認讀者的 turnstile token 是否合法
    s4 --> s3: 4-1. 真實人物 token 正確
    s4 --> s7: 4-2. token 是空的
    s3 --> s5: 5. 檢查留言<br>內容是否包含垃圾訊息
    s5 --> s3: 6. 確認完內容後回傳
    s3 --> s6: 7. 留言內容標記 pending 送進 Google 表單裡
    s5 --> s6: 7-1. 若包含垃圾留言則標記為 spam
    s6 --> s3: 8. 站長回傳留言審查結果
    s3 --> s2: 9. 回傳通過審查的留言 
    s2 --> s1: 10. 更新留言板 
{{< /mermaid >}}

### 平台
作業系統：macOS Sequoia 15.7.3

瀏覽器：Brave v1.89.141

### Hugo 版本
```bash
hugo v0.154.1-e2fd6764be86d0cde988a7de6334fda0f43de871 darwin/arm64 BuildDate=2026-01-01T17:32:20Z VendorInfo=gohugoio
```

### 模板版本
```bash
Blowfish v2.102.0
```

## 限制
{{< alert >}}
由於此方法我只在 Blowfish 模板上測試，如果讀者的模板是不同的，請先參考自己模板的 class 階層做調整
{{< /alert >}}

{{< alert >}}
如果讀者不需要 Cloudflare Turnstile 以及 Akismet 功能，可以跳過這兩個設定步驟，並且修改 Google Apps Script 專案以及留言內嵌的 Cloudflare 驗證區塊
{{< /alert >}}

{{< alert >}}
由於我們採用 Google 表單紀錄的方法，所以留言回覆時沒有任何 SMTP 伺服器可以像 Artalk 那樣寄送信件通知站長已回覆，需要請讀者自行確認，並且因為我們將內容存放在 GitHub Gist 的隱密檔案內，所以 Google 搜尋引擎無法收錄留言內容，代表其他人無法透過搜尋引擎找到該留言，留言無法提升 SEO 效果，但可以在部落格內利用 <code> 留言搜尋頁面 </code> 搜尋到所有留言內容
{{< /alert >}}

## 設置 Google 表單
因為我們架設靜態網站沒有後端資料庫，而且我將其部署在 Cloudflare Workers 上，所以也沒有底層的 Linux 機器有空間給我儲存，而我們的想法就是將所有留言內容儲存在 Google Sheet 裡，各位可以選一個帳號底下建立一個新的表單，並且在第一列輸入留言標籤，下列是我設定的所有標籤

```bash
A: id | B: postKey | C: name | D: email | E: content | F: date | G: website | H: reply | I: approved
```

![Comments Google Sheet](https://media.secologies.com/Google%20Sheet%20Comments.webp)

請記得把表單的 ID 記錄到空白的文字檔內，之後會用到，Google Sheet ID 可以從網址內找到：<code> https://docs.google.com/spreadsheets/d/你的Google Sheet ID/edit?gid=0#gid=0 </code>

## 建立 Github Gist JSON
由於我們要將 Github Gist 當作快取空間，所以讀者請先建立一個 [Github](https://github.com/) 帳號，登入後請到 [Github Gist](https://gist.github.com/) 服務上，一進去應該就可以建立新的一組 json 檔，檔案名稱可以自訂，但之後會用到所以請取個有意義的名字，並且在檔案內容填寫下列空符號，以便未來可以存放留言內容進去，請注意建立時選擇預設的隱密 gist

```bash
{}
```

![Github Gist JSON](https://media.secologies.com/Github%20Gist%20JSON.webp)

創造完檔案後，我們點進檔案本身，將 Gist ID 記錄下來，Gist ID 從網址上就可以看到，找出 <code> gist.github.com/你的 Github 帳戶名/GIST_ID </code>，同樣存放在空格文字檔內，之後會用到

再來還必須建立一個 GitHub Personal Access Token 開出權限用來給其他服務寫入該 Gist 檔案，我們先回到 [Github](https://github.com/)，點擊右上角的頭貼出現下拉式選單，選擇 Settings

![GithubSettings](https://media.secologies.com/GithubSettings.webp)

從左側欄位最下面找到 Developer settings

![GithubDeveloperSettings](https://media.secologies.com/GithubDeveloperSettings.webp)

然後一樣在左側欄位最下面找到 Personal access tokens，下拉選單選擇 Tokens (classic) 即可

![GithubPersonalAccessToken](https://media.secologies.com/GithubPersonalAccessToken.webp)

進到創建頁面，從右上角看到 Generate new token，按下拉選單選擇 Generate new token (classic) 就行

![GithubGenerateNewToken](https://media.secologies.com/GithubGenerateNewToken.webp)

該 Token 名稱可以自行輸入，並且選擇有效期限，最重要的是往下找到 gist 選項勾選創建後即可

![GithubSelectGist](https://media.secologies.com/GithubSelectGist.webp)

建立完成後請將 Token 內容一樣存在空的文字檔內，後續也會用到

## Cloudflare Turnstile
再來我們先把 Cloudflare Turnstile 用來驗證的 Token 跟 Key 申請好，首先要申請好 [Cloudflare](https://www.cloudflare.com/) 帳號，如果讀者的網域是在 Cloudflare 上買的應該就會很熟悉他們家的介面，意思是我們要進去後台 [Dashboard](https://dash.cloudflare.com/)，直接從左側搜尋 Turnstile 會比較快

![Turnstile](https://media.secologies.com/Turnstile.webp)

到 Turnstile 頁面後新增 Widgets，中間的 Add widget

![TurnstileAddWidget](https://media.secologies.com/TurnstileAddWidget.webp)

填選該 Widget 的內容，名稱自訂，新增網域名稱，如果是在 Cloudflare 上買的可以直接找，如果是外部的就請自行新增，模式選擇預設的管理模式，並且驗證方法我預設每一次都要進行驗證，不管過去是否已經驗證過了，確認完之後點選建立

![TurnstileWidgetSetting](https://media.secologies.com/TurnstileWidgetSetting.webp)

建立完之後應該會出現一個頁面顯示你的 Site Key 跟 Secret Key，請一樣將兩把 Key 都記錄在文字檔裡，之後還會用到，確認完後關閉會跳轉回 Turnstile widgets 頁面就可以看到方才建立的配置

![TurnstileWidgetResult](https://media.secologies.com/TurnstileWidgetResult.webp)

## Akismet
接著我們到 [Akismet](https://akismet.com/) 申請一組 API Key，基本上只要部落格或網站沒盈利他們就提供免費版的垃圾留言檢測服務給申請者，流程就照個官方的說明走就行，雖然會出現結帳畫面但個人版的申請不會要求輸入任何信用卡資訊

![Akismet](https://media.secologies.com/Akismet.webp)

記得一樣把 Akismet API key 記錄在文字檔案

## Google Apps Script 設定
再來我們要設定最重要的中繼服務配置，利用 Google Apps Script 轉發讀者留言、GitHub Gist 以及 Google Sheet 之間的聯繫，首先進到後台頁面 [https://script.google.com/](https://script.google.com/)，請注意 Google 帳號是不是跟之前建立 Google Sheet 的是同一個，在同一個帳號底下，我們建立一個新專案

![GoogleAppsScriptAddNewProject](https://media.secologies.com/GoogleAppsScriptAddNewProject.webp)

進到新專案內後可以貼上我最後設計的版本如下，我在全域宣告的變數內容請通通換成方才要讀者儲存的所有資訊

```bash
const SHEET_ID = '你的 Google Sheet ID';
const GIST_ID = '你的 GitHub Gist ID';
const GITHUB_TOKEN = '你的 GitHub Token';
const TURNSTILE_SECRET = '你的 Turnstile Secret Key'
const GIST_FILENAME = '你的 GitHub Gist 檔案名稱';
const AKISMET_KEY  = '你的 Akismet API Key';
const AKISMET_BLOG = '你的網域名稱';

function doPost(e) {
  const params = JSON.parse(e.postData.contents);

  // Verify Turnstile token first
  if (!verifyTurnstile(params.turnstileToken)) {
    return ContentService
      .createTextOutput(JSON.stringify({ success: false, error: 'Human verification failed' }))
      .setMimeType(ContentService.MimeType.JSON);
  }

  // Check Akismet
  const isSpam = checkAkismet(params.name, params.content);

  const sheet = SpreadsheetApp.openById(SHEET_ID).getActiveSheet();
  const id    = Utilities.getUuid();
  const now   = new Date().toISOString();

  sheet.appendRow([
    id,                          // A: id
    params.postKey,              // B: postKey
    params.name,                 // C: name
    params.email || '',          // D: email
    params.content,              // E: content
    now,                         // F: date
    params.website || '',        // G: website
    '',                          // H: reply (empty, you fill this later)
    isSpam ? 'spam' : 'pending'  // I: approved
]);

  return ContentService
    .createTextOutput(JSON.stringify({ success: true }))
    .setMimeType(ContentService.MimeType.JSON);
}

function doGet(e) {
  // Allow CORS preflight
  return ContentService
    .createTextOutput('OK')
    .setMimeType(ContentService.MimeType.TEXT);
}

function doOptions(e) {
  return ContentService
    .createTextOutput('')
    .setMimeType(ContentService.MimeType.TEXT);
}

function verifyTurnstile(token) {
  if (!token) return false;
  
  const response = UrlFetchApp.fetch('https://challenges.cloudflare.com/turnstile/v0/siteverify', {
    method: 'POST',
    headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
    payload: `secret=${TURNSTILE_SECRET}&response=${token}`
  });

  const result = JSON.parse(response.getContentText());
  return result.success === true;
}

function checkAkismet(name, content) {
  try {
    const response = UrlFetchApp.fetch(
      `https://${AKISMET_KEY}.rest.akismet.com/1.1/comment-check`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
      payload: [
        `blog=${AKISMET_BLOG}`,
        `comment_author=${encodeURIComponent(name)}`,
        `comment_content=${encodeURIComponent(content)}`,
        `comment_type=comment`
      ].join('&')
    });
    return response.getContentText() === 'true';
  } catch(err) {
    return false; // if Akismet fails, don't block the comment
  }
}
function syncToGist() {
  const sheet = SpreadsheetApp.openById(SHEET_ID).getActiveSheet();
  const rows  = sheet.getDataRange().getValues();
  const data  = rows.slice(1);

  const comments = {};
  data.forEach(row => {
    const obj = {
      id:       row[0],
      postKey:  row[1],
      name:     row[2],
      email:    row[3],
      content:  row[4],
      date:     row[5],
      website:  row[6],
      reply:    row[7],
      approved: row[8],
    };

    if (obj.approved !== true && obj.approved !== 'approved') return;

    const key = obj.postKey;
    if (!comments[key]) comments[key] = [];
    comments[key].push({
      id:      obj.id,
      name:    obj.name,
      content: String(obj.content).replace(/\r\n/g, '\n').replace(/\r/g, '\n'),
      date:    obj.date,
      website: obj.website || '',
      reply:   String(obj.reply || '').replace(/\r\n/g, '\n').replace(/\r/g, '\n')
    });
  });

  // ✅ Sort comments by date ascending (oldest first)
  Object.keys(comments).forEach(key => {
    comments[key].sort((a, b) => new Date(a.date) - new Date(b.date));
  });

  const payload = JSON.stringify({
    files: {
      [GIST_FILENAME]: {
        content: JSON.stringify(comments)
      }
    }
  });

  UrlFetchApp.fetch(`https://api.github.com/gists/${GIST_ID}`, {
    method: 'PATCH',
    headers: {
      'Authorization': `token ${GITHUB_TOKEN}`,
      'Content-Type': 'application/json'
    },
    payload: payload
  });
}
```

而我們要試著部署這份 Script，在右上角可以找到 Deploy 的按鈕，點擊 <code> New deployment </code>

![NewDepolyment](https://media.secologies.com/NewDepolyment.webp)

選擇目前我們這個 Web app，執行人員選擇目前建立 Script 的帳號，而存取權限請選擇所有人，因為我們要讓任何人都可以留言

![NewDepolymentSetting](https://media.secologies.com/NewDepolymentSetting.webp)

按下部署後就會完成該次紀錄，會出現一個 Web app 的網址我們一樣儲存起來放進文字檔案裡，之後還會用到

![DeploymentID](https://media.secologies.com/DeploymentID.webp)

再來我們還要設定自動同步的條件，回到頁面中左側欄位有個碼錶的功能是觸發器

![Trigger](https://media.secologies.com/Trigger.webp)

點選後進到設定，新增一組觸發條件每 5 分鐘執行一次 <code> syncToGist </code> 的函式，確認完時間後就可以按下建立

![TriggerSettings](https://media.secologies.com/TriggerSettings.webp)

至此 Google Apps Script 中繼站這樣就設定完成了

## Hugo 留言區/留言板設定
在我的 Blowfish 模板上我設定每一篇文章都可以留言，也同時設立獨立的留言板，首先我在全域配置設定需要的參數，從 <code>config/_default/params.toml</code> 檔案新增

```bash
[comments]
enabled = true
gistId = "你的 GitHub Gist ID"
gistUsername = "你的 GitHub 用戶名"
appsScriptUrl = "你的 Apps Script Web App URL"
ownerName = "你的名字"
turnstileSiteKey = "你的 Turnstile Site Key"
```

### 文章留言區

接著 <code>layouts/partials/comments.html</code> 設定留言區系統

```bash
{{ $postKey := .RelPermalink }}

<div class="artalk-comments mt-12">
  <h2 class="text-xl font-semibold mb-6 text-neutral-800 dark:text-neutral-200">
    留言
  </h2>

  <script src="https://challenges.cloudflare.com/turnstile/v0/api.js" async defer></script>

  <!-- Comment List -->
  <div id="comment-list" class="space-y-4 mb-8"></div>

  <!-- Comment Form -->
  <div class="bg-neutral-50 dark:bg-neutral-800 rounded-xl p-6">
    <h3 class="text-lg font-medium mb-4 text-neutral-700 dark:text-neutral-300">發表留言</h3>
    <div class="space-y-3">
      <input id="c-name" type="text" placeholder="您的名稱 *"
        class="w-full px-4 py-2 rounded-lg border border-neutral-300 dark:border-neutral-600 bg-white dark:bg-neutral-700 text-neutral-800 dark:text-neutral-200 focus:outline-none focus:ring-2 focus:ring-primary-500" />
      <input id="c-website" type="url" placeholder="您的網站 (選填)"
  class="w-full px-4 py-2 rounded-lg border border-neutral-300 dark:border-neutral-600 bg-white dark:bg-neutral-700 text-neutral-800 dark:text-neutral-200 focus:outline-none focus:ring-2 focus:ring-primary-500" />
      <input id="c-email" type="email" placeholder="電子郵件 (選填且不公開)"
        class="w-full px-4 py-2 rounded-lg border border-neutral-300 dark:border-neutral-600 bg-white dark:bg-neutral-700 text-neutral-800 dark:text-neutral-200 focus:outline-none focus:ring-2 focus:ring-primary-500" />
      <textarea id="c-content" rows="4" placeholder="留言內容 *"
        class="w-full px-4 py-2 rounded-lg border border-neutral-300 dark:border-neutral-600 bg-white dark:bg-neutral-700 text-neutral-800 dark:text-neutral-200 focus:outline-none focus:ring-2 focus:ring-primary-500"></textarea>
      <div class="cf-turnstile" data-sitekey="{{ site.Params.comments.turnstileSiteKey }}"></div>
      <button id="c-submit"
        onclick="submitComment('{{ $postKey }}')"
        class="px-6 py-2 bg-primary-600 hover:bg-primary-700 text-white rounded-lg font-medium transition-colors">
        送出留言
      </button>
      <p id="c-msg" class="text-sm text-neutral-500 dark:text-neutral-400 hidden"></p>
    </div>
  </div>
</div>

<script>
const GIST_USERNAME = '{{ site.Params.comments.gistUsername }}';
const GIST_ID      = '{{ site.Params.comments.gistId }}';
const APPS_SCRIPT  = '{{ site.Params.comments.appsScriptUrl }}';
const POST_KEY     = '{{ $postKey }}';
const OWNER_NAME   = '{{ site.Params.comments.ownerName }}';

// Load comments from Gist
async function loadComments() {
  const el = document.getElementById('comment-list');
  try {
    const res  = await fetch(`https://gist.githubusercontent.com/${GIST_USERNAME}/${GIST_ID}/raw/comments.json?t=${Date.now()}`);
    const all  = await res.json();
    const list = (all[POST_KEY] || []).sort((a, b) => new Date(a.date) - new Date(b.date));

    if (list.length === 0) {
      el.innerHTML = '<p class="text-neutral-400 dark:text-neutral-500 text-sm">還沒有留言，來第一個留言吧！</p>';
      return;
    }
    el.innerHTML = list.map(c => `
  <div class="bg-neutral-100 dark:bg-neutral-900 border border-neutral-300 dark:border-neutral-700 rounded-xl p-5 mb-4">
    <div class="flex items-center justify-between mb-3">
      <div class="flex items-center gap-2">
        ${c.website
          ? `<a href="${escHtml(normalizeUrl(c.website))}" target="_blank" rel="noopener noreferrer"
                class="font-medium text-neutral-800 dark:text-neutral-100 hover:text-primary-500 dark:hover:text-primary-400 transition-colors duration-200">${escHtml(c.name)}</a>`
          : `<span class="font-medium text-neutral-800 dark:text-neutral-100">${escHtml(c.name)}</span>`
        }
      </div>
      <span class="text-xs text-neutral-500 dark:text-neutral-400">${new Date(c.date).toLocaleDateString('zh-TW', { year:'numeric', month:'2-digit', day:'2-digit', hour:'2-digit', minute:'2-digit' })}</span>
    </div>
    <p class="text-neutral-700 dark:text-neutral-200 text-base leading-relaxed">${formatContent(c.content)}</p>
    ${c.reply ? `
    <div class="mt-4 bg-neutral-200 dark:bg-neutral-800 rounded-lg p-4">
      <div class="flex items-center gap-2 mb-2">
	<span class="font-medium text-neutral-800 dark:text-neutral-100">${escHtml(OWNER_NAME)}</span>
	<span class="text-xs bg-primary-600 dark:bg-primary-700 text-white px-2 py-0.5 rounded">站長</span>
        <span class="text-xs text-neutral-500 dark:text-neutral-400 ml-auto">${new Date(c.date).toLocaleDateString('zh-TW', { year:'numeric', month:'2-digit', day:'2-digit', hour:'2-digit', minute:'2-digit' })}</span>
      </div>
      <p class="text-neutral-700 dark:text-neutral-200 text-base leading-relaxed">${formatContent(c.reply)}</p>
    </div>` : ''}
  </div>
`).join('');
  } catch(err) {
    el.innerHTML = '<p class="text-red-400 text-sm">載入留言失敗</p>';
  }
}

async function submitComment(postKey) {
  const name    = document.getElementById('c-name').value.trim();
  const website = document.getElementById('c-website').value.trim();
  const email   = document.getElementById('c-email').value.trim();
  const content = document.getElementById('c-content').value.trim();
  const msg     = document.getElementById('c-msg');
  const btn     = document.getElementById('c-submit');

  // Get Turnstile token
  const token = document.querySelector('[name="cf-turnstile-response"]')?.value;
  if (!token) {
	  showMsg('請完成人機驗證', 'error');
	  return;
  }

  if (!name || !content) {
    showMsg('請填寫名稱和留言內容', 'error');
    return;
  }

  btn.disabled = true;
  btn.textContent = '送出中...';

  try {
    await fetch(APPS_SCRIPT, {
      method: 'POST',
      mode: 'no-cors',
      body: JSON.stringify({ postKey, name, email, content, website, turnstileToken: token }),
    });
    showMsg('留言已送出，審核後將顯示。感謝您！', 'success');
    document.getElementById('c-name').value    = '';
    document.getElementById('c-website').value = '';
    document.getElementById('c-email').value   = '';
    document.getElementById('c-content').value = '';
  } catch(err) {
    showMsg('送出失敗，請稍後再試', 'error');
  } finally {
    btn.disabled = false;
    btn.textContent = '送出留言';
  }
}

function showMsg(text, type) {
  const el = document.getElementById('c-msg');
  el.textContent = text;
  el.className = `text-sm mt-2 ${type === 'error' ? 'text-red-500' : 'text-green-500'}`;
  el.classList.remove('hidden');
}

function escHtml(str) {
  return String(str).replace(/&/g,'&amp;').replace(/</g,'&lt;').replace(/>/g,'&gt;').replace(/"/g,'&quot;');
}

function normalizeUrl(url) {
  if (!url) return '';
  if (!/^https?:\/\//i.test(url)) {
    return 'https://' + url;
  }
  return url;
}

function formatContent(str) {
  const mdLink   = new RegExp('\\[([^\\]]+)\\]\\((https?:\\/\\/[^\\)]+)\\)', 'g');
  const bareUrl  = new RegExp('(?<!href=")(https?:\\/\\/[^\\s<]+)', 'g');

  return escHtml(str)
    .replace(mdLink,  '<a href="$2" target="_blank" rel="noopener noreferrer" class="text-primary-500 hover:underline">$1</a>')
    .replace(bareUrl, '<a href="$1" target="_blank" rel="noopener noreferrer" class="text-primary-500 hover:underline">$1</a>')
    .replace(/\n/g, '<br>');
}

loadComments();
</script>
```

### 文章留言計算
再來我想在每一張文章卡片加上留言數量，我先複製模板裡的 basic.html 到我自己的 layouts 裡

```bash
cp themes/blowfish/layouts/partials/article-meta/basic.html layouts/partials/article-meta/basic.html
```

接著我把計算的模組加在閱讀時間之後

```bash
{{ if and (.Params.showReadingTime | default (.Site.Params.article.showReadingTime | default true)) (ne .ReadingTime 0) }}
  {{ $meta.Add "partials" (slice (partial "meta/reading-time.html" .)) }}
{{ end }}

//需要新增的是底下的內容

{{/* Comment count */}}
{{ if ne .Params.showComments false }}
  {{ $commentBadge := printf `<span class="comment-count text-xs text-neutral-500 dark:text-neutral-400 flex items-center gap-1" data-postkey="%s">💬 <span class="count-num">...</span></span>` .RelPermalink }}
  {{ $meta.Add "partials" (slice $commentBadge) }}
{{ end }}
```

嘗試利用 JavaScript 動態的去抓取，所以我建立 <code> layouts/partials/comment-counts.html </code> 檔案

```bash
<script>
(async () => {
  const GIST_USERNAME = '{{ site.Params.comments.gistUsername }}';
  const GIST_ID       = '{{ site.Params.comments.gistId }}';

  try {
	  const res = await fetch(`https://gist.githubusercontent.com/${GIST_USERNAME}/${GIST_ID}/raw/comments.json?t=${Date.now()}`);
	  const all = await res.json();
	  document.querySelectorAll('.comment-count').forEach(el => {
		  const key   = el.dataset.postkey;
		  const count = (all[key] || []).length;
		  el.querySelector('.count-num').textContent = count;
	  });
  } catch(e) {
	  document.querySelectorAll('.count-num').forEach(el => el.textContent = '0');
  }
})();
</script>
```

為了要改變文章的內容，我複製卡片的基本檔案 

```bash
cp themes/blowfish/layouts/_default/baseof.html layouts/_default/baseof.html
```

在 body 結束之前引用計算留言數量的模組

```bash
  </div>
    {{ partial "comment-counts.html" . }}
  </body>
```

這樣我們就在每一篇文章底下都新增留言區塊

### 留言板
跟許多 Hugo 模板建立獨立頁面的方法一樣，新增新的頁面 <code> content/guestbook/index.zh-tw.md </code>

```bash
---
title: "留言板"
description: "歡迎留言！"
---
```

然後我們嘗試為這個留言板的頁面獨立做設計，建立 <code> layouts/guestbook/single.html </code> 檔案

```bash
{{ define "main" }}
<div class="max-w-2xl mx-auto px-4 py-8">

  <script src="https://challenges.cloudflare.com/turnstile/v0/api.js" async defer></script>

  <p class="text-neutral-500 dark:text-neutral-400 mb-8">
    留言板功能已開放，無需登入、不限字數，站內留言板塊皆有審核機制，包括留言審查、Cloudflare Turnstile 驗證機制與 Akismet Anti-Spam 功能，請注意留言內容
  </p>

  <!-- Comment Form -->
  <div class="bg-neutral-100 dark:bg-neutral-900 border border-neutral-300 dark:border-neutral-700 rounded-xl p-6 mb-10">
    <h3 class="text-lg font-medium mb-4 text-neutral-700 dark:text-neutral-300">發表留言</h3>
    <div class="space-y-3">
      <input id="c-name" type="text" placeholder="您的名稱 *"
        class="w-full px-4 py-2 rounded-lg border border-neutral-300 dark:border-neutral-600 bg-white dark:bg-neutral-700 text-neutral-800 dark:text-neutral-200 focus:outline-none focus:ring-2 focus:ring-primary-500" />
      <input id="c-website" type="url" placeholder="您的網站 (選填)"
        class="w-full px-4 py-2 rounded-lg border border-neutral-300 dark:border-neutral-600 bg-white dark:bg-neutral-700 text-neutral-800 dark:text-neutral-200 focus:outline-none focus:ring-2 focus:ring-primary-500" />
      <input id="c-email" type="email" placeholder="電子郵件 (選填且不公開)"
        class="w-full px-4 py-2 rounded-lg border border-neutral-300 dark:border-neutral-600 bg-white dark:bg-neutral-700 text-neutral-800 dark:text-neutral-200 focus:outline-none focus:ring-2 focus:ring-primary-500" />
      <textarea id="c-content" rows="4" placeholder="留言內容 *"
        class="w-full px-4 py-2 rounded-lg border border-neutral-300 dark:border-neutral-600 bg-white dark:bg-neutral-700 text-neutral-800 dark:text-neutral-200 focus:outline-none focus:ring-2 focus:ring-primary-500"></textarea>
      <div class="cf-turnstile" data-sitekey="{{ site.Params.comments.turnstileSiteKey }}"></div>
      <button id="c-submit"
        onclick="submitComment()"
        class="px-6 py-2 bg-primary-600 hover:bg-primary-700 text-white rounded-xl font-medium transition-colors">
        送出留言
      </button>
      <p id="c-msg" class="text-sm mt-2 hidden"></p>
    </div>
  </div>

  <!-- Comment List -->
  <h3 class="text-lg font-medium mb-4 text-neutral-700 dark:text-neutral-300">
    所有留言 / <span id="c-count">0</span> 則
  </h3>
  <div id="comment-list" class="space-y-4">
    <p class="text-neutral-400 text-sm">載入中...</p>
  </div>

</div>

<script>
const GIST_USERNAME = '{{ site.Params.comments.gistUsername }}';
const GIST_ID       = '{{ site.Params.comments.gistId }}';
const APPS_SCRIPT   = '{{ site.Params.comments.appsScriptUrl }}';
const POST_KEY      = '/guestbook/';  // fixed key for guestbook
const OWNER_NAME    = '{{ site.Params.comments.ownerName }}';

async function loadComments() {
  const el = document.getElementById('comment-list');
  try {
    const res  = await fetch(`https://gist.githubusercontent.com/${GIST_USERNAME}/${GIST_ID}/raw/comments.json?t=${Date.now()}`);
    const all  = await res.json();
    const list = (all[POST_KEY] || []).sort((a, b) => new Date(a.date) - new Date(b.date));

    document.getElementById('c-count').textContent = list.length;

    if (list.length === 0) {
      el.innerHTML = '<p class="text-neutral-400 dark:text-neutral-500 text-sm">還沒有留言，來第一個留言吧！</p>';
      return;
    }

    el.innerHTML = list.map(c => `
      <div class="bg-neutral-100 dark:bg-neutral-900 border border-neutral-300 dark:border-neutral-700 rounded-xl p-5 mb-4">
        <div class="flex items-center justify-between mb-3">
          <div class="flex items-center gap-2">
            ${c.website
              ? `<a href="${escHtml(normalizeUrl(c.website))}" target="_blank" rel="noopener noreferrer"
                    class="font-medium text-neutral-800 dark:text-neutral-100 hover:text-primary-500 dark:hover:text-primary-400 transition-colors duration-200">${escHtml(c.name)}</a>`
              : `<span class="font-medium text-neutral-800 dark:text-neutral-100">${escHtml(c.name)}</span>`
            }
          </div>
          <span class="text-xs text-neutral-500 dark:text-neutral-400">${new Date(c.date).toLocaleDateString('zh-TW', { year:'numeric', month:'2-digit', day:'2-digit', hour:'2-digit', minute:'2-digit' })}</span>
        </div>
        <p class="text-neutral-700 dark:text-neutral-200 text-base leading-relaxed">${formatContent(c.content)}</p>
        ${c.reply ? `
        <div class="mt-4 bg-neutral-200 dark:bg-neutral-800 rounded-lg p-4">
          <div class="flex items-center gap-2 mb-2">
            <span class="font-medium text-neutral-800 dark:text-neutral-100">${escHtml(OWNER_NAME)}</span>
            <span class="text-xs bg-primary-600 dark:bg-primary-700 text-white px-2 py-0.5 rounded">站長</span>
          </div>
          <p class="text-neutral-700 dark:text-neutral-200 text-base leading-relaxed">${formatContent(c.reply)}</p>
        </div>` : ''}
      </div>
    `).join('');
  } catch(err) {
    el.innerHTML = '<p class="text-red-400 text-sm">載入留言失敗</p>';
  }
}

async function submitComment() {
  const name    = document.getElementById('c-name').value.trim();
  const website = document.getElementById('c-website').value.trim();
  const email   = document.getElementById('c-email').value.trim();
  const content = document.getElementById('c-content').value.trim();
  const btn     = document.getElementById('c-submit');

  // Get Turnstile token
  const token = document.querySelector('[name="cf-turnstile-response"]')?.value;
  if (!token) {
	  showMsg('請完成人機驗證', 'error');
	  return;
  }

  if (!name || !content) {
    showMsg('請填寫名稱和留言內容', 'error');
    return;
  }

  btn.disabled = true;
  btn.textContent = '送出中...';

  try {
    await fetch(APPS_SCRIPT, {
      method: 'POST',
      body: JSON.stringify({ postKey: POST_KEY, name, email, content, website, turnstileToken: token }),
    });
    showMsg('留言已送出，審核後將顯示。感謝您！', 'success');
    document.getElementById('c-name').value    = '';
    document.getElementById('c-website').value = '';
    document.getElementById('c-email').value   = '';
    document.getElementById('c-content').value = '';
  } catch(err) {
    showMsg('送出失敗，請稍後再試', 'error');
  } finally {
    btn.disabled = false;
    btn.textContent = '送出留言';
  }
}

function showMsg(text, type) {
  const el = document.getElementById('c-msg');
  el.textContent = text;
  el.className = `text-sm mt-2 ${type === 'error' ? 'text-red-500' : 'text-green-500'}`;
  el.classList.remove('hidden');
}

function escHtml(str) {
  return String(str).replace(/&/g,'&amp;').replace(/</g,'&lt;').replace(/>/g,'&gt;').replace(/"/g,'&quot;');
}

function normalizeUrl(url) {
  if (!url) return '';
  if (!/^https?:\/\//i.test(url)) {
    return 'https://' + url;
  }
  return url;
}

function formatContent(str) {
  const mdLink   = new RegExp('\\[([^\\]]+)\\]\\((https?:\\/\\/[^\\)]+)\\)', 'g');
  const bareUrl  = new RegExp('(?<!href=")(https?:\\/\\/[^\\s<]+)', 'g');

  return escHtml(str)
    .replace(mdLink,  '<a href="$2" target="_blank" rel="noopener noreferrer" class="text-primary-500 hover:underline">$1</a>')
    .replace(bareUrl, '<a href="$1" target="_blank" rel="noopener noreferrer" class="text-primary-500 hover:underline">$1</a>')
    .replace(/\n/g, '<br>');
}

loadComments();
</script>
{{ end }}
```

同時在功能欄位裡新增 <code> config/_default/menus.zh-tw.toml </code> 這個獨立的新頁面

```bash
[[main]]
  name = "留言板"
  pageRef = "guestbook"
  weight = 90
```

文章留言區跟獨立留言板的差異是計算方法，留言板上我們用固定的 <code> /guestbook/ </code> 作為唯一存取 Gist 的 Key，所以讀取速度比較快，但每篇文章底下因為我紀錄每個讀者的 id 跟 key 不同，所以倚賴 JavaScript 動態讀取留言數量

另外，因為我們在 <code> comments.html </code> 的設計是把所有的頁面都預設開啟留言區，如果讀者有任何頁面不想要顯示留言區的話就利用 <code> showComments </code> 這個參數來關閉

```bash
---
title: "My Post"
showComments: false
---
```

## 留言搜尋
由於所有的留言內容都是存放在 GitHub Gist JSON 檔案內，而 Hugo 內建搜尋功能是針對內部靜態編譯時產生的 index.json 檔案做搜尋，所以無法透過內建的方法搜尋到留言內容

我們想到的方法就是另外建置一個搜尋系統，蒐錄 Gist 所有的快取內容到瀏覽器內，再進行單一搜索，而在 Gist 檔案裡除了留言板是統一固定的 postKey，其餘文章內的留言則是不同的 id 跟 postKey

```bash
{
  "/posts/post-one/": [...comments],
  "/posts/post-two/": [...comments],
  "/guestbook/":      [...comments],
  ...
}
```

所以我們想利用一個頁面搜集所有的 postKey，建立 <code> content/comment-search/index.zh-tw.md </code> 頁面

```bash
---
title: "搜尋留言"
showComments: false
---
```

然後為這個頁面特別設計 <code> layouts/comment-search/single.html </code>

```bash
{{ define "main" }}
<div class="max-w-2xl mx-auto px-4 py-8">

  <p class="text-neutral-500 dark:text-neutral-400 mb-6">搜尋所有文章的留言</p>

  <!-- Search Input -->
  <div class="mb-6">
    <input id="c-search" type="text" placeholder="輸入關鍵字搜尋留言..."
      oninput="filterComments(this.value)"
      class="w-full px-4 py-2 rounded-lg border border-neutral-300 dark:border-neutral-600 bg-white dark:bg-neutral-700 text-neutral-800 dark:text-neutral-200 focus:outline-none focus:ring-2 focus:ring-primary-500" />
  </div>

  <!-- Result Count -->
  <p class="text-sm text-neutral-500 dark:text-neutral-400 mb-4">
    找到 <span id="c-count">0</span> 則留言
  </p>

  <!-- Results -->
  <div id="comment-list">
    <p class="text-neutral-400 text-sm">載入中...</p>
  </div>

</div>

<script>
const GIST_USERNAME = '{{ site.Params.comments.gistUsername }}';
const GIST_ID       = '{{ site.Params.comments.gistId }}';
const OWNER_NAME    = '{{ site.Params.comments.ownerName }}';

let allComments = []; // flat array of all comments across all posts

async function loadAllComments() {
  const el = document.getElementById('comment-list');
  try {
    const res = await fetch(`https://gist.githubusercontent.com/${GIST_USERNAME}/${GIST_ID}/raw/comments.json?t=${Date.now()}`);
    const all = await res.json();

    // Flatten all posts' comments into one array, attach postKey to each
    allComments = Object.entries(all).flatMap(([postKey, comments]) =>
      comments.map(c => ({ ...c, postKey }))
    ).sort((a, b) => new Date(a.date) - new Date(b.date));

    document.getElementById('c-count').textContent = allComments.length;
    renderComments(allComments);
  } catch(err) {
    el.innerHTML = '<p class="text-red-400 text-sm">載入留言失敗</p>';
  }
}

function filterComments(query) {
  if (!query.trim()) {
    document.getElementById('c-count').textContent = allComments.length;
    renderComments(allComments);
    return;
  }
  const q = query.toLowerCase();
  const filtered = allComments.filter(c =>
    c.name.toLowerCase().includes(q) ||
    c.content.toLowerCase().includes(q) ||
    c.postKey.toLowerCase().includes(q)
  );
  document.getElementById('c-count').textContent = filtered.length;
  renderComments(filtered);
}

function renderComments(list) {
  const el = document.getElementById('comment-list');
  if (list.length === 0) {
    el.innerHTML = '<p class="text-neutral-400 dark:text-neutral-500 text-sm">找不到符合的留言</p>';
    return;
  }
  el.innerHTML = list.map(c => `
    <div class="bg-neutral-100 dark:bg-neutral-900 border border-neutral-300 dark:border-neutral-700 rounded-xl p-5 mb-4">
      <div class="flex items-center justify-between mb-3">
        <div class="flex items-center gap-2">
          ${c.website
            ? `<a href="${escHtml(normalizeUrl(c.website))}" target="_blank" rel="noopener noreferrer"
                  class="font-medium text-neutral-800 dark:text-neutral-100 hover:text-primary-500 dark:hover:text-primary-400 hover:underline transition-colors duration-200">${escHtml(c.name)}</a>`
            : `<span class="font-medium text-neutral-800 dark:text-neutral-100">${escHtml(c.name)}</span>`
          }
        </div>
        <span class="text-xs text-neutral-500 dark:text-neutral-400">${new Date(c.date).toLocaleDateString('zh-TW', { year:'numeric', month:'2-digit', day:'2-digit', hour:'2-digit', minute:'2-digit' })}</span>
      </div>

      <!-- Post link -->
      <a href="${escHtml(c.postKey)}"
        class="inline-block text-xs text-primary-500 dark:text-primary-400 hover:underline mb-2">
        📄 ${escHtml(c.postKey)}
      </a>

      <p class="text-neutral-700 dark:text-neutral-200 text-base leading-relaxed">${formatContent(c.content)}</p>
      ${c.reply ? `
      <div class="mt-4 bg-neutral-200 dark:bg-neutral-800 rounded-lg p-4">
        <div class="flex items-center gap-2 mb-2">
          <span class="font-medium text-neutral-800 dark:text-neutral-100">${escHtml(OWNER_NAME)}</span>
          <span class="text-xs bg-primary-600 dark:bg-primary-700 text-white px-2 py-0.5 rounded">站長</span>
        </div>
        <p class="text-neutral-700 dark:text-neutral-200 text-base leading-relaxed">${formatContent(c.reply)}</p>
      </div>` : ''}
    </div>
  `).join('');
}

function escHtml(str) {
  return String(str).replace(/&/g,'&amp;').replace(/</g,'&lt;').replace(/>/g,'&gt;').replace(/"/g,'&quot;');
}

function normalizeUrl(url) {
  if (!url) return '';
  if (!/^https?:\/\//i.test(url)) {
    return 'https://' + url;
  }
  return url;
}

function formatContent(str) {
  const mdLink   = new RegExp('\\[([^\\]]+)\\]\\((https?:\\/\\/[^\\)]+)\\)', 'g');
  const bareUrl  = new RegExp('(?<!href=")(https?:\\/\\/[^\\s<]+)', 'g');

  return escHtml(str)
    .replace(mdLink,  '<a href="$2" target="_blank" rel="noopener noreferrer" class="text-primary-500 hover:underline">$1</a>')
    .replace(bareUrl, '<a href="$1" target="_blank" rel="noopener noreferrer" class="text-primary-500 hover:underline">$1</a>')
    .replace(/\n/g, '<br>');
}

loadAllComments();
</script>
{{ end }}
```

最後我們在 Hugo 內建的搜尋工具底下新增這個頁面，複製模板內的搜尋功能到我們自己的檔案底下 <code> layouts/partials/search.html </code>

```bash
cp themes/blowfish/layouts/partials/search.html layouts/partials/search.html
```

新增一項指引到留言搜尋頁面的元件

```bash
<section class="flex-auto px-2 overflow-auto">
  <ul id="search-results">
    <!-- 原本預設的內容保持不動 -->
  </ul>
  
  <!-- 新增底下的內容 -->
  <div id="comment-search-hint" class="px-3 py-3 border-t border-neutral-200 dark:border-neutral-700">
    <p class="text-xs text-neutral-400 dark:text-neutral-500">
      想搜尋留言？請前往
      <a href="/comment-search" class="text-primary-500 hover:underline">留言搜尋頁面</a>
    </p>
  </div>
</section>
```

![CommentSearch](https://media.secologies.com/CommentSearch.webp)

並且在檔案內搜尋欄位 html 之後加上 JavaScript 動態存取

```bash
<script>
  const searchQuery = document.getElementById('search-query');
  const hint = document.getElementById('comment-search-hint');

  searchQuery.addEventListener('input', () => {
    if (searchQuery.value.trim() === '') {
      hint.classList.remove('hidden');
    } else {
      hint.classList.add('hidden');
    }
  });
</script>
```

整個 <code> search.html </code> 檔案會長得像

```bash
<div
  id="search-wrapper"
  class="invisible fixed inset-0 flex h-screen w-screen cursor-default flex-col bg-neutral-500/50 p-4 backdrop-blur-sm dark:bg-neutral-900/50 sm:p-6 md:p-[10vh] lg:p-[12vh] z-500"
  data-url="{{ "" | absLangURL }}">
  <div
    id="search-modal"
    class="flex flex-col w-full max-w-3xl min-h-0 mx-auto border rounded-md shadow-lg top-20 border-neutral-200 bg-neutral dark:border-neutral-700 dark:bg-neutral-800">
    <header class="relative z-10 flex items-center justify-between flex-none px-2">
      <form class="flex items-center flex-auto min-w-0">
        <div class="flex items-center justify-center w-8 h-8 text-neutral-400">
          {{ partial "icon.html" "search" }}
        </div>
        <input
          type="search"
          id="search-query"
          class="flex flex-auto h-12 mx-1 bg-transparent appearance-none focus:outline-dotted focus:outline-2 focus:outline-transparent"
          placeholder="{{ i18n "search.input_placeholder" }}"
          tabindex="0">
      </form>
      <button
        id="close-search-button"
        class="flex items-center justify-center w-8 h-8 text-neutral-700 hover:text-primary-600 dark:text-neutral dark:hover:text-primary-400"
        title="{{ i18n "search.close_button_title" }}">
        {{ partial "icon.html" "xmark" }}
      </button>
    </header>
    <section class="flex-auto px-2 overflow-auto">
      <ul id="search-results">
        <!-- <li class="mb-2">
          <a class="flex items-center px-3 py-2 rounded-md appearance-none bg-neutral-100 dark:bg-neutral-700 focus:bg-primary-100 hover:bg-primary-100 dark:hover:bg-primary-900 dark:focus:bg-primary-900 focus:outline-dotted focus:outline-transparent focus:outline-2" href="${value.item.permalink}" tabindex="0">
            <div class="grow">
              <div class="-mb-1 text-lg font-bold">${value.item.title}</div>
              <div class="text-sm text-neutral-500 dark:text-neutral-400">${value.item.section}<span class="px-2 text-primary-500">&middot;</span>${value.item.date}</span></div>
              <div class="text-sm italic">${value.item.summary}</div>
            </div>
            <div class="ml-2 ltr:block rtl:hidden text-neutral-500">&rarr;</div>
            <div class="mr-2 ltr:hidden rtl:block text-neutral-500">&larr;</div>
          </a>
        </li> -->
      </ul>
      <div id="comment-search-hint" class="px-3 py-3 border-t border-neutral-200 dark:border-neutral-700">
      <p class="text-xs text-neutral-400 dark:text-neutral-500">
        想搜尋留言？請前往
        <a href="/comment-search" class="text-primary-500 hover:underline">留言搜尋頁面</a>
      </p>
      </div>
    </section>
  </div>
</div>

<script>
  const searchQuery = document.getElementById('search-query');
  const hint = document.getElementById('comment-search-hint');

  searchQuery.addEventListener('input', () => {
    if (searchQuery.value.trim() === '') {
      hint.classList.remove('hidden');
    } else {
      hint.classList.add('hidden');
    }
  });
</script>
```

![CommentSearchResult](https://media.secologies.com/CommentSearchResult.webp)

## 測試
到這裡我們就將所有留言區、留言板以及留言搜尋功能所需要的內容都設定完成了，可以來測試看看，請讀者在自己的留言區塊裡輸入一些內容，此時 Google 表單裡就會出現留言，而我們要做的就是把 <code> pending </code> 的狀態改成 <code> approved </code>，並且讀者可以選擇要不要回覆，變更完畢後等待 5 分鐘後就會自動觸發同步了，又或者可以手動測試

在 Google Apps Script 上方有個執行欄位，選擇 <code> syncToGist </code> 並且按下 <code> Run </code>，就可以手動執行同步功能

![ManualSync](https://media.secologies.com/ManualSync.webp)

最後成功的話就可以回到留言區塊去看是否成功

![CommentsResult](https://media.secologies.com/CommentsResult.webp)

{{< alert >}}
請注意 Cloudflare Turnstile 在 Hugo 測試模式下無法正常驗證，所以確認完內容無誤可以先部署到正式的環境中再測試
{{< /alert >}}

對了，如果對於要用 Google Sheets 當作儲存空間有點感冒，可以參考碩人哥的[留言板系統全新改版](https://shuojen.com/blog/2026/05/09/guestbook_v2)，改用 Cloudflare D1 搭配 Worker 啟動的更快

[^1]: JN, "留言板上線啦！（以及我怎麼挑選我的留言板系統）," 2025-07-30, 資工小廢物 - JN, [https://blog.giveanornot.com/comment-system-launched/](https://blog.giveanornot.com/comment-system-launched/)
[^2]: Wiwi Kuan, "留言區," 2024年10月26日, Wiwi.Blog｜官大為的部落格, [https://wiwi.blog/blog/social-media-feedback/](https://wiwi.blog/blog/social-media-feedback/).
[^3]: Ivon Huang, "為什麼我要用Disqus取代Giscus當作Hugo網站的留言板," 2023年10月20日, Ivon的部落格, [https://ivonblog.com/posts/replace-giscus-with-disqus/](https://ivonblog.com/posts/replace-giscus-with-disqus/).
[^4]: ikuka, "留言區大戰," 2026.04.21, ikukaroom, [https://blog.ikukaroom.com/comment-yes-or-no/](https://blog.ikukaroom.com/comment-yes-or-no/).
[^5]: YangBear, "留言與寄信," 14 Apr, 2026, YangBear'Blog ᓚᘏᗢ, [https://yangbear.bearblog.dev/2455/](https://yangbear.bearblog.dev/2455/).
[^6]: Alex Hsu, "部落格到底要不要開放評論區？," 2026年4月13日, Alex Hsu, [https://alexhsu.com/turn-on-comments](https://alexhsu.com/turn-on-comments)
[^7]: 皮皮, "DIY 網站留言簿," 2025 年 12 月 20 日, 廢文小天地, [https://trashposts.com/blog/build-my-own-guestbook/](https://trashposts.com/blog/build-my-own-guestbook/).
[^8]: LQ7, "做一個留言板," January 3, 2026, LQ7的創作與想像, [https://lq7.tw/tech/make-a-guestbook/](https://lq7.tw/tech/make-a-guestbook/).

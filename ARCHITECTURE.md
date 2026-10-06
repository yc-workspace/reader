# ARCHITECTURE

無伺服器新聞快照：GitHub Actions 定時抓 RSS／網頁／YouTube 字幕，摘要成繁體中文，
寫進 `data/archive.json` 並 commit。沒有資料庫，這個檔案就是整個系統的狀態。

## 1. 模組

| 檔案 | 職責 |
|---|---|
| `common.py` | archive 讀寫、摘要空值常數、id／語言碼驗證、host 比對、**唯一的對外 GET**（`common.get`）與解碼（`decode_body`） |
| `update_news.py` | OPML（檔案或 `FOLLOW_OPML_B64*` secret）／RSS、Telegram／即刻、`data/inbox.json` → 正規化 url、id、去重、留存 |
| `summarize_feed.py` | 路由 pending 條目、Pass 3 回填、`--backfill-only`、`--mine-boilerplate` |
| `extract.py` | 取文退路、HTML 萃取、擋頁偵測 |
| `textproc.py` | VTT 轉文字、樣板移除、抽取式摘要 |
| `thumbs.py` | 縮圖過濾與抽取 |
| `lang.py` | OpenCC、語言判斷、亂碼修復與偵測、DeepL／gtx 翻譯 |
| `subtitle_priority.py` | 字幕軌優先序（下載端與讀取端共用） |
| `download_sub.py` | yt-dlp 字幕、faster-whisper ASR 後備 |
| `selftest.py` | 離線契約檢查；兩個 workflow 都在碰資料前先跑 |
| `summary_boilerplate.json` | 樣板規則（資料，不是程式） |

依賴：`requirements.txt`，兩個 workflow 共用，全部釘死版本。

排程：`updatenews.yml` 每 8 小時、`downloadsub.yml` 每日。兩者都寫 `archive.json`，
共用 concurrency 群組 `archive-writer`（排隊、不取消）。

## 2. 資料

**格式**：頂層鍵各一行、每個條目一行（`common.dumps`）。所有寫入者都經過 `common.save_doc`。

| 欄位 | 寫入者 | 備註 |
|---|---|---|
| `id` | `update_news` | sha1(category, source, title, url)，40 位小寫 hex；也是字幕檔名，公式不可改 |
| `category` `source` `title` `url` | `update_news` | url 經 `canonical_url()`，只收 http(s) |
| `published_at` `last_seen_at` | `update_news` | feed 給的日期一律覆寫；`last_seen_at` 是 feed 最後一次列出該條目的時間 |
| `feed_content` | `update_news` | 暫存；摘要成功即刪，`summarize_feed`（含 `--offline-only`）收尾全刪 |
| `summary` `thumbnail` | `summarize_feed` | |
| `length` | `download_sub` | 影片秒數 |

**留存**：`last_seen_at` 超過 `ARCHIVE_DAYS` 天（預設 60，在 `updatenews.yml` 的 `env` 設定）的條目，
每次 `update_news` 執行時從 `archive.json` 刪除；沒有 `last_seen_at` 的舊條目改看 `published_at`。
對應的字幕檔由下一次 `download_sub` 清掉。
| `status` | 前端 | 本專案不產生，必須保留 |

**`summary` 的三種「沒有內容」不可互換：**

| 值 | 意義 | 後續 |
|---|---|---|
| 無此鍵 | 未處理 | 每次都重試 |
| `" "`（`BLANK_SUMMARY`） | 刻意留白：豆瓣標記、無語音、影片超過 ASR 上限、內容只剩樣板 | 不再重試 |
| `GONE_SUMMARY` | 404/410 且無存檔 | 不再重試 |

只有「條目本身的性質」能寫後兩種。被擋、逾時、預算不足、環境缺套件都保持 pending。

### `data/status.json`

`download_sub` 寫、閱讀器讀的健康旗標：`{"cookies": {"state": "ok" | "expired", "checked_at"}}`。
YouTube cookies 被拒絕（`[EXPIRED]`）時寫 `expired`；有影片實際被探測且 cookies 沒被拒絕時寫 `ok`；
這一輪沒有任何影片可探測就不動它（沒有證據，不能說 cookies 正常）。只有狀態改變、或仍是 `expired` 時才重寫，
健康的日子不會因此每天多一個 commit。閱讀器拿 `checked_at` 與自己上次送出 cookies 的時間比，
判斷「新 cookies 還在等下一次字幕下載確認」或「仍然失效」。這個檔案只是提示，不是狀態來源，壞了或不見就重來。

### `data/inbox.json`

閱讀器（`feed-github.js`）以 GitHub API 寫入：`{"urls": [{"url", "added"}]}`。`update_news` **只讀不寫**，
所以閱讀器在 workflow 執行途中送出也不會造成 rebase 衝突；修剪（保留 90 天、最多 1000 筆）由閱讀器負責。
每輪把 `added` 在留存期內的網址以 `category=inbox`、`source=URL` 送進 `ingest`：已知的沿用存下的標題
（`id` 含標題，換標題就是新條目），新的抓頁面 `og:title`／`<title>`，抓不到用 host + path。
只收 http(s)、不帶帳密、2048 字元內；摘要照一般路線由 `summarize_feed` 處理。

閱讀器同時負責 secret：連接資料夾 `context/` 裡的 `follow.opml`、`cookies.txt` 內容變了就以 sealed box
加密後更新 `FOLLOW_OPML_B64*`（多出來的分段刪除）與 `YT_COOKIES`。`opml_secret.html` 保留為手動備援。

## 3. `summarize_feed`

pending 依發布時間新到舊，先跑不連網的，再跑連網的；每筆變動立即存檔，單筆例外不中斷執行。

| 路由 | 條件 | 計入 `--max-items` |
|---|---|---|
| `blank` | 豆瓣「想讀／想看／想聽」 | ✗ |
| `youtube` | 有本地 `.vtt` 才處理，否則等 `download_sub` | ✗ |
| `feed` | `FEED_FIRST_HOSTS` 且有 `feed_content` | ✗ |
| `techmeme` | Sources/Report/Documents 開頭抓頁；其餘翻譯標題 + `↛` | 只有抓頁的 |
| `fetch` | 其餘 | ✓ |
| — | `news.google.com` | 跳過 |

`fetch` 退路：直連 → curl_cffi 指紋（`SLOW_HOSTS` 直接從這步開始）→ `feed_content`（≥200 字）→
reader proxy → Wayback → 短 `feed_content`（標 `↛`）。只拿到 meta 時停在第三步之後（`READER_ON_META` 可改）。

| 結局 | 寫入 | 本輪暫停該站 |
|---|---|---|
| `blocked` | ✗ pending | ✓（`douban.com` 整個網域） |
| `gone` | `GONE_SUMMARY` | ✗ |
| 讀到內容但只剩樣板 | `BLANK_SUMMARY` | ✗ |
| 內容是亂碼（`lang.garbled`） | ✗ pending，改走下一個退路 | ✗ |
| 暫時失敗 | ✗ pending | ✗ |

縮圖不依賴摘要成功。`douban.com`（含子網域）與 `finance.technews.tw` 完全不取縮圖。

時間預算 `TIME_BUDGET_SECONDS`：抓取在一半時停，回填可用到全部。

**Pass 3 回填**（全檔，規則改變後存量跟上）：縮圖重驗、樣板規則重套（`textproc.restrip`，
保留結尾標記，什麼都不剩就留白）、簡轉繁、翻譯。翻譯每 25 筆一批、每批存檔，連續 4 次失敗即停。

## 4. 摘要（`textproc.build`）

1. 去時間戳 → `strip_boilerplate`（`remove_block` → `remove_inline` → `cut_to_end`，作用在**原文字形**）。
2. `drop_unit` 段落／句子規則（轉繁後比對，依來源 `scope`）。
3. 中文直接抽取；其他語言多抽 20% 再翻譯，失敗存原文，由回填補救。
4. TF-IDF × 實體／數字加權 × 開頭加權，空洞問句降權；MMR 去重；依原文順序輸出整句。
5. 預算 = 長度 × `SUMMARY_RATIO`（預設 1），上限 `SUMMARY_MAX`；結尾標記先扣預算。

新規則用 `summarize_feed.py --mine-boilerplate N` 從語料挖候選，並確認不誤傷正文。

## 5. 語言與翻譯（`lang.py`）

**解碼**（`common.decode_body`）：BOM → 合法 UTF-8 一律勝出 → 宣告的編碼（header、`<meta>`、`<?xml?>`）→
對整個本文猜測。`latin-1`／`ascii`／`cp1252` 標籤常是伺服器預設值：猜到多位元組編碼（Big5、GBK…）時以猜測為準，
否則用標籤。舊版用大小寫敏感的 key 讀 header（永遠讀不到），又只拿前 200 KB 猜，把中文頁判成 ascii／cp1256，
亂碼再被「翻譯」成胡言亂語存進摘要。

**亂碼**：`fix_mojibake` 逐段修復被當成 cp1252／cp1256 讀的 UTF-8（cp1256 只接受修成 CJK 的段，以免動到真阿拉伯文）；
`garbled` 判斷殘留亂碼、U+FFFD、或同一片語重複十次以上（機翻垃圾的特徵）。亂碼不進摘要、不送翻譯、譯文是亂碼就拒收；
回填時把既存的亂碼摘要刪回 pending，以修正後的解碼重抓。

- 日文：假名佔「漢字＋假名」≥0.35（實測日文 ≥0.515、中文 ≤0.171）。
- `needs_translation`：漢字佔字母 >2% 即視為中文；否則 ≥8 個拉丁單字，或其他文字 ≥20 字。
  **接受譯文與選取待譯用同一個述詞**，否則永遠循環。
- DeepL 優先（一次 ≤50 段、`ZH-HANT`、413 對半、配額／授權錯誤後本行程停用）；gtx 退路，以編碼後位元組分段。
- 沒有 OpenCC 時原樣傳回，絕不輸出簡體。

## 6. 縮圖（`thumbs.py`）

只看 url（相對位址以 `urljoin` 解析，非 http(s) 一律拒絕）。順序：明列拒絕 → svg → 信任 host（Blogger）→
副檔名 → 資產目錄 → 圖庫／通訊社 → 版型／截圖 → 圖表詞彙（保留）→ 介面元件 → 裝飾檔名 → 照片描述詞 → 過長描述。
新增拒絕規則後，確認同站真實文章圖仍然通過。

## 7. 字幕（`download_sub.py`）

- 優先序：不需機翻 > 人工 > 語言。`live_chat` 不是字幕；`zh-Hant-TW` 是地區，`zh-Hant-en` 才是二次機翻。
- 檔名 `{id}.{原文語言}.{字幕語言}.vtt`；語言碼來自遠端，經 `safe_lang` 過濾才進檔名或 `--sub-langs`。
- ASR 預算以音訊秒數計；撞牆且剩不到 10 分鐘就收工。逾時殺整個行程群組並清半成品。
- 開始前刪除 id 已不在 archive 裡的 `.vtt`。

## 8. 安全

輸入（feed、網頁、YouTube 中繼資料）一律不可信，而輸出會進公開 commit。

| 威脅 | 處置 |
|---|---|
| SSRF（feed 連到 `169.254.169.254`、`localhost` 等，內容被摘要後公開） | `common.get`：只允許 http(s)、host 解析結果必須全是公網位址、手動跟隨轉址且每一跳重驗、不收帶帳密的 url、不吃環境變數代理 |
| DNS rebinding | 檢查在**連線當下**做：urllib3 的每條連線由 `_guarded_connect` 解析、驗證並直接連到通過的位址；curl 以 `CURLOPT_RESOLVE` 釘住 |
| 超大回應耗盡記憶體 | 串流讀取，上限 `MAX_BYTES`（8 MB） |
| 路徑穿越（遠端語言碼、竄改過的 id 進檔名） | `valid_id`（40 位 hex）、`safe_lang` |
| 參數注入（url 被 yt-dlp 當成選項） | url 一律放在 `--` 之後 |
| 遠端欄位進檔名 | 字幕輸出樣板不用 `%(language)s`，改用經 `safe_lang` 的語言碼 |
| 執行期下載未釘版的程式 | 移除 `--remote-components ejs:github`，改用 pip 鎖定的 `yt-dlp-ejs` |
| 惡意影片標題中止整輪 | cookies 失效只看 yt-dlp 的 stderr，不看含遠端中繼資料的 stdout |
| 橋接頁屬性拼成 url | Telegram post、即刻 id 以白名單正則驗證 |
| `opml_secret.html` 洩漏訂閱清單／注入 | 不用 `innerHTML`；CSP `default-src 'none'`，頁面不能連網 |
| 存入 `javascript:` 等連結 | 條目 url 與縮圖只收 http(s) |
| 亂碼／機翻垃圾公開發布 | `decode_body`；`garbled` 擋在萃取、摘要、翻譯三處，回填清除存量 |
| 閱讀器加入的網址 | `fetch_inbox` 只收 http(s)、無帳密、限長；抓標題同樣走 `common.get` |
| ReDoS | feed HTML 改用 BeautifulSoup 轉文字，不用回溯式正則 |
| workflow 指令注入／secret 外洩 | secret 只經 `env` 傳入；cookies 以 `umask 077` 寫到 `/dev/shm`，用完即刪；OPML 只在記憶體解開；cookie 值遮罩 |
| token 被任意步驟取用 | `persist-credentials: false`；`permissions: {}`，job 只給 `contents: write`；token 只出現在 push 那一步 |
| 供應鏈 | actions 釘 commit SHA；Python 套件釘版本；服務映像釘版本 |
| 並行寫入互相覆蓋 | 共用 concurrency 群組；rebase 衝突直接失敗，不再 `-X ours` 靜默丟資料 |

殘餘風險：reader proxy／Wayback 由第三方代抓（它們連到哪裡不在我們控制內）；yt-dlp 自己的連線不經 `common.get`（只對 YouTube 網址呼叫）；
pip 未用雜湊驗證、間接依賴未釘版；服務映像只釘 tag 未釘 digest；前端須自行跳脫 `title`／`summary`。

## 9. 執行

`downloadsub.yml` 下載完字幕後立刻跑 `summarize_feed.py --offline-only`（只處理 feed 副本與本機字幕，不抓網頁、
不回填，`feed_content` 照常收尾刪除），影片在同一輪就有摘要，不必等下一次 Update News。兩個 workflow 頻率不變，
同屬 `archive-writer` 並行群組，不會同時寫 `archive.json`。


| 工作 | 指令 |
|---|---|
| 蒐集 | `python scripts/update_news.py --output-dir data [--archive-days 60] [--rss-opml follow.opml]` |
| 摘要 | `ITEMS_FILE=data/archive.json MAX_ITEMS=200 python scripts/summarize_feed.py` |
| 只跑回填 | `python scripts/summarize_feed.py --backfill-only [--dry-run] [--limit N] [--no-translate]` |
| 字幕 | `python scripts/download_sub.py --archive data/archive.json --output-dir data/subtitles --cookies-path …` |
| 檢查 | `python scripts/selftest.py` |

環境變數：`ARCHIVE_DAYS` `MAX_ITEMS` `TRANSLATE` `SUMMARY_RATIO` `SUMMARY_MAX` `TIME_BUDGET_SECONDS` `USE_READER_PROXY`
`USE_WAYBACK` `READER_ON_META` `DEEPL_API_KEY` `SUBTITLES_DIR` `BOILERPLATE_FILE`。
Secrets：`FOLLOW_OPML_B64`（必要時加 `_2`…`_5`）、`DEEPL_API_KEY`、`YT_COOKIES`（三者都可由閱讀器從 `context/` 更新）。

**OPML 只放在 secret**。單一 secret 上限 48 KB，所以用 `opml_secret.html` 打包：
在瀏覽器開啟、按按鈕選 OPML 所在資料夾，會在同一資料夾寫出 `FOLLOW_OPML_B64.txt`（必要時 `_2.txt`…），
每個 txt 是同名 secret 的值。打包內容：只留 `xmlUrl` `htmlUrl` `title` `category`，gzip，base64，每段 ≤47,000 字元。
寫入資料夾需要 Chrome／Edge；其他瀏覽器改選檔案，txt 會下載到「下載」資料夾。
`update_news.py` 沒給 `--rss-opml` 時從環境變數合併解開（gzip、xz、純 base64 都接受），只在記憶體中，不落地；
CI 加 `--require-opml`，沒設定 secret 時工作直接失敗。

## 10. 已知盲區

| 盲區 | 說明 |
|---|---|
| 摘要品質 | 選錯句子不會報錯，只能人工抽樣 |
| 站台改版 | 表現為 `↛` 比例上升或摘要變導覽列文字 |
| 第三方服務 | reader proxy、Wayback、gtx 皆無 SLA |
| 大量匯入 | 單輪新增遠超 `MAX_ITEMS` 時，未處理條目的 `feed_content` 會在收尾被清掉 |

# ARCHITECTURE

無伺服器新聞快照：GitHub Actions 定時抓 RSS／網頁／YouTube 字幕／podcast 語音轉文字，摘要成繁體中文，
寫進 `data/archive.json` 並 commit。沒有資料庫，這個檔案就是整個系統的狀態。

## 1. 模組

| 檔案 | 職責 |
|---|---|
| `store.py` | 摘要空值常數、id／語言碼驗證、**`Config`（唯一讀環境變數的地方，`store.configure()` 由入口呼叫，各模組在呼叫時讀 `store.CFG`，不在 import 時讀）**、`rules.json` 讀取（`rules()`、`site()`）、host 比對與網址約定（`is_youtube`、`is_episode`）、日期解析、`__NEXT_DATA__` 讀取、archive 讀寫（`save_doc` 欄位白名單）、字幕檔約定（軌道優先序、`pick_subtitle`、`prune_subtitles`：每個 id 只留一個 `.vtt`） |
| `net.py` | **唯一的對外連線**（`get`、`request`、`download`，見 §8）與解碼（`decode_body`） |
| `textproc.py` | 唯一一套 HTML 工具：串流讀取器（同一個 `HTMLParser` 核心 `_Stream`，三種文字輸出：`html_to_text` 全部區塊、`region_text` 限定 `main`／`article`／`body` 並略過 nav／header／footer、`paragraph_text` 只取 `<p>`／`<li>`，以第一個含段落的 `article`、否則 `main` 限定範圍，略過區（aside 等）裡的 article 不算；同一核心的 `elements` 回傳 `Doc`（元素清單、`first`、`text`），給需要原始 HTML 片段的地方——澎湃正文、`strip_code`——元素起訖位置；`strip_code` 只在原始碼上切片、不重新序列化，含 `pre`／`samp`／`kbd` 的 `<code>` 整段移除）、`meta_tags`、`meta_first`、`page_title`；頁面抽取（`from_html`：trafilatura → 段落退路 → 介面文字判定 `chrome_only`，以及 `strip_code`、`meta_description`、`unescape`、`MIN_BODY`）；`sanitize_markup`、`JUNK_RE`；OpenCC、語言判斷、亂碼修復與偵測（`garbled` 擋輸入；`diagnose` 給 QC，範圍較廣）、DeepL／gtx 翻譯；VTT 轉文字、樣板移除（`rules.json` 的 `boilerplate`）、抽取式摘要（`build`） |
| `collect.py` | 入口一（`updatenews.yml`）。OPML（檔案或 `FOLLOW_OPML_B64*` secret）／RSS、Telegram／即刻、YouTube 頻道、`data/inbox.json`、抓取來源（BestBlogs、澎湃，受 `SCRAPED_SOURCES` 控制）→ 正規化 url、id、去重、留存；接著在同一行程跑 `summarize.run`（`--no-summarize` 時不跑） |
| `summarize.py` | 縮圖過濾（`thumb_*`）、取文退路（`fetch_article`，抽取本身在 `textproc.from_html`）、摘要 QC（`qc_*`，只輸出到 log）、路由 pending 條目、跨 host 並行抓取、回填；`run()` 供兩個入口呼叫；命令列是維護工具（`backfill`、`mine`） |
| `media.py` | 入口二（`downloadsub.yml`）。yt-dlp 字幕（子行程只拿白名單環境變數）、faster-whisper ASR 後備；podcast 單集頁（小宇宙、Firstory）→ 音檔網址與時長（`resolve_episode`）→ ASR；`data/status.json`；最後在同一行程摘要新逐字稿 |
| `selftest.py` | 離線契約檢查；兩個 workflow 都在碰資料前先跑。每個檢查前重讀設定，單一檢查崩潰會記為失敗而不中斷其他檢查 |
| `rules.json` | 唯一的資料檔，三個獨立區段：`boilerplate`（摘要時刪除的樣板）、`qc_chrome`（介面文字樣式：抽取時判定「沒有正文」、QC 判定「只剩介面文字」，兩處共用 `textproc.chrome_re`）、`sites`（站台表，在使用處以 `store.site()` 讀取，不在 import 時讀）。`boilerplate` 與 `qc_chrome` 不可合併：前者會在 QC 看到之前就把介面文字刪掉 |

依賴：`requirements.txt`，兩個 workflow 共用，全部釘死版本（`updatenews.yml` 安裝時略過 faster-whisper、yt-dlp、bgutil 三行，那個 job 用不到）。網路只用 curl_cffi（本專案不直接使用 `requests`，它是 trafilatura 等套件的依賴）。

內部 import（無循環；`store`、`net` 不依賴本專案任何模組）：

```
collect, media -> store, net, textproc, summarize
summarize      -> store, net, textproc
textproc       -> store, net
store, net     -> (無)
```

排程：`updatenews.yml` 每 8 小時、`downloadsub.yml` 每日。兩者都寫 `archive.json`，
共用 concurrency 群組 `archive-writer`（排隊、不取消）。

## 2. 資料

**格式**：頂層鍵各一行、每個條目一行（`store.dumps`）。所有寫入者都經過 `store.save_doc`。頂層除了 `items` 還有 `generated_at`、`total_items`，以及 `rules_hash`（見 §3 回填）。

| 欄位 | 寫入者 | 備註 |
|---|---|---|
| `id` | `collect` | sha1(category, source, title, url)，40 位小寫 hex；也是字幕檔名，公式不可改 |
| `category` `source` `title` `url` | `collect` | url 經 `canonical_url()`，只收 http(s)；同站多個網域統一為一個（`HOST_ALIASES`：`open.firstory.me` → `open.firstory.fm`）。規則改變後，`load_archive` 會在下次執行時改寫已存的 url、重算 id，並合併因此重複的條目（重算後撞上既有 id 的也合併，不覆蓋） |
| `published_at` `last_seen_at` | `collect` | feed 給的日期一律覆寫；`last_seen_at` 是 feed 最後一次列出該條目的時間 |
| `summary` `thumbnail` | `summarize` | |
| `length` | `media` | 影片／單集秒數，給前端顯示 |
| `status` | 前端 | 本專案不產生，必須保留 |

**條目只有上表這 11 個欄位**（`store.ITEM_FIELDS`）：`save_doc` 只寫這些欄位（保留原鍵序），其他鍵一律丟棄。也不另設狀態檔記錄條目的處理狀態（`data/status.json` 只記 cookies 健康）。
`feed_content`／`feed_images` 是 `collect` 交給 `summarize` 的 feed 副本，**只在同一行程的記憶體中**（`collect.py` 蒐集後直接摘要），沒有任何暫存檔。

**留存**：`last_seen_at` 超過 `ARCHIVE_DAYS` 天（預設 60，在 `updatenews.yml` 的 `env` 設定）的條目，
每次 `collect` 執行時從 `archive.json` 刪除；沒有 `last_seen_at` 的條目看 `published_at`。
對應的字幕檔由下一次 `media` 清掉。

**`summary` 的三種「沒有內容」不可互換：**

| 值 | 意義 | 後續 |
|---|---|---|
| 無此鍵 | 未處理 | 每次都重試 |
| `" "`（`BLANK_SUMMARY`） | 刻意留白：豆瓣標記的 feed 副本沒有備註、podcast 單集頁沒有可用音檔、無語音、影片超過 ASR 上限、內容只剩樣板 | 不重試 |
| `GONE_SUMMARY` | 404/410 且無存檔 | 不重試 |

只有「條目本身的性質」能寫後兩種。被擋、逾時、預算不足、環境缺套件都保持 pending。

### `data/status.json`

`media.py` 寫、閱讀器讀的健康旗標：`{"cookies": {"state": "ok" | "expired", "checked_at"}}`。
YouTube cookies 被拒（yt-dlp 的 stderr 含 `cookies`）時寫 `expired`；這一輪有 YouTube 影片實際被探測且沒被拒時寫 `ok`；
沒有探測任何 YouTube 影片（包括只處理 podcast 的一輪，podcast 不用 cookies）就不動它。
只有狀態改變、或仍是 `expired` 時才重寫，健康的日子不會每天多一個 commit。檔案壞了或不見就重來，它只是提示。

### OPML 來源的特殊網址

- **Telegram**：`xmlUrl`／`htmlUrl` 是 `t.me/NAME`、`t.me/s/NAME`、`telegram.me/NAME` 或貼文 `t.me/NAME/123` 時，改抓公開預覽頁
  預覽頁以 `collect._TelegramPage`（`HTMLParser` 串流，不建樹）一次讀完：每則貼文的全文（不含引用的回覆，保留換行，連同連結預覽的標題與描述）存成 `feed_content`，相片／影片縮圖／連結預覽圖（inline `background-image`）存成 `feed_images`；`t.me` 在 `feed_first_hosts`，所以摘要只用這份副本、不抓單則貼文頁（副本已含全文與圖片，避免每則貼文多一次請求），縮圖取自 `feed_images`（`telesco.pe` 為信任圖床）。標題取第一個 `.tgme_widget_message_text` 的文字，空白合併後截 96 字；id 由標題計算，所以這個算法固定，避免既有條目換 id 而重複出現。
  `https://t.me/s/NAME`。NAME 須符合 `[A-Za-z][A-Za-z0-9_]{3,31}`，邀請連結、`t.me/c/…`、保留路徑一律不收。rsshub 的 telegram／即刻路由同樣橋接。
- **YouTube**：`/channel/UC…` 與 `playlist?list=…` 直接換成 RSS 網址；`/@handle`、`/c/…`、`/user/…` 抓取時先讀頁面，
  從 RSS `<link>`、canonical `<link>` 或 `"externalId"` 取出格式正確的頻道 id 再抓 RSS，取不到記為該 feed 失敗。

### 抓取來源（無 RSS）：BestBlogs、澎湃

由 `updatenews.yml` 頂層的 `SCRAPED_SOURCES` 控制：`on`（或 `1`、`true`、`yes`）才執行，其他值或未設定都不執行，**預設 `off`**。selftest 一律強制關閉，不受 workflow 設定影響。

- **BestBlogs**（`www.bestblogs.dev/newsletter/issueN`）：archive 已有期數時，從最大期數往上探測，連續 3 期不存在就停；沒有任何期數時，先倍增再二分找出最新一期，再往回取留存期內的期數（最多 30 期）。內文去掉導覽列等介面元素後，轉成純文字作為 feed 副本。各語言版本的網址一律正規化成同一個。
- **澎湃**（`THEPAPER_CHANNELS`，目前是上海書評頻道）：從頻道頁的 `__NEXT_DATA__` 取得文章列表，失敗時改用頁面上的連結。只有新條目或仍為 pending 的條目才會抓文章頁，而且只取該篇自己的內文（依 `contId` 比對），日期沒有時區時視為北京時間。
- 兩個網站都在 `FEED_FIRST_HOSTS` 中，`summarize` 只使用 feed 副本，不會再抓頁面。
- 網址正規化規則（`canonical_url`）不受開關影響，所以切換開關不會改變 id。關閉之後，既有條目照常依留存期限淘汰。

### `data/inbox.json`

閱讀器（`feed-github.js`）以 GitHub API 寫入：`{"urls": [{"url", "added"}]}`。`collect` **只讀不寫**，
所以閱讀器在 workflow 執行途中送出也不會造成 rebase 衝突；修剪（保留 90 天、最多 1000 筆）由閱讀器負責。
每輪把 `added` 在留存期內的網址以 `category=inbox`、`source=URL` 送進 `ingest`：已知的沿用存下的標題
（`id` 含標題，換標題就是新條目），新的抓頁面 `og:title`／`<title>`，抓不到用 host + path。
只收 http(s)、不帶帳密、2048 字元內；摘要照一般路線由 `summarize` 處理。

閱讀器同時負責 secret：連接資料夾 `context/` 裡的 `follow.opml`、`cookies.txt` 內容變了就以 sealed box
加密後更新 `FOLLOW_OPML_B64*`（多出來的分段刪除）與 `YT_COOKIES`。`opml_secret.html` 保留為手動備援。

## 3. `summarize`

pending 依發布時間新到舊，先跑不連網的，再跑連網的；單筆例外不中斷執行。存檔節流：最多每 30 秒（`SAVE_EVERY`）寫一次，結束時（含例外）一定寫。

**並行抓取**：取 pending 中最新的 `MAX_ITEMS` 筆，依 host 分組（`douban.com` 整個網域算一組），每組一個任務依日期逐筆抓、兩筆之間隔 `SLEEP`（1.5 秒），同時最多 `HOST_WORKERS`（6）組。某組被拒（`blocked`）就結束該組，其餘條目本輪不抓、保持 pending。工作執行緒只抓頁、建摘要，log 先寫進自己的緩衝；寫入條目、QC `gate`、暫停站台、存檔都只在主執行緒做。時間預算到了各組就不開始新條目，進行中的照常收完。gtx 翻譯有全行程共用的限速（`GTX_SLEEP`），不因執行緒數倍增。

| 路由 | 條件 | 計入 `MAX_ITEMS` |
|---|---|---|
| `douban` | 豆瓣「想看／想讀／想聽／看過／讀過／聽過」且本行程持有 feed 副本：取 `feed_content` 的備註（去掉推薦、標籤行），副本沒有備註才留白、亂碼則保持 pending；從不抓頁。**沒有副本**（`media` 的離線摘要、後續輪次）時不處理、保持 pending，等下一次蒐集送來副本 | ✗ |
| `youtube` | 有本地 `.vtt` 才處理，否則等 `media`。逐字稿只剩樣板或無法解碼時寫 `BLANK_SUMMARY`（同一個檔案每輪只會得到同樣結果） | ✗ |
| `podcast` | 小宇宙／Firstory 單集頁且有本地 `.vtt`：摘要逐字稿。沒有 `.vtt` 時**不處理**（等 `media`），沒有時間上限，也從不改用 show notes | ✗ |
| `feed` | `FEED_FIRST_HOSTS` 且有 `feed_content` 或 `feed_images`（YouTube 先被 `youtube` 路由接走，且 `collect` 不為它存 feed 副本）：只用 feed，從不抓頁；縮圖取 `feed_images`；沒有文字就留白，亂碼則留 pending | ✗ |
| `fetch` | 其餘 | ✓ |
| — | `news.google.com` | 跳過 |

`fetch` 退路：直連（curl，不偽裝指紋）→ 瀏覽器指紋（`SLOW_HOSTS` 直接從這步開始）→ `feed_content`（≥200 字）→
reader proxy → Wayback → 短 `feed_content`（標 `↛`）。只拿到 meta 時停在第三步之後。reader proxy 與 Wayback 由 `FALLBACKS` 決定（預設 `reader,wayback`；加上 `reader-meta` 則只拿到 meta 的頁面也送 reader；空字串表示都不用）。每次嘗試都在 log 留下 `via=reader ok|miss`、`via=wayback ok|miss`，可直接統計效益。

| 結局 | 寫入 | 本輪暫停該站 |
|---|---|---|
| `blocked` | ✗ pending | ✓（`douban.com` 整個網域） |
| `gone` | `GONE_SUMMARY` | ✗ |
| 讀到內容但只剩樣板 | `BLANK_SUMMARY` | ✗ |
| 內容是亂碼（`textproc.garbled`） | ✗ pending，改走下一個退路 | ✗ |
| 正文只剩介面文字（`textproc.chrome_only`：去掉 `qc_chrome` 樣式後不到 `MIN_BODY`） | 視為沒有正文，改走下一個退路 | ✗（與 `blocked` 分開判定，不暫停該站） |
| 暫時失敗 | ✗ pending | ✗ |

`collect` 把條目 HTML 轉純文字存成 `feed_content`，同時把其中的圖片網址（`<img>` 在前，`media:content`／`media:thumbnail`／圖片 enclosure 在後，最多 12 個）存成 `feed_images`；兩者只在記憶體中交給同一行程的 `summarize`，從不寫進 `archive.json`，也沒有暫存檔。feed 列出某條目時一定留下 `feed_content` 鍵（即使是空字串），所以「有這個鍵」就代表本行程持有副本；豆瓣路由靠這點區分「沒有副本」（等待）與「副本沒有備註」（留白）。

縮圖不依賴摘要成功。`rules.json` `sites.thumb_skip_hosts` 列出的站（`douban.com`、`granitefirm.com` 等，含子網域）完全不取縮圖；回填時這些站已存的縮圖也會被移除。

時間預算 `TIME_BUDGET_SECONDS`：抓取在一半時停，回填可用到全部。

**摘要 QC**（回填第一步，`summarize.qc_apply`）：

| 類別 | 修正 |
|---|---|
| `garbled` | `fix_mojibake` 修得好就地修；否則刪回 pending 重抓（破碎機翻同此：原文已不在，重翻無用） |
| `loop` | `collapse_loops` 就地收斂；迴圈佔大半才重建（`build` 也先收斂，字幕不會卡在 pending） |
| `scrape` | 只剩介面文字 → 重抓；HTML 殘片去標籤後夠長就地修。技術文章（`CODE_NOTE`、HTML/CSS/標籤等語境）不判 |
| `mismatch` | 標題與摘要最吻合的一筆保留，其餘重抓；網址含 `undefined/null` 一律重抓；標題關鍵字重疊 ≥0.3 視為改標題轉載，不判 |
| `spam` | 推銷訊號分組（儲值、代辦行銷、聯絡方式…），需兩種不同類別，或一種加上博彩／成人主題；`Rp` 金額只作輔助、不單獨成立；有新聞語氣（「表示」「警方」「said」…）時需三種類別。寫 `BLANK_SUMMARY` |

不存任何狀態：`Run.done` 寫入前先過 `summarize.qc_gate`（以 `shared_index` 查找共用摘要，不逐筆掃全檔），新摘要若有同樣問題就修正或寫 `BLANK_SUMMARY`，
所以每筆最多重抓一次，不會循環。YouTube 直接留白（本機字幕只會重建出同樣內容）。
`restrip` 對技術文章（`textproc.looks_like_code`）不跑 `sanitize_markup`，保留 HTML 範例。
log 為英文，依類別列出有問題的標題、處置與修改提示，不印統計數字（Actions 另加 `::warning::`，標題中的 `%`、換行、`::` 先跳脫），不產生報表檔。

**回填**（全檔，規則改變後存量跟上；`backfill --qc-only` 只做上面的 QC）：QC、縮圖重驗、樣板規則重套（`textproc.restrip`，
保留結尾標記，什麼都不剩就留白）、簡轉繁、翻譯。翻譯每 25 筆一批、每批存檔，連續 4 次失敗即停。

**只在規則改變時跑全檔**：`store.rules_hash()` 是 `rules.json`、`textproc.py`、`summarize.py` 的雜湊。完整回填結束後寫入 archive 的 `rules_hash`；之後的執行若雜湊相同，只重試仍需翻譯的條目（新摘要寫入前已過 `qc_gate`，且用的就是現行規則，重跑全檔不會有結果）。改了任一檔案，下一輪自動全檔跑一次。命令列 `summarize.py backfill` 一律跑全檔。

## 4. 摘要（`textproc.build`）

1. 去時間戳 → `strip_boilerplate`（`remove_block` → `remove_inline` → `cut_to_end`，作用在**原文字形**）。每條規則各自編譯、各自套用（不串成一條 alternation：反向參照會指錯群組，行內旗標或重複群組名會讓整組編譯失敗）；`cut_to_end` 取「之前至少保留 `min_keep_after_cut` 字」的最早標記，開頭附近的標記（如標題引用）不會遮住後面真正的頁尾。
2. `drop_unit` 段落／句子規則（轉繁後比對，依來源 `scope`）。
3. 中文直接抽取；其他語言多抽 20% 再翻譯，失敗存原文，由回填補救。
4. TF-IDF × 實體／數字加權 × 開頭加權，空洞問句降權；MMR 去重；依原文順序輸出整句。
5. 預算 = 長度 × `SUMMARY_RATIO`（預設 1），上限 `SUMMARY_MAX`；結尾標記先扣預算。
6. **換行**：選出的句子若在原文中分屬不同行，摘要在兩句之間換行；同一行的句子接在一起。
   每行去掉多餘空白（中文標點前後不留空白），不留空行，只用單一 `\n`。字幕與 podcast 逐字稿（`subtitle`）不保留換行，
   因為那些行是字幕 cue 或合併單位，不是講者的段落。
   翻譯時每行各自送出（`textproc.translate_keep_lines`），結果的換行與原文一致；有任何一行翻不成，就整段合成一行重試一次（不保留換行）。
   回填翻譯與 QC 去 HTML 修復也保留換行。既存摘要沒有換行資訊，不會因此改變；閱讀器要以 `white-space: pre-line` 之類的方式顯示換行，前端不在這個 repo。

新規則用 `summarize.py mine N` 從語料挖候選，並確認不誤傷正文。

## 5. 語言與翻譯（`textproc.py`）

**解碼**（`net.decode_body`）：BOM → 合法 UTF-8 一律勝出 → 宣告的編碼（header、`<meta>`、`<?xml?>`）→
對整個本文猜測。`latin-1`／`ascii`／`cp1252` 標籤常是伺服器預設值：猜到多位元組編碼（Big5、GBK…）時以猜測為準，
否則用標籤。header 以不分大小寫的方式讀取（大小寫敏感的 key 會永遠讀不到 `Content-Type`）；
猜測以整個本文為準（只看開頭時，前面是大段 ASCII 標記的中文頁會被判成 ascii／cp1256），避免亂碼再被「翻譯」成胡言亂語存進摘要。

**亂碼**：`fix_mojibake` 逐段修復被當成 cp1252／cp1256 讀的 UTF-8（cp1256 只接受修成 CJK 的段，以免動到真阿拉伯文）；
`garbled` 判斷殘留亂碼、U+FFFD、或同一片語重複十次以上（機翻垃圾的特徵）。亂碼不進摘要、不送翻譯、譯文是亂碼就拒收；
回填時把既存的亂碼摘要刪回 pending，重新抓取並以現行解碼處理。

- 日文：假名佔「漢字＋假名」≥0.35（實測日文 ≥0.515、中文 ≤0.171）。
- `needs_translation`：漢字佔字母 >2% 即視為中文；否則 ≥8 個拉丁單字，或其他文字 ≥20 字。
  **接受譯文與選取待譯用同一個述詞**，否則永遠循環。
- DeepL 優先（一次 ≤50 段、`ZH-HANT`、413 對半，成功的一半保留結果（DeepL 已計費），只有失敗的段落改走 gtx；配額／授權錯誤後本行程停用：是否停用由 `_deepl` 依狀態碼（401／403／456）在產生錯誤時決定，存在 `TranslateError.kind`，不比對訊息文字；gtx 被跳過以 `Skipped` 型別辨識）；gtx 退路，以編碼後位元組分段。`stop_after` 只計 gtx 連續的傳輸失敗（拒絕、逾時、非 200）；gtx 有回應但譯文不合格不算拒絕。逐行翻譯因 gtx 拒絕而被跳過時直接回報跳過、不整段重送，避免對方拒絕時請求量加倍。
- 沒有 OpenCC 時原樣傳回，絕不輸出簡體。

## 6. 縮圖（`summarize.py`）

只看 url（相對位址以 `urljoin` 解析，非 http(s) 一律拒絕）。先 `clean()` 去掉雲端圖片處理參數（`x-oss-process`、`x-image-process`、`x-bce-process`、`imageMogr2`、`imageView2`，例如 `?x-oss-process=image/resize,w_1024`），其他參數與 `#` 片段保留，存下的是原圖網址；回填時已存的縮圖同樣清理。
再依序：明列拒絕（`DENY`，含 Barron's 標誌圖）→ svg → 信任 host（Blogger）→
副檔名 → 資產目錄 → 圖庫／通訊社 → 版型／截圖 → 圖表詞彙（保留）→ 介面元件 → 裝飾檔名 → 照片描述詞 → 過長描述。
新增拒絕規則後，確認同站真實文章圖仍然通過。

## 7. 字幕（`media.py`）

- 優先序：不需機翻 > 人工 > 語言。`live_chat` 不是字幕；`zh-Hant-TW` 是地區，`zh-Hant-en` 才是二次機翻。
- 每個 id 只留一個 `.vtt`：下載或 ASR 成功後，以 `store.prune_subtitles` 刪除同 id 排名較低的檔案；每輪開始時也清一次既有的多餘檔案。排序仍由 `pick_subtitle` 決定。
- ASR 輸出 VTT 時，與前一個 cue 文字相同的 cue 併入前一個（延長結束時間），避免 Whisper 幻覺迴圈把同一句寫成幾十個 cue；不相鄰的重複保留。
- 檔名 `{id}.{原文語言}.{字幕語言}.vtt`；語言碼來自遠端，經 `safe_lang` 過濾才進檔名或 `--sub-langs`。
- ASR 預算以音訊秒數計；撞牆且剩不到 10 分鐘就收工。逾時殺整個行程群組並清半成品。
- 開始前刪除 id 已不在 archive 裡的 `.vtt`。
- **每支影片只萃取一次**：`--dump-json` 探測的結果存成暫存 info JSON（系統暫存目錄，權限 0700，用完即刪，永遠不在 repo 內：它可能含 session cookies），字幕以 `--load-info-json` 從它下載，避免重複萃取（那會多一次 YouTube 請求，也提高被限流的機會），也不需要固定等待。若這一步不是因為「沒有這個語言的字幕」而失敗（例如 PO token 過期），改用網址重試一次。
- **cookies 失效**只認 yt-dlp 的特定訊息（`cookies are no longer valid`、`have been rotated`、`confirm you're not a bot`），且只看 stderr。年齡限制、會員影片的提示也會提到 `--cookies`，不算失效，不中止整輪。
- 下載與 ASR 結束後，在同一行程以 `summarize.run(offline_only=True)` 摘要新逐字稿，archive 只寫一次。
- **ASR 失敗的歸類**：只有條目本身的性質寫 `BLANK_SUMMARY`——沒有語音（`NoSpeech`）、影片超過 ASR 時長上限（3 小時，`TooLong` 或已知時長）。音檔下載失敗、逾時、缺 faster-whisper、預算不足一律保持 pending。
- **Podcast**（`www.xiaoyuzhoufm.com/episode/…`、`open.firstory.fm|me/story/…`；節目首頁、使用者頁不算）：`media.resolve_episode` 找音檔——小宇宙取頁面 `og:audio`（.m4a），時長取 `__NEXT_DATA__` 的 `duration`、退而取「N分钟」；Firstory 先找頁面裡的 mp3／m4a 網址，沒有就讀頁面連到的節目 RSS（`feed.firstory.me/rss/user/…`），取網址含該集 id 的條目的 enclosure 與 `itunes:duration`。音檔以 `net.download` 下載（與 `net.get` 同一套位址檢查，每一跳轉址與每條連線都重驗；上限 512 MB、整體時限同 ASR 下載逾時；不經 yt-dlp、不帶 cookies），再走同一套 Whisper 與同一份 ASR 預算；時長未知時以實際音訊長度計入預算，時長同時寫入 `length`。檔名 `{id}.zh.{偵測語言}.vtt`。單集頁讀得到但沒有可用音檔（頁面無音檔、頁面 404/410、音檔指向非公網位址）：寫 `BLANK_SUMMARY`，這是單集本身的性質。讀不到單集頁或節目 RSS（無回應、403、5xx 等，`EpisodeUnreachable`）、解析錯誤、下載或轉錄失敗、超過 ASR 時長上限：保持 pending，下一輪再試，次數不限、不做任何標記。沒有語音同樣寫 `BLANK_SUMMARY`。

## 8. 安全

輸入（feed、網頁、YouTube 中繼資料）一律不可信，而輸出會進公開 commit。

| 威脅 | 處置 |
|---|---|
| feedparser 把本文當路徑或網址 | `feedparser.parse()` 收到 str／bytes 會先當本機檔名開啟，str 若像網址（含 `file://`）還會用 urllib 自己抓、不經 `net`。所有 feed 本文都經 `store.parse_feed`（包成串流）|
| SSRF（feed 連到 `169.254.169.254`、`localhost` 等，內容被摘要後公開） | 所有對外連線（頁面、feed、podcast 音檔、DeepL／gtx、Wayback、reader proxy）都走 `net.request`；303 轉為 GET 只影響該次嘗試，重試從原請求重來：只允許 http(s)（`CURLOPT_PROTOCOLS_STR`）、不收帶帳密的 url、host 解析結果必須全是公網位址（含拒絕 NAT64／6to4／Teredo 這類內嵌 IPv4 的 IPv6 位址）、手動跟隨轉址且每一跳重驗，轉到不同 origin（scheme／host／port）時拿掉 `Authorization`、`Proxy-Authorization`、`Cookie`（DeepL 金鑰不跟著轉址外送），301／302 的 POST 依瀏覽器慣例改為 GET；非公網目標在建立任何連線前就拒絕，不會被重試 |
| DNS rebinding | 每一跳只解析一次，連線以 `CURLOPT_RESOLVE` 釘在剛驗過的位址；每個請求用全新的 curl handle（`use_thread_local_curl=False`），不會重用舊的解析或連線 |
| 經由代理繞過位址檢查 | `trust_env=False`，並明確設定 `CURLOPT_PROXY=""`、`CURLOPT_NOPROXY="*"`：libcurl 自己也會讀 `http_proxy` 等環境變數，只關 `trust_env` 不夠 |
| 慢速傳輸拖住整輪（每次只送一個 byte） | 每個請求有**總時限**（不是單次讀取逾時），由 libcurl `TIMEOUT_MS` 與接收 callback 一起把關；連同重試也不會超過 |
| 超大回應耗盡記憶體 | 接收時累計大小，超過上限（頁面 8 MB、podcast 512 MB）立即中止；`MAXFILESIZE` 也先擋下宣告過大的回應 |
| TLS 憑證 | curl 預設驗證憑證與主機名；selftest 以本機自簽憑證確認會被拒絕（一般與指紋兩種模式） |
| 路徑穿越（遠端語言碼、竄改過的 id 進檔名） | `valid_id`（40 位 hex）、`safe_lang` |
| 參數注入（url 被 yt-dlp 當成選項） | url 一律放在 `--` 之後 |
| podcast 頁面給的音檔網址指向內網 | `media.resolve_episode` 先以 `net.safe_url` 篩；下載走 `net.download`，轉址每一跳、每條連線都釘在驗過的公網位址（DNS rebinding 同樣擋下），大小與時間有上限 |
| 遠端欄位進檔名 | 字幕輸出樣板用經 `safe_lang` 的語言碼，不用 `%(language)s`（遠端字串會直接進檔名） |
| 執行期下載未釘版的程式 | 用 pip 鎖定的 `yt-dlp-ejs`，不用 `--remote-components ejs:github`（那會在執行期從 GitHub 下載未釘版的程式） |
| 子行程取得 secret | `DEEPL_API_KEY` 會出現在 `media` 的環境（摘要要用），但 `media.run_cmd` 只把白名單內的環境變數（`PATH HOME LANG HF_HOME DENO_* XDG_*` 等）交給 yt-dlp，其中名稱含 `KEY/TOKEN/SECRET/PASSWORD/OPML/COOKIE/CREDENTIAL/AUTH` 的仍剔除 |
| feed 副本落地／被 commit | 沒有暫存檔：feed 副本只在記憶體；`save_doc` 以欄位白名單寫檔 |
| 惡意影片標題中止整輪 | cookies 失效只看 yt-dlp 的 stderr 與特定訊息，不看含遠端中繼資料的 stdout |
| 頁面內容拼成 url | Telegram post／頻道名、即刻 id、YouTube 頻道 id 以白名單正則驗證（OPML 的 rsshub 路由同樣過白名單） |
| 遠端標題注入 workflow 指令 | QC 的 `::warning::` 先跳脫 `%`、CR、LF、`::` |
| 提交範圍過寬 | `updatenews.yml` 只 `git add data/archive.json`；`.gitignore` 排除 `data/context/`（閱讀器的 token、cookies、OPML） |
| `opml_secret.html` 洩漏訂閱清單／注入 | 不用 `innerHTML`；CSP `default-src 'none'`，頁面不能連網 |
| 存入 `javascript:` 等連結 | 條目 url 與縮圖只收 http(s) |
| 亂碼／機翻垃圾公開發布 | `decode_body`；`garbled` 擋在萃取、摘要、翻譯三處，回填清除存量 |
| 閱讀器加入的網址 | `fetch_inbox` 只收 http(s)、無帳密、限長；抓標題同樣走 `net.get` |
| ReDoS／二次方解析 | 讀遠端 HTML 的標籤（meta、img、link、title、`__NEXT_DATA__`）一律用 `store.iter_tags`／`tag_text` 線性掃描；Telegram 預覽頁用 `HTMLParser` 串流，不用樹狀解析器（對巢狀貼文是二次方：8,000 層約 6 分鐘），屬性正則用 possessive 量詞；`html_to_text`、`region_text`、段落退路（`textproc.paragraph_text`）與 `elements` 共用 `_Stream` 串流核心，不建樹（`elements` 是扁平清單，孤立結束標籤以計數忽略、不回掃堆疊）（樹狀解析器在深層巢狀上 `append`／逐段 `get_text` 是二次方）；`sanitize_markup` 的註解與 script/style 區塊以 `find` 移除（在 `store.ascii_lower` 的副本上找：`str.lower()` 會改變長度，如 `İ`，位移後會切掉內文；`store.tag_text`、`media` 的音檔網址掃描同理），其餘標籤正則以 `[^<>]*+` 止於下一個 `<`；markdown／PTT 清理不讓 `\s*` 跨行或從空白串中段起跑；`strip_code` 只處理最外層。這些做法是為了避免 `<meta a`、`<script a`、`<img`、巢狀 `<p>` 等輸入讓天真的正則或建樹解析器在 8 MB 頁面上耗費數小時到數天。selftest 對每個解析器餵 19 種惡意樣式 |
| workflow 指令注入／secret 外洩 | secret 只經 `env` 傳入；cookies 以 `umask 077` 寫到 `/dev/shm`，用完即刪；OPML 只在記憶體解開；cookie 值遮罩 |
| token 被任意步驟取用 | `persist-credentials: false`；`permissions: {}`，job 只給 `contents: write`；token 只出現在 push 那一步 |
| 供應鏈 | actions 釘 commit SHA；Python 套件釘版本；服務映像釘版本 |
| 並行寫入互相覆蓋 | 共用 concurrency 群組；rebase 衝突直接失敗，不用 `-X ours`（那會靜默丟掉另一方的資料） |

殘餘風險：reader proxy／Wayback 由第三方代抓（它們連到哪裡不在我們控制內，被抓的網址也會送給它們）；yt-dlp 自己的連線不經 `net.request`（只剩 YouTube 網址）；
pip 未用雜湊驗證、間接依賴未釘版；服務映像只釘 tag 未釘 digest；前端須自行跳脫 `title`／`summary`。

## 9. 執行

兩個入口都是「一個行程做完、archive 寫一次」：`collect.py` 蒐集後直接摘要與回填；`media.py` 下載字幕／ASR 後直接摘要新逐字稿（只處理本機字幕與逐字稿，不抓網頁、不回填；這個 job 沒有 feed 副本），影片在同一輪就有摘要。兩個 workflow 各自排程，同屬 `archive-writer` 並行群組，不會同時寫 `archive.json`。參數一律來自環境變數（兩個 yml 的 `env` 已設好），命令列只用來切換模式。

| 工作 | 指令 |
|---|---|
| 蒐集＋摘要＋回填 | `python scripts/collect.py [--rss-opml follow.opml]` |
| 只蒐集 | `python scripts/collect.py --no-summarize …`（feed 副本隨行程結束而消失） |
| 字幕／podcast＋摘要 | `python scripts/media.py [--no-transcribe] [--no-summarize]` |
| 回填 | `python scripts/summarize.py backfill [--limit N] [--no-translate]` |
| 摘要 QC 報告 | `python scripts/summarize.py backfill --dry-run`（不寫檔） |
| 只套用 QC 修正 | `python scripts/summarize.py backfill --qc-only` |
| 挖樣板規則 | `python scripts/summarize.py mine N` |
| 檢查 | `python scripts/selftest.py` |

**log 格式**（兩個 workflow 都設 `PYTHONUNBUFFERED=1`，逐行輸出，job 逾時被終止也不會丟掉最後幾行）：
- `WARNING:`／`ERROR:`：需要注意的事；`::warning::`：QC 的 Actions 註記
- `[TAG] …`（大寫）：整輪層級的狀態，如 `[OPML]`、`[FEEDS]`、`[INBOX]`、`[ARCHIVE]`、`[RETENTION]`、`[SUMMARIZE]`、`[BACKFILL]`、`[MEDIA]`、`[CLEANUP]`、`[ASR]`、`[TRACK]`
- `[route] (日期) 標題`（小寫）：summarize 的單筆條目開頭；其下縮排 4 格的行是該筆的細節，包括可直接統計的 `via=reader ok|miss`、`via=wayback ok|miss`
- 工作執行緒的細節先寫進自己的緩衝（`summarize.log`），由主執行緒連同結果一起印出；`rules.json` 的規則在主執行緒先編譯，壞規則的 `WARNING:` 只印一次、不夾在條目之間

環境變數（全部由 `store.Config.from_env` 讀取；數字格式錯誤時印出警告並用預設值）：
- 路徑：`DATA_DIR`（預設 `data`）；`archive.json`、`subtitles/`、`status.json`、`inbox.json` 都在其下
- 蒐集與摘要：`ARCHIVE_DAYS` `RSS_MAX_FEEDS` `SCRAPED_SOURCES` `MAX_ITEMS` `TIME_BUDGET_SECONDS` `TRANSLATE` `SUMMARY_RATIO` `SUMMARY_MAX` `FALLBACKS` `DEEPL_API_KEY`（程式預設：`MAX_ITEMS` 200、`RSS_MAX_FEEDS` 400）
- 字幕：`COOKIES_PATH`（檔案不存在就不帶 cookies）`MAX_ASR_TOTAL_SECONDS`（預設 28800）`MAX_TRANSCRIBE`（預設 40）`MAX_ASR_DURATION_SECONDS`（預設 10800）`WHISPER_MODEL` `WHISPER_COMPUTE_TYPE`；每輪最多看 400 個條目（`media.Limits`），依發布時間新到舊
- OPML secret（`FOLLOW_OPML_B64*`）不進 `Config`，由 `collect.opml_from_env` 直接在記憶體解開

Secrets：`FOLLOW_OPML_B64`（必要時加 `_2`…`_5`）、`DEEPL_API_KEY`、`YT_COOKIES`。

**OPML 只放在 secret**。單一 secret 上限 48 KB，所以用 `opml_secret.html` 打包：
在瀏覽器開啟、按按鈕選 OPML 所在資料夾，會在同一資料夾寫出 `FOLLOW_OPML_B64.txt`（必要時 `_2.txt`…），
每個 txt 是同名 secret 的值。打包內容：只留 `xmlUrl` `htmlUrl` `title` `category`，gzip，base64，每段 ≤47,000 字元。
寫入資料夾需要 Chrome／Edge；其他瀏覽器改選檔案，txt 會下載到「下載」資料夾。
`collect.py` 沒給 `--rss-opml` 時從環境變數合併解開（gzip、xz、純 base64 都接受），只在記憶體中，不落地；
找不到 OPML 時直接失敗，不碰 archive。

## 10. 已知盲區

| 盲區 | 說明 |
|---|---|
| 摘要品質 | 選錯句子不會報錯，只能人工抽樣 |
| 站台改版 | 表現為 `↛` 比例上升或摘要變導覽列文字 |
| 第三方服務 | reader proxy、Wayback、gtx 皆無 SLA |
| 大量匯入 | 單輪新增遠超 `MAX_ITEMS` 時，未處理條目的 feed 副本隨行程結束而消失 |
| 中途被殺 | 存檔節流，job 逾時被強制終止最多遺失約 30 秒的進度（下輪重做；寫入是原子的，檔案不會損壞） |
| podcast 辨識品質 | `downloadsub.yml` 使用 `WHISPER_MODEL=small`（程式預設仍為 `base`）；比 `base` 中文錯字少，但轉錄速度約慢一倍，`MAX_ASR_TOTAL_SECONDS` 以音訊秒數計，不隨模型調整，實際耗時需對照 job 時限 |
| podcast 失敗無上限 | ASR 一直失敗的單集（音檔下載失敗、格式無法解碼、超過 3 小時上限）或單集頁一直讀不到的，每天重試，永遠沒有摘要，直到留存期滿被移出 archive。失敗不計入 ASR 預算（預算只算成功的音訊秒數），但下載與轉錄失敗每次都耗用 workflow 執行時間；已知時長超過上限的在下載前就被擋下，幾乎不耗時。`downloadsub.yml` 停擺或預算長期被佔滿時亦同 |

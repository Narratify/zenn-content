---
title: "クランクを回して、AIの言葉を一行ずつ読む——CrankBotの設計と思想"
emoji: "🔩"
type: "tech"
topics: ["playdate", "ai", "lua", "python", "chatbot"]
published: true
---

![CrankBot Demo](https://raw.githubusercontent.com/Narratify/CrankBot/main/docs/demo.gif)

夜中の二時、部屋の明かりを落としてクランクを回す。カリカリという小さな抵抗が指先に伝わってきて、画面の文字が一行ずつせり上がる。白と黒しかない。400×240ピクセル、1ビット。AIの言葉が、手の動きと同じ速度で現れる。

これが[CrankBot](https://github.com/Narratify/CrankBot)——Playdate用のAIチャットボット。Lua約500行とPython約80行。MITライセンス。

## すべてのAIインターフェースが同じ顔をしている

テキストボックス、送信ボタン、ブラウザ上のチャット画面。高解像度ディスプレイに高速で流れるトークン。技術は進歩し、体験は収束していく。

逆に行ってみたかった。

[Playdate](https://play.date/)という小さな黄色いゲーム機がある。400×240のモノクロディスプレイ、十字キー、ボタン2つ、そして横に付いた手回しのクランク。$229。こいつでAIと会話できるようにした。

[AI-MY](https://ai-my.net)では「アンチイノベーション／再発見」をテーマにプロダクトを作っている——テクノロジーの進歩が見落としてきたものを、テクノロジーで再発見する試み。[Lo-Fi Camera](https://note.com/deepcore_kernel/n/n445bfc8d562a)（Claude Hackathon 2025 3位 + Anthropic賞）は被写体をドット絵に変換して感熱紙に印刷する。ハイレゾが奪った想像力——それをもう一度、手に触れられるものにする。CrankBotも同じ衝動から生まれた。

AIの進歩が見落としてきたもの。それは「読む」という行為そのものかもしれない。

400×240の白黒1ビット画面で、クランクを回して一行ずつ送っていくと——一語一語を*読む*ようになる。流し読みしない。コピーしない。「再生成」もない。あるのは自分と、クランクと、AIの言葉だけ。

## アーキテクチャ: 意図的にシンプル

```
┌──────────────┐     HTTPS      ┌──────────────┐     API      ┌──────────────┐
│   Playdate   │ ────────────▶  │   FastAPI    │ ──────────▶  │   LLM API    │
│   (Lua)      │ ◀────────────  │   (Python)   │ ◀──────────  │  (Claude等)  │
└──────────────┘    JSON resp   └──────────────┘  Completions └──────────────┘
```

Playdateのオンスクリーンキーボードでメッセージを入力し、HTTPS経由でセルフホストのPythonサーバーに送る。サーバーがOpenAI互換のChat Completions APIを叩き、応答をJSONで返す。プロバイダはAnthropic（Claude）、OpenAI、Google（Gemini）、Groq——何でもいい。環境変数を書き換えるだけ。

コード量はこれで全部。500行と80行。

## 制約との対話

Playdate向けに書くということは、制約の中で設計するということ。そしてときどき、制約のほうが正しい判断をしている。

### 1ビットのテキスト描画

グレースケールなし、アンチエイリアスなし。全ピクセルが白か黒。テキストがくっきり読めなければ話にならないので、Playdate内蔵のRoobert 11pxを使った。マージンとスクロールバーを除いた使えるテキスト幅は約380px。プロポーショナルフォントなので、ワードラップは文字数ではなくピクセル幅で判定する:

```lua
local function wrapText(text, maxWidth, font)
    local lines = {}
    for segment in text:gmatch("([^\n]*)\n?") do
        if segment == "" then
            lines[#lines + 1] = ""
        else
            local line = ""
            for word in segment:gmatch("%S+") do
                local test = line == "" and word or (line .. " " .. word)
                if font:getTextWidth(test) > maxWidth and line ~= "" then
                    lines[#lines + 1] = line
                    line = word
                else
                    line = test
                end
            end
            if line ~= "" then lines[#lines + 1] = line end
        end
    end
    return lines
end
```

`font:getTextWidth()`がピクセル単位の幅を返す。380pxに収まるかどうか、単語ごとに確認して折り返す。

### クランクスクロール

`playdate.getCrankChange()`がクランクの回転角度を返す。これをスクロールオフセットに変換する:

```lua
local change = playdate.getCrankChange()
if change ~= 0 then
    scrollY = scrollY + change * 0.5
    scrollY = math.max(0, math.min(scrollY, maxScroll))
end
```

`0.5`の係数は体感で決めた。大きすぎると文字が流れて読めない。小さすぎると長い応答で手が疲れる。物理的な回転と読む速度が同じリズムになるところ——ちょうどそこを探った。

### メモリとスライディングウィンドウ

Playdateのメモリは限られている。全会話履歴を保持するとメモリがあふれるので、直近6往復だけをサーバーに送る:

```lua
local MAX_HISTORY <const> = 6

local function buildHistoryJson()
    local parts = {}
    local start = math.max(1, #history - MAX_HISTORY + 1)
    for i = start, #history do
        local h = history[i]
        parts[#parts + 1] = '{"role":"' .. h.role
            .. '","content":"' .. jsonEscape(h.content) .. '"}'
    end
    return "[" .. table.concat(parts, ",") .. "]"
end
```

JSONライブラリは使っていない。Playdate SDKに標準搭載されていないし、この程度なら文字列結合で十分。AIは会話を覚えている——しばらくの間は。パーティーで隣に座った人との会話みたいに、やがて忘れる。

### Playdate SDKの罠

最も厄介だったのはネットワーク周り。Playdate SDKのHTTPはコールバックベースだがメインスレッドで動く。通信中もUI更新が必要なので、15フレームごとにドットが増える「Sending...」アニメーションを表示した。

そしてもう一つ——**Playdate SDKはHTTP 200以外のステータスコードでレスポンスコールバックが発火しない**。サーバー側で全レスポンスを200で返す必要がある:

```python
@app.exception_handler(HTTPException)
async def always_200(request, exc):
    """Playdate SDKの仕様: 200以外だとコールバックが呼ばれない"""
    return JSONResponse(
        status_code=200,
        content={"response": f"[Error] {exc.detail}", "error": True},
    )
```

バグなのか仕様なのかわからない。ただ、知らないとデバッグで数時間を失う。

## サーバー: Python 80行

FastAPIの単一ファイル。設定は環境変数だけ:

```python
LLM_BASE_URL = os.environ.get("LLM_BASE_URL", "https://api.anthropic.com/v1")
LLM_MODEL = os.environ.get("LLM_MODEL", "claude-sonnet-4-20250514")
```

システムプロンプトで応答を300文字以内に制限している。400×240の画面に長文は入らない。Bearerトークン認証で、知らないリクエストがAPIの課金を膨らませるのを防ぐ。

これだけ。80行のサーバーが、小さな黄色い箱とLLMの間に立っている。

## 遅さが変えるもの

CrankBotを使っていて、一番意外だったこと。

キーボード入力が遅い。Playdateのオンスクリーンキーボードは長文入力を想定していない。だから何を聞くかを慎重に考える。言葉を選ぶようになる。応答が返ってきたら、クランクで一行ずつ送る。ストリーミングトークンもプログレスバーもない。待つ。読む。

「不便」が「丁寧」に変わる瞬間がある。

制約が体験を改善するなんて、もっともらしい嘘かもしれない。けれど深夜にクランクを回していると、画面の小ささが——不思議と——言葉の重さを変えるのがわかる。ブラウザの中では流れていくだけの文字列が、ここでは一行ごとに手の動きと結びつく。

低解像度の写真がハイレゾより*リアル*に感じられることがある——想像力を要求するから。Lo-Fi AI体験、というものがあるとしたら、たぶんこういうことだ。情報が減ることで、何かが戻ってくる。

それが何なのかは、まだ正確には言えないけれど。

## 試す

**[github.com/Narratify/CrankBot](https://github.com/Narratify/CrankBot)**（MIT）

必要なもの:
1. [Playdate](https://play.date/) 本体
2. [Playdate SDK](https://play.date/dev/)（無料）
3. Python 3 + LLM APIキー

`main.lua`のHOSTとAUTH_TOKENを書き換えて、`pdc`でビルドして、サイドロード。あなたのPlaydate、あなたのAI、あなたのサーバー。

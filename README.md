# hom-paifu

HOM リーグの牌譜置き場。雀魂から落ちてきたバイト列をそのまま置いてるだけ。

- `raw/<uuid>.bin` … 1試合1ファイル。加工なし
- `json/<uuid>.json` … 同じ牌譜を JSON にほどいたもの
- `games.json` … 試合の一覧（誰がどの席か、点数、時刻）

対戦日の夜中に勝手に増える。

## json/

読むだけならこっちが楽。一手が一行。

```
{"name": ".lq.GameDetailRecords", "version": 210715, "actions": [
{"passed":1161,"type":3,"user_event":{"seat":2,"type":1}},
...
{"passed":...,"type":1,"result":{"name":".lq.RecordDiscardTile","data":{"seat":0,"tile":"9s",...}}},
...
]}
```

- `actions[].result` … 一手ぶんの記録。`name` が型、`data` が中身
- `passed` … 対局が始まってからのミリ秒

bin から作り直したものなので、bin に戻してもバイト単位では一致しない（中身は同じ）。届いたままの形がほしいときは `raw/` の方。

## 中身は protobuf

外側が `lq.Wrapper`、中が `lq.GameDetailRecords`。その中にまた `lq.Wrapper` が一手ずつ並んでる。

何が入ってるか見るだけなら、ライブラリなしでいける。

```python
# peek.py
import sys, collections

def varint(b, i):
    x = s = 0
    while True:
        c = b[i]; i += 1
        x |= (c & 127) << s; s += 7
        if c < 128:
            return x, i

def fields(b):
    """protobuf を (フィールド番号, 値) にばらすだけ"""
    i = 0
    while i < len(b):
        key, i = varint(b, i)
        num, wire = key >> 3, key & 7
        if wire == 0:   v, i = varint(b, i)
        elif wire == 2: n, i = varint(b, i); v = b[i:i+n]; i += n
        elif wire == 1: v = b[i:i+8]; i += 8
        elif wire == 5: v = b[i:i+4]; i += 4
        else: raise ValueError(wire)
        yield num, v

def unwrap(b):
    """lq.Wrapper → (name, data)"""
    d = dict(fields(b))
    return d[1].decode(), d.get(2, b"")

name, data = unwrap(open(sys.argv[1], "rb").read())
print(name)                              # .lq.GameDetailRecords

count = collections.Counter()
for num, v in fields(data):
    if num == 3:                         # actions（新しい形式）
        act = dict(fields(v))
        if act.get(3):                   # result に、一手ぶんの記録が入ってる
            count[unwrap(act[3])[0]] += 1
    elif num == 1:                       # records（古い形式）
        count[unwrap(v)[0]] += 1
for k, n in count.most_common():
    print(n, k)
```

```
$ python peek.py raw/260905-411d7b72-c74a-488a-98dd-0dc245285018.bin
.lq.GameDetailRecords
603 .lq.RecordDiscardTile
574 .lq.RecordDealTile
20 .lq.RecordChiPengGang
11 .lq.RecordNewRound
8 .lq.RecordHule
3 .lq.RecordNoTile
```

中身までちゃんと読むなら、protobufjs と `liqi.json`（雀魂の型定義。https://github.com/takayama-lily/mjsoul にあるやつ）。

```js
// read.js
const protobuf = require("protobufjs");
const fs = require("fs");

const root = protobuf.Root.fromJSON(require("./liqi.json"));
const Wrap = root.lookupType("lq.Wrapper");
const GDR  = root.lookupType("lq.GameDetailRecords");

const gd = GDR.decode(Wrap.decode(fs.readFileSync(process.argv[2])).data);
const frames = [];
for (const a of gd.actions || []) if (a.result && a.result.length) frames.push(a.result);
for (const r of gd.records || []) if (r.length) frames.push(r);

for (const f of frames) {
  const w = Wrap.decode(f);
  const rec = root.lookupType(w.name.replace(/^\./, "")).decode(w.data);
  console.log(w.name, JSON.stringify(rec));
}
```

```
$ npm i protobufjs
$ node read.js raw/260905-411d7b72-c74a-488a-98dd-0dc245285018.bin
.lq.RecordNewRound {"chang":0,"ju":0,"ben":0,"scores":[25000,25000,25000,25000], ...
```

## games.json

```
{
  "contest_id": 657782,
  "games": [
    {
      "uuid": "260905-...",
      "file": "raw/260905-....bin",
      "start_time": UNIX秒, "end_time": UNIX秒,
      "rounds": 局数,
      "accounts": [{"seat": 0, "nickname": "...", "account_id": ...}, ...],
      "result":   [{"seat": 0, "total_point": ..., "score": ...}, ...]
    }
  ]
}
```

古い順に並んでる。

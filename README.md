# constraint-llm-agent

[参考1](https://aclanthology.org/2024.lrec-main.444.pdf)  
[参考2](https://aclanthology.org/2024.findings-emnlp.374.pdf)  
[参考3](https://aclanthology.org/2023.ijcnlp-srw.10.pdf)  
[参考4](https://www.anlp.jp/proceedings/annual_meeting/2025/pdf_dir/Q7-11.pdf)  
[参考5](https://github.com/facebookresearch/MUSE)  

---

source .venv/bin/activate

## 全体ワークフロー

1. **初期化**
2. **LLM によるアクション提案**
3. **制約検証（CLP チェック）**
4. **自己修正ループ**
5. **アクション適用**
6. **完了判定／次ステップ**

---

### 1. 初期化

* **world\_model.py** を読み込み、空のワールド状態を生成

  * ブロック配置状態を保持するデータ構造（例：辞書や Datalog の述語）を初期化
  * エージェント初期座標 `(ax,ay,az)` を設定
* **main.py** でタスク定義（例：吊り橋を構築）を読み込む

  * 目標構造を YAML/JSON 形式でパース

### 2. LLM によるアクション提案

* **call\_llm.py** の関数 `generate_actions(prompt, history)` を呼び出し

  * `prompt`：

    * タスク説明（例：「吊り橋を架けるためにまず土台を3つ横に置いてください」）
    * 直前までのアクション履歴（再提示時は「前回××で失敗したので…」を追加）
  * **出力**：LLM のテキストレスポンス（例：`PLACE block=stone AT (1,0,0)` のリスト）

### 3. 制約検証（CLP チェック）

* 提案された各アクションをパースして構造体化
* **world\_model.py** の関数 `is_valid(action)` を呼び出し

  * **Horn clause / Datalog** で定義した制約（衝突・サポート・リーチ）を順番に評価
* **出力**：アクションごとに `OK` または `Violation(code, details)` を返却

### 4. 自己修正ループ

* **Violation** が得られたら：

  1. `code`（例：`collision`）と `details`（例：`(1,0,0) に既存 block がある`）を受け取り
  2. `explain_violation(code, details)` で自然言語化（「その位置には既にブロックがあるため、衝突します」）
  3. フォローアップ用 `prompt` を生成：

     > 「前回のアクションで“既存ブロックとの衝突”が起きました。別の座標で同じブロックを配置してください」
  4. **再度** `generate_actions(prompt, history)` を呼び出し
  5. 最大３ラウンドまで試行し、成功したアクションが返るまでループ

### 5. アクション適用

* `is_valid(action) == OK` になったアクションのみを

  * `apply_action(action)` でワールド状態に反映
  * 履歴リストに追加

### 6. 完了判定／次ステップ

* 現在のワールド状態が **タスク定義**（例：吊り橋の完成条件）を満たすか判定
* 2Dの絵での評価も面白そう

  * 満たしていなければ **ステップ2** に戻り、「次に何を置くか」を LLM に聞く
  * 満たしていれば終了

---

## モジュール対応表

| ステップ     | モジュール           | 主な関数・処理                                   |
| -------- | --------------- | ----------------------------------------- |
| 初期化      | world\_model.py | `init_world()`, `load_task_definition()`  |
| LLM 呼び出し | call\_llm.py    | `generate_actions(prompt, history)`       |
| 制約検証     | world\_model.py | `is_valid(action)`, `explain_violation()` |
| 自己修正     | main.py (制御ループ) | フォローアッププロンプト生成                            |
| 適用       | world\_model.py | `apply_action(action)`                    |
| 完了判定     | world\_model.py | `is_task_complete()`                      |

---

このように、「LLM ⇆ CLP エンジン ⇆ 自己修正ループ」を主軸に、段階的にアクションを積み上げていきます。

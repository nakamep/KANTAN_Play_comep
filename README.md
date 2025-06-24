# KANTAN Play Core（かんぷれ）アプリケーション  
# KANTAN Play Core Application

---

## 概要 / Overview

KANTAN Play Core（通称：かんぷれ）は、M5Stack Core2 以降の機種で動作するオープンソース音楽ガジェットです。  
このプログラムは、専用ハードウェア「KANTAN Play base」と組み合わせてご利用ください。  
KANTAN Play baseは、スイッチ類、MIDI音源、アンプ回路、バッテリー、各種インターフェイスを内蔵した専用デバイスです。  
KANTAN Play baseが接続されていない場合は、プログラムは起動しません。

KANTAN Play Core is an open-source music gadget for M5Stack Core2 or later devices.  
This program is designed to be used together with the dedicated hardware "KANTAN Play base."  
KANTAN Play base is a special hardware device that includes switches, a MIDI sound source, amplifier circuit, battery, and various interfaces.  
The program will not start if KANTAN Play base is not connected.

---

## 特徴 / Features

- 指一本で簡単にコード演奏が可能  
  Play chords easily with just one finger  
- オープンソースによる自由なカスタマイズ  
  Freely customizable thanks to open-source design  
- ソフトウェアと専用ハードウェアの組み合わせに最適化  
  Optimized for use with dedicated hardware  

---

## インストール方法 / Installation

1. このリポジトリをクローンします。  
   Clone this repository.
2. 必要な依存関係をインストールします。  
   Install required dependencies.
3. ビルドしてM5Stack Core2（以降）に書き込みます。  
   Build and flash to your M5Stack Core2 (or later).

---

## ライセンス / License

- このリポジトリ全体はMITライセンスの下で公開されています。詳細は [LICENSE](./LICENSE) をご覧ください。  
  The main repository is licensed under the MIT License. See [LICENSE](./LICENSE) for details.

- `main/kantan-music/` フォルダ内の KANTAN Music API は、別のライセンス条件が適用されます。詳細は [main/kantan-music/LICENSE_KANTAN_MUSIC.md](./main/kantan-music/LICENSE_KANTAN_MUSIC.md) をご覧ください。  
  The `main/kantan-music/` folder contains the KANTAN Music API, which is subject to a separate license. See [main/kantan-music/LICENSE_KANTAN_MUSIC.md](./main/kantan-music/LICENSE_KANTAN_MUSIC.md) for details.

---

## アーキテクチャ / Architecture

### 設計原則 / Design Principles

#### 1. 通信タスクとリソースの1対1マッピング / One-to-One Mapping of Communication Tasks and Resources

各通信を行うタスクは、専用の通信リソースと1対1で結び付けられています。  
Each task that performs communication is bound one-to-one with its own communication resource.

**利点 / Benefits:**
- 同一バスに対するタスク間でのmutex/排他制御が不要  
  Eliminates the need for mutex/exclusion handling between tasks for the same bus
- 各バスが常に並列動作可能  
  Allows each bus to always operate in parallel

#### 2. system_registryによるタスク間協調 / Inter-Task Coordination via system_registry

更新時にタスクに通知する機能を持つカスタムデータストアで、すべてのランタイムパラメータを一元管理します。  
A custom data store that also notifies tasks on updates, centralizing all runtime parameters.

**利点 / Benefits:**
- すべてのパラメータを1箇所で管理することで、あらゆるデータの所在を明確化  
  Clarifies where every piece of data lives by managing all parameters in one place
- 新機能はsystem_registryの読み書きだけで既存フローに簡単に組み込み可能  
  New features can hook into existing flows simply by reading/updating system_registry
- ロジックとレンダリングの分離：レンダリングはsystem_registryの状態のみを反映  
  Separates logic and rendering: rendering only reflects system_registry state
- 外部からsystem_registryを更新することで完全な外部制御が可能  
  Enables full external control by updating system_registry from outside sources

### 処理シーケンス概要 / Processing Sequence Overview

メインボタン押下からMIDI送信までのシーケンスを示します。  
Shows sequence from main button press through to MIDI transmission.

```
[task_i2c] → [internal_input] → [task_commander] → [command_request] 
→ [task_kantanplay] → [midi_out_control] → [task_midi] → [subtask_midi] → (hardware buses)
```

**詳細フロー / Detailed Flow:**

1. **task_i2c → internal_input**  
   スケジュールに従ってボタン状態を読み取り → internal_inputに状態を登録  
   Read button state on schedule → register state in internal_input

2. **internal_input → task_commander**  
   新しい状態をtask_commanderに通知  
   Notify task_commander of new state

3. **task_commander → command_request**  
   状態を内部コマンドに変換 → command_requestに登録  
   Convert state into internal command → register in command_request

4. **command_request → task_kantanplay**  
   更新されたコマンドをtask_kantanplayに通知  
   Notify task_kantanplay of updated command

5. **task_kantanplay → midi_out_control**  
   再生ロジックを実行 → MIDIペイロードをmidi_out_controlに登録  
   Perform playback logic → register MIDI payload in midi_out_control

6. **midi_out_control → task_midi**  
   新しいMIDIペイロードをtask_midiに通知  
   Notify task_midi of new MIDI payload

7. **task_midi → subtask_midi**  
   サブタスクを再開 & External-MIDI、BLE-MIDI、USB-MIDIに振り分け  
   Resume subtasks & dispatch to External-MIDI, BLE-MIDI, USB-MIDI

8. **subtask_midi → (hardware buses)**  
   各バス上で実際のMIDIメッセージを送信  
   Send actual MIDI messages over each bus

**注記 / Notes:**
- アクター = FreeRTOSタスク；参加者 = system_registryのセクション  
  Actors = FreeRTOS tasks; Participants = sections in system_registry
- 追加タスク：task_spi（レンダリング＆TFカード）、task_i2s（オーディオ入力）  
  Additional tasks: task_spi (rendering & TF-card), task_i2s (audio input)
- 画面再描画とLED制御も同様にsystem_registryの状態によって駆動  
  Screen redraw and LED control likewise driven by system_registry state

---

## お問い合わせ / Contact

- 技術的なご質問は、[GitHub Issues](https://github.com/InstaChord/KANTAN_Play_core/issues) からご連絡ください。
- その他のお問い合わせは、[公式WEBサイトのコンタクトフォーム](https://instachord.com/contact/) よりご連絡ください。

For technical questions, please use [GitHub Issues](https://github.com/InstaChord/KANTAN_Play_core/issues).
For other inquiries, please contact us via our [official website contact form](https://en.instachord.com/#contact).


---


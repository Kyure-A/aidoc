# predict-on 関数の詳細解説

## 概要
`predict-on` は、入力中の文字に基づいて履歴から自動的にコマンドを予測・展開する「魔法の履歴検索」機能を実装した Zsh 関数です。

## 主要関数の説明

### 1. `predict-on()` - 予測機能を有効化
```bash
predict-on() {
  zle -N self-insert insert-and-predict          # 通常の文字入力を予測機能付きに置き換え
  zle -N magic-space insert-and-predict          # スペース入力も予測機能付きに
  zle -N backward-delete-char delete-backward-and-predict  # バックスペースも予測対応
  zle -N delete-char-or-list delete-no-predict   # 削除キーは予測無効
  zstyle -t :predict verbose && zle -M predict-on # verboseモードならメッセージ表示
  return 0
}
```
標準のキー入力機能を予測機能付きのものに置き換えます。

### 2. `predict-off()` - 予測機能を無効化
```bash
predict-off() {
  zle -A .self-insert self-insert                # 元の機能に戻す
  zle -A .magic-space magic-space
  zle -A .backward-delete-char backward-delete-char
  zstyle -t :predict verbose && zle -M predict-off
  return 0
}
```
キー入力を元の標準機能に戻します。

### 3. `insert-and-predict()` - メイン予測ロジック
```bash
insert-and-predict () {
  # 複数行編集中やペースト中は予測を無効化
  if [[ $LBUFFER == *$'\012'* ]] || (( PENDING )); then
    zstyle -t ":predict" toggle && predict-off
    zle .$WIDGET "$@"
    return
  
  # 入力文字と右側の文字が同じなら単にカーソル移動
  elif [[ ${RBUFFER[1]} == ${KEYS[-1]} ]]; then
    ((++CURSOR))
  
  else
    LBUFFER="$LBUFFER$KEYS"  # 入力文字を追加
    
    # 直前の操作が適切なものかチェック
    if [[ $LASTWIDGET == (self-insert|magic-space|backward-delete-char) || ... ]]; then
      # 履歴から検索
      if ! zle .history-beginning-search-backward; then
        RBUFFER=""  # 見つからなければ右側をクリア
        
        # スペース以外なら補完を試行
        if [[ ${KEYS[-1]} != ' ' ]]; then
          unsetopt automenu recexact
          comppostfuncs=( predict-limit-list )  # 補完リストを制限
          zle complete-word
          
          # カーソル位置の決定
          # - complete: 補完が決めた位置
          # - key: 入力文字のn番目の出現位置
          # - その他: 元の位置
        fi
      fi
    else
      zstyle -t ":predict" toggle && predict-off
    fi
  fi
}
```

### 4. `delete-backward-and-predict()` - バックスペース処理
```bash
delete-backward-and-predict() {
  if (( $#LBUFFER > 1 )); then
    # 複数行編集中や直前が特定操作以外なら予測無効
    if [[ 複数行 || 不適切な直前操作 ]]; then
      zstyle -t ":predict" toggle && predict-off
      LBUFFER="$LBUFFER[1,-2]"  # 最後の文字を削除
    else
      ((--CURSOR))  # カーソルを戻す
      zle .history-beginning-search-forward || RBUFFER=""  # 前方検索
    fi
  else
    zle .kill-whole-line  # 行全体を削除
  fi
}
```

### 5. `predict-limit-list()` - 補完リスト制限
```bash
predict-limit-list() {
  # 補完候補が多すぎる場合はリスト表示を抑制
  if (( 行数超過 || 最大数超過 )); then
    compstate[list]=''
  elif zstyle -t ":predict" list always; then
    compstate[list]='force list'  # 常に表示
  fi
}
```

## 動作の特徴
1. **即時展開**: 文字を入力するたびに履歴から一致する行を検索し、即座に展開
2. **補完統合**: 履歴に一致しない場合は通常の補完に切り替え
3. **スマート制御**: 複数行編集やペースト時は自動的に無効化
4. **カーソル制御**: 予測後のカーソル位置を柔軟に設定可能（complete/key/デフォルト）

## 使用上の注意
- 行の途中での編集は予測を混乱させ、行の残りを削除する可能性がある
- TABキーで「興味深い」文字位置（通常は単語の終わり）へジャンプ可能
- 完全な行が得られたらカーソルを最後まで移動せずにRETURNで確定可能
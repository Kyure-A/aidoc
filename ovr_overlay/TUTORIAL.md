# ovr_overlay チュートリアル

このチュートリアルでは、`ovr_overlay`クレートを使用してOpenVRオーバーレイアプリケーションを作成する方法を説明します。

## 目次

1. [はじめに](#はじめに)
2. [前提条件](#前提条件)
3. [セットアップ](#セットアップ)
4. [基本的な使い方](#基本的な使い方)
5. [オーバーレイの作成と表示](#オーバーレイの作成と表示)
6. [オーバーレイのプロパティ設定](#オーバーレイのプロパティ設定)
7. [高度な機能](#高度な機能)
8. [エラーハンドリング](#エラーハンドリング)
9. [ベストプラクティス](#ベストプラクティス)

## はじめに

`ovr_overlay`は、Rustで書かれたOpenVRオーバーレイ作成用のバインディングライブラリです。このライブラリは、`autocxx`を使用してC++ APIをラップしており、主にオーバーレイ機能に焦点を当てています。

### なぜovr_overlayを使うのか？

- **オーバーレイ特化**: OpenXRではまだサポートされていないオーバーレイ機能に特化
- **安全性**: RustとC++の型安全性を活用
- **最新のOpenVR**: 最新のOpenVR C++ APIを使用

## 前提条件

開発を始める前に、以下の要件を満たしていることを確認してください：

1. **Rust**: 1.71.1以上
2. **libclang**: `cxx`と`autocxx`クレートに必要
3. **SteamVR**: インストールされ、実行可能な状態
4. **Git**: サブモジュールのクローンに必要

## セットアップ

### 1. プロジェクトの作成

```bash
cargo new my_vr_overlay
cd my_vr_overlay
```

### 2. 依存関係の追加

`Cargo.toml`に以下を追加：

```toml
[dependencies]
ovr_overlay = { git = "https://github.com/TheButlah/ovr_overlay.git" }
log = "0.4"
env_logger = "0.10"
```

### 3. リポジトリのクローン（開発用）

開発に貢献する場合は、サブモジュールを含めてクローン：

```bash
git clone --recursive https://github.com/TheButlah/ovr_overlay.git
# または
git clone https://github.com/TheButlah/ovr_overlay.git
cd ovr_overlay
git submodule update --init
```

## 基本的な使い方

### 最小限の例

```rust
use ovr_overlay::Context;

fn main() -> Result<(), Box<dyn std::error::Error>> {
    // ログの初期化（オプション）
    env_logger::init();

    // OpenVRコンテキストの初期化
    let context = Context::init()?;
    
    // オーバーレイマネージャーの取得
    let mut overlay_mgr = context.overlay_mngr();
    
    // アプリケーションロジック
    
    // 安全にシャットダウン
    unsafe {
        context.shutdown();
    }
    
    Ok(())
}
```

## オーバーレイの作成と表示

### 基本的なオーバーレイの作成

```rust
use ovr_overlay::{Context, overlay::OverlayHandle};

fn create_simple_overlay() -> Result<(), Box<dyn std::error::Error>> {
    let context = Context::init()?;
    let mut overlay_mgr = context.overlay_mngr();
    
    // オーバーレイの作成
    // key: ユニークな識別子（他のアプリと重複しないように）
    // friendly_name: ユーザーに表示される名前
    let overlay_handle = overlay_mgr.create_overlay(
        "com.example.my_overlay",
        "My VR Overlay"
    )?;
    
    // オーバーレイを表示
    overlay_mgr.set_visibility(overlay_handle, true)?;
    
    // メインループ（実際のアプリケーションではイベント処理などが入る）
    std::thread::sleep(std::time::Duration::from_secs(10));
    
    unsafe { context.shutdown(); }
    Ok(())
}
```

## オーバーレイのプロパティ設定

### 曲率の設定

```rust
// 0.0 = 平面（クアッド）
// 1.0 = 円筒形
overlay_mgr.set_curvature(overlay_handle, 0.3)?;

// 現在の曲率を取得
let current_curvature = overlay_mgr.curvature(overlay_handle)?;
```

### 不透明度の設定

```rust
// 0.0 = 完全に透明
// 1.0 = 完全に不透明
overlay_mgr.set_opacity(overlay_handle, 0.8)?;
```

### 色の調整

```rust
use ovr_overlay::ColorTint;

let color_tint = ColorTint {
    r: 1.0,  // 赤チャンネル
    g: 0.8,  // 緑チャンネル
    b: 0.8,  // 青チャンネル
    a: 1.0,  // アルファチャンネル
};
overlay_mgr.set_color_tint(overlay_handle, color_tint)?;
```

### サイズと位置の設定

```rust
// 幅の設定（メートル単位）
overlay_mgr.set_width(overlay_handle, 2.0)?;

// HMD相対位置の設定
use ovr_overlay::pose::{Matrix3x4, TrackingUniverseOrigin};

let transform = Matrix3x4::identity(); // 変換行列を適切に設定
overlay_mgr.set_transform_tracked_device_relative(
    overlay_handle,
    TrackedDeviceIndex::HMD,
    &transform
)?;
```

## 高度な機能

### フィーチャーフラグ

`ovr_overlay`は複数のオプション機能を提供：

```toml
[dependencies]
ovr_overlay = { 
    git = "https://github.com/TheButlah/ovr_overlay.git",
    features = ["ovr_input", "ovr_system", "ovr_applications"] 
}
```

利用可能なフィーチャー：
- `ovr_applications`: アプリケーション管理
- `ovr_chaperone_setup`: シャペロン設定
- `ovr_input`: 入力処理
- `ovr_system`: システム情報

### 入力処理（ovr_inputフィーチャー使用時）

```rust
#[cfg(feature = "ovr_input")]
fn handle_input(context: &Context) {
    let input_mgr = context.input_mngr();
    // 入力処理のロジック
}
```

### システム情報の取得（ovr_systemフィーチャー使用時）

```rust
#[cfg(feature = "ovr_system")]
fn get_system_info(context: &Context) {
    let system_mgr = context.system_mngr();
    // システム情報の取得
}
```

## エラーハンドリング

### 基本的なエラー処理

```rust
use ovr_overlay::errors::{InitError, EVROverlayError};

match Context::init() {
    Ok(context) => {
        // 成功時の処理
    }
    Err(InitError::AlreadyInitialized) => {
        eprintln!("OpenVRは既に初期化されています");
    }
    Err(e) => {
        eprintln!("初期化エラー: {:?}", e);
    }
}
```

### オーバーレイ操作のエラー処理

```rust
match overlay_mgr.create_overlay("key", "name") {
    Ok(handle) => {
        println!("オーバーレイが作成されました");
    }
    Err(EVROverlayError(error_code)) => {
        eprintln!("オーバーレイ作成エラー: {:?}", error_code);
    }
}
```

## ベストプラクティス

### 1. 適切なシャットダウン

```rust
// 必ずunsafeブロック内でshutdownを呼ぶ
unsafe {
    context.shutdown();
}
```

### 2. ユニークなオーバーレイキー

```rust
// 逆ドメイン記法を使用
let overlay_key = "com.yourcompany.yourapp.overlay_name";
```

### 3. エラーログの活用

```rust
use log::{info, error};

fn main() {
    env_logger::init();
    
    match Context::init() {
        Ok(_) => info!("OpenVR初期化成功"),
        Err(e) => error!("OpenVR初期化失敗: {:?}", e),
    }
}
```

### 4. リソースの適切な管理

```rust
struct OverlayApp {
    context: Context,
    overlay_handle: OverlayHandle,
}

impl Drop for OverlayApp {
    fn drop(&mut self) {
        // クリーンアップロジック
        info!("オーバーレイアプリケーションを終了します");
    }
}
```

## 完全な例

```rust
use ovr_overlay::{Context, ColorTint, overlay::OverlayHandle};
use log::{info, error};
use std::time::Duration;
use std::thread;

fn main() -> Result<(), Box<dyn std::error::Error>> {
    env_logger::init();
    
    info!("VRオーバーレイアプリケーションを開始します");
    
    // OpenVR初期化
    let context = Context::init()?;
    let mut overlay_mgr = context.overlay_mngr();
    
    // オーバーレイ作成
    let overlay = overlay_mgr.create_overlay(
        "com.example.tutorial_overlay",
        "Tutorial Overlay"
    )?;
    
    // プロパティ設定
    overlay_mgr.set_visibility(overlay, true)?;
    overlay_mgr.set_width(overlay, 2.0)?;
    overlay_mgr.set_curvature(overlay, 0.2)?;
    overlay_mgr.set_opacity(overlay, 0.9)?;
    
    let tint = ColorTint {
        r: 1.0,
        g: 1.0,
        b: 1.0,
        a: 1.0,
    };
    overlay_mgr.set_color_tint(overlay, tint)?;
    
    info!("オーバーレイが作成され、表示されました");
    
    // 10秒間実行
    thread::sleep(Duration::from_secs(10));
    
    info!("シャットダウン中...");
    unsafe {
        context.shutdown();
    }
    
    Ok(())
}
```

## トラブルシューティング

### よくある問題

1. **ビルドエラー**: `libclang`がインストールされているか確認
2. **初期化エラー**: SteamVRが実行されているか確認
3. **オーバーレイが表示されない**: キーが他のアプリと重複していないか確認

### デバッグのヒント

- `RUST_LOG=debug`環境変数を設定してログレベルを上げる
- SteamVRのコンソールでエラーメッセージを確認
- 最小限の例から始めて段階的に機能を追加

## 次のステップ

- [公式リポジトリ](https://github.com/TheButlah/ovr_overlay)でより多くの例を確認
- OpenVR公式ドキュメントで詳細なAPI情報を学ぶ
- イシューやプルリクエストでプロジェクトに貢献

## 実用的なオーバーレイ機能

### テクスチャとコンテンツの表示

#### 画像ファイルの表示

```rust
use std::ffi::CString;

fn set_overlay_image(
    overlay_mgr: &mut OverlayManager,
    overlay: OverlayHandle,
    image_path: &str
) -> Result<(), Box<dyn std::error::Error>> {
    let c_path = CString::new(image_path)?;
    overlay_mgr.set_image(overlay, &c_path)?;
    Ok(())
}
```

#### 生のピクセルデータの表示

```rust
// RGBA形式の画像データを作成
fn create_rgba_texture(width: usize, height: usize) -> Vec<u8> {
    let mut data = vec![0u8; width * height * 4];
    
    // 例: グラデーションを作成
    for y in 0..height {
        for x in 0..width {
            let idx = (y * width + x) * 4;
            data[idx] = (x * 255 / width) as u8;     // R
            data[idx + 1] = (y * 255 / height) as u8; // G
            data[idx + 2] = 128;                       // B
            data[idx + 3] = 255;                       // A
        }
    }
    data
}

// テクスチャを設定
let texture_data = create_rgba_texture(512, 512);
overlay_mgr.set_raw_data(
    overlay,
    &texture_data,
    512,
    512,
    4  // RGBA = 4 bytes per pixel
)?;
```

### テキストレンダリング

OpenVRは直接的なテキストレンダリングAPIを提供していないため、外部ライブラリを使用する必要があります。

```rust
// 依存関係を追加
// [dependencies]
// image = "0.24"
// imageproc = "0.23"
// rusttype = "0.9"

use image::{Rgba, RgbaImage};
use imageproc::drawing::{draw_text_mut, text_size};
use rusttype::{Font, Scale};

fn render_text_to_overlay(
    overlay_mgr: &mut OverlayManager,
    overlay: OverlayHandle,
    text: &str,
    font_size: f32,
) -> Result<(), Box<dyn std::error::Error>> {
    // フォントの読み込み
    let font_data = include_bytes!("../assets/fonts/DejaVuSans.ttf");
    let font = Font::try_from_bytes(font_data).unwrap();
    
    // テキストサイズを計算
    let scale = Scale::uniform(font_size);
    let (text_width, text_height) = text_size(scale, &font, text);
    
    // 画像バッファを作成
    let mut image = RgbaImage::new(
        (text_width + 20) as u32,
        (text_height + 20) as u32
    );
    
    // 背景を半透明の黒に
    for pixel in image.pixels_mut() {
        *pixel = Rgba([0, 0, 0, 200]);
    }
    
    // テキストを描画
    draw_text_mut(
        &mut image,
        Rgba([255, 255, 255, 255]),
        10,
        10,
        scale,
        &font,
        text
    );
    
    // オーバーレイに設定
    let raw_data = image.into_raw();
    overlay_mgr.set_raw_data(
        overlay,
        &raw_data,
        (text_width + 20) as usize,
        (text_height + 20) as usize,
        4
    )?;
    
    Ok(())
}
```

### インタラクティブ要素（ボタンの実装）

注意: 現在の`ovr_overlay`クレートには、イベント処理APIが実装されていない可能性があります。
以下は、将来的な実装を想定した例です。

```rust
// ボタンの状態を管理する構造体
struct Button {
    x: f32,
    y: f32,
    width: f32,
    height: f32,
    text: String,
    is_hovered: bool,
    is_pressed: bool,
}

impl Button {
    fn new(x: f32, y: f32, width: f32, height: f32, text: String) -> Self {
        Self {
            x, y, width, height, text,
            is_hovered: false,
            is_pressed: false,
        }
    }
    
    fn contains_point(&self, x: f32, y: f32) -> bool {
        x >= self.x && x <= self.x + self.width &&
        y >= self.y && y <= self.y + self.height
    }
    
    fn render(&self, width: u32, height: u32) -> Vec<u8> {
        use image::{Rgba, RgbaImage};
        use imageproc::rect::Rect;
        use imageproc::drawing::{draw_filled_rect_mut, draw_text_mut};
        
        let mut image = RgbaImage::new(width, height);
        
        // ボタンの背景色を状態に応じて変更
        let bg_color = if self.is_pressed {
            Rgba([100, 100, 100, 255])
        } else if self.is_hovered {
            Rgba([80, 80, 80, 255])
        } else {
            Rgba([60, 60, 60, 255])
        };
        
        // ボタンを描画
        draw_filled_rect_mut(
            &mut image,
            Rect::at(self.x as i32, self.y as i32)
                .of_size(self.width as u32, self.height as u32),
            bg_color
        );
        
        // テキストを中央に描画（実装は省略）
        
        image.into_raw()
    }
}

// UIマネージャー
struct UIManager {
    buttons: Vec<Button>,
    overlay_width: u32,
    overlay_height: u32,
}

impl UIManager {
    fn handle_mouse_move(&mut self, x: f32, y: f32) {
        for button in &mut self.buttons {
            button.is_hovered = button.contains_point(x, y);
        }
    }
    
    fn handle_mouse_click(&mut self, x: f32, y: f32) -> Option<usize> {
        for (index, button) in self.buttons.iter_mut().enumerate() {
            if button.contains_point(x, y) {
                button.is_pressed = true;
                return Some(index);
            }
        }
        None
    }
    
    fn render(&self) -> Vec<u8> {
        let mut image = RgbaImage::new(self.overlay_width, self.overlay_height);
        
        // 背景を描画
        for pixel in image.pixels_mut() {
            *pixel = Rgba([30, 30, 30, 230]);
        }
        
        // 各ボタンを描画
        for button in &self.buttons {
            // ボタンのレンダリングロジック
        }
        
        image.into_raw()
    }
}
```

### 通知オーバーレイの作成

```rust
use std::time::{Duration, Instant};

struct NotificationOverlay {
    context: Context,
    overlay: OverlayHandle,
    show_until: Option<Instant>,
}

impl NotificationOverlay {
    fn new(message: &str, duration: Duration) -> Result<Self, Box<dyn std::error::Error>> {
        let context = Context::init()?;
        let mut overlay_mgr = context.overlay_mngr();
        
        // 通知用オーバーレイを作成
        let overlay = overlay_mgr.create_overlay(
            "com.example.notification",
            "Notification"
        )?;
        
        // テキストをレンダリング
        render_text_to_overlay(&mut overlay_mgr, overlay, message, 24.0)?;
        
        // 通知の位置を設定（画面右上）
        let mut transform = Matrix3x4::identity();
        transform.set_translation(1.0, 1.5, -1.0);
        overlay_mgr.set_transform_tracked_device_relative(
            overlay,
            TrackedDeviceIndex::HMD,
            &transform
        )?;
        
        // サイズと表示設定
        overlay_mgr.set_width(overlay, 0.3)?;
        overlay_mgr.set_opacity(overlay, 0.9)?;
        overlay_mgr.set_visibility(overlay, true)?;
        
        Ok(Self {
            context,
            overlay,
            show_until: Some(Instant::now() + duration),
        })
    }
    
    fn update(&mut self) -> bool {
        if let Some(show_until) = self.show_until {
            if Instant::now() > show_until {
                let mut overlay_mgr = self.context.overlay_mngr();
                let _ = overlay_mgr.set_visibility(self.overlay, false);
                return false; // 通知を終了
            }
        }
        true // 通知を継続
    }
}
```

### ダッシュボードオーバーレイ

```rust
// ダッシュボードに表示されるオーバーレイ
fn create_dashboard_overlay(
    context: &Context
) -> Result<OverlayHandle, Box<dyn std::error::Error>> {
    let mut overlay_mgr = context.overlay_mngr();
    
    let overlay = overlay_mgr.create_overlay(
        "com.example.dashboard",
        "My Dashboard Widget"
    )?;
    
    // ダッシュボード用の設定
    // 注: set_dashboard_overlay等のAPIが実装されている場合
    // overlay_mgr.set_dashboard_overlay(overlay)?;
    
    overlay_mgr.set_width(overlay, 2.0)?;
    overlay_mgr.set_curvature(overlay, 0.0)?; // フラット
    
    Ok(overlay)
}
```

### マルチオーバーレイ管理

```rust
struct OverlaySystem {
    context: Context,
    main_overlay: OverlayHandle,
    notification_overlay: Option<OverlayHandle>,
    menu_overlay: Option<OverlayHandle>,
}

impl OverlaySystem {
    fn new() -> Result<Self, Box<dyn std::error::Error>> {
        let context = Context::init()?;
        let mut overlay_mgr = context.overlay_mngr();
        
        // メインオーバーレイ
        let main_overlay = overlay_mgr.create_overlay(
            "com.example.main",
            "Main Display"
        )?;
        
        Ok(Self {
            context,
            main_overlay,
            notification_overlay: None,
            menu_overlay: None,
        })
    }
    
    fn show_menu(&mut self) -> Result<(), Box<dyn std::error::Error>> {
        let mut overlay_mgr = self.context.overlay_mngr();
        
        if self.menu_overlay.is_none() {
            let menu = overlay_mgr.create_overlay(
                "com.example.menu",
                "Menu"
            )?;
            self.menu_overlay = Some(menu);
        }
        
        if let Some(menu) = self.menu_overlay {
            overlay_mgr.set_visibility(menu, true)?;
            
            // メインオーバーレイの相対位置に配置
            let mut transform = Matrix3x4::identity();
            transform.set_translation(0.5, 0.0, 0.0);
            overlay_mgr.set_transform_overlay_relatve(
                menu,
                self.main_overlay,
                &transform
            )?;
        }
        
        Ok(())
    }
}
```

### パフォーマンス最適化

```rust
// テクスチャ更新の最適化
struct OptimizedOverlay {
    overlay: OverlayHandle,
    texture_data: Vec<u8>,
    width: usize,
    height: usize,
    dirty: bool,
}

impl OptimizedOverlay {
    fn new(
        overlay_mgr: &mut OverlayManager,
        width: usize,
        height: usize
    ) -> Result<Self, Box<dyn std::error::Error>> {
        let overlay = overlay_mgr.create_overlay(
            "com.example.optimized",
            "Optimized Overlay"
        )?;
        
        let texture_data = vec![0u8; width * height * 4];
        
        Ok(Self {
            overlay,
            texture_data,
            width,
            height,
            dirty: false,
        })
    }
    
    fn update_region(&mut self, x: usize, y: usize, w: usize, h: usize, data: &[u8]) {
        // 部分的な更新
        for dy in 0..h {
            for dx in 0..w {
                let src_idx = (dy * w + dx) * 4;
                let dst_idx = ((y + dy) * self.width + (x + dx)) * 4;
                
                self.texture_data[dst_idx..dst_idx + 4]
                    .copy_from_slice(&data[src_idx..src_idx + 4]);
            }
        }
        self.dirty = true;
    }
    
    fn flush(&mut self, overlay_mgr: &mut OverlayManager) -> Result<(), EVROverlayError> {
        if self.dirty {
            overlay_mgr.set_raw_data(
                self.overlay,
                &self.texture_data,
                self.width,
                self.height,
                4
            )?;
            self.dirty = false;
        }
        Ok(())
    }
}

// フレームレート制限
struct FrameRateLimiter {
    target_fps: u32,
    last_frame: Instant,
}

impl FrameRateLimiter {
    fn new(target_fps: u32) -> Self {
        Self {
            target_fps,
            last_frame: Instant::now(),
        }
    }
    
    fn limit(&mut self) {
        let target_duration = Duration::from_secs_f64(1.0 / self.target_fps as f64);
        let elapsed = self.last_frame.elapsed();
        
        if elapsed < target_duration {
            thread::sleep(target_duration - elapsed);
        }
        
        self.last_frame = Instant::now();
    }
}
```

## まとめ

このチュートリアルでは、`ovr_overlay`クレートの基本的な使い方から実用的な機能まで説明しました。テキストレンダリング、インタラクティブ要素、通知システム、マルチオーバーレイ管理など、実際のVRアプリケーション開発に必要な機能を実装する方法を示しました。

注意: 一部の機能（イベント処理など）は、現在の`ovr_overlay`クレートには実装されていない可能性があります。これらの機能が必要な場合は、プロジェクトへの貢献を検討してください。
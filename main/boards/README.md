# カスタム開発ボードガイド

このガイドでは、シャオジーAI音声チャットボットプロジェクト用に新しい開発ボードの初期化プログラムをカスタマイズする方法について説明します。シャオジーAIは70種類以上のESP32シリーズ開発ボードをサポートしており、各開発ボードの初期化コードは対応するディレクトリに配置されています。

## 重要事項

> **警告**: カスタム開発ボードでIO構成が元の開発ボードと異なる場合、元の開発ボードの構成を直接上書きしてファームウェアをコンパイルしないでください。新しい開発ボードタイプを作成するか、config.jsonファイルのbuilds構成で異なるnameとsdkconfigマクロ定義を使用して区別する必要があります。`python scripts/release.py [開発ボードのディレクトリ名]` を使用してファームウェアをコンパイルおよびパッケージ化します。
>
> 元の構成を直接上書きすると、将来OTAアップデート時に、カスタムファームウェアが元の開発ボードの標準ファームウェアで上書きされ、デバイスが正常に動作しなくなる可能性があります。各開発ボードには一意の識別子と対応するファームウェアアップデートチャネルがあるため、開発ボードの識別子の一意性を保つことが非常に重要です。

## ディレクトリ構造

各開発ボードのディレクトリ構造には、通常、次のファイルが含まれます：

- `xxx_board.cc` - 主要なボードレベルの初期化コード。ボード関連の初期化と機能を実装します。
- `config.h` - ボードレベルの構成ファイル。ハードウェアのピンマッピングやその他の構成項目を定義します。
- `config.json` - コンパイル構成。ターゲットチップと特別なコンパイルオプションを指定します。
- `README.md` - 開発ボード関連の説明ドキュメント。

## カスタム開発ボードの手順

### 1. 新しい開発ボードディレクトリの作成

まず、`boards/`ディレクトリに新しいディレクトリを作成します。例：`my-custom-board/`

```bash
mkdir main/boards/my-custom-board
```

### 2. 構成ファイルの作成

#### config.h

`config.h`ですべてのハードウェア構成を定義します。これには以下が含まれます：

- オーディオのサンプルレートとI2Sピン構成
- オーディオコーデックチップのアドレスとI2Cピン構成
- ボタンとLEDのピン構成
- ディスプレイのパラメータとピン構成

参考例（lichuang-c3-devより）：

```c
#ifndef _BOARD_CONFIG_H_
#define _BOARD_CONFIG_H_

#include <driver/gpio.h>

// オーディオ構成
#define AUDIO_INPUT_SAMPLE_RATE  24000
#define AUDIO_OUTPUT_SAMPLE_RATE 24000

#define AUDIO_I2S_GPIO_MCLK GPIO_NUM_10
#define AUDIO_I2S_GPIO_WS   GPIO_NUM_12
#define AUDIO_I2S_GPIO_BCLK GPIO_NUM_8
#define AUDIO_I2S_GPIO_DIN  GPIO_NUM_7
#define AUDIO_I2S_GPIO_DOUT GPIO_NUM_11

#define AUDIO_CODEC_PA_PIN       GPIO_NUM_13
#define AUDIO_CODEC_I2C_SDA_PIN  GPIO_NUM_0
#define AUDIO_CODEC_I2C_SCL_PIN  GPIO_NUM_1
#define AUDIO_CODEC_ES8311_ADDR  ES8311_CODEC_DEFAULT_ADDR

// ボタン構成
#define BOOT_BUTTON_GPIO        GPIO_NUM_9

// ディスプレイ構成
#define DISPLAY_SPI_SCK_PIN     GPIO_NUM_3
#define DISPLAY_SPI_MOSI_PIN    GPIO_NUM_5
#define DISPLAY_DC_PIN          GPIO_NUM_6
#define DISPLAY_SPI_CS_PIN      GPIO_NUM_4

#define DISPLAY_WIDTH   320
#define DISPLAY_HEIGHT  240
#define DISPLAY_MIRROR_X true
#define DISPLAY_MIRROR_Y false
#define DISPLAY_SWAP_XY true

#define DISPLAY_OFFSET_X  0
#define DISPLAY_OFFSET_Y  0

#define DISPLAY_BACKLIGHT_PIN GPIO_NUM_2
#define DISPLAY_BACKLIGHT_OUTPUT_INVERT true

#endif // _BOARD_CONFIG_H_
```

#### config.json

`config.json`でコンパイル構成を定義します：

```json
{
    "target": "esp32s3",  // ターゲットチップモデル: esp32, esp32s3, esp32c3など
    "builds": [
        {
            "name": "my-custom-board",  // 開発ボード名
            "sdkconfig_append": [
                // 追加で必要なコンパイル構成
                "CONFIG_ESPTOOLPY_FLASHSIZE_8MB=y",
                "CONFIG_PARTITION_TABLE_CUSTOM_FILENAME=\"partitions/v1/8m.csv\""
            ]
        }
    ]
}
```

### 3. ボードレベルの初期化コードの記述

`my_custom_board.cc`ファイルを作成し、開発ボードのすべての初期化ロジックを実装します。

基本的な開発ボードクラスの定義には、次のいくつかの部分が含まれます：

1. **クラス定義**：`WifiBoard`または`Ml307Board`から継承します。
2. **初期化関数**：I2C、ディスプレイ、ボタン、IoTなどのコンポーネントの初期化を含みます。
3. **仮想関数のオーバーライド**：`GetAudioCodec()`、`GetDisplay()`、`GetBacklight()`など。
4. **開発ボードの登録**：`DECLARE_BOARD`マクロを使用して開発ボードを登録します。

```cpp
#include "wifi_board.h"
#include "audio_codecs/es8311_audio_codec.h"
#include "display/lcd_display.h"
#include "application.h"
#include "button.h"
#include "config.h"
#include "iot/thing_manager.h"

#include <esp_log.h>
#include <driver/i2c_master.h>
#include <driver/spi_common.h>

#define TAG "MyCustomBoard"

// フォントの宣言
LV_FONT_DECLARE(font_puhui_16_4);
LV_FONT_DECLARE(font_awesome_16_4);

class MyCustomBoard : public WifiBoard {
private:
    i2c_master_bus_handle_t codec_i2c_bus_;
    Button boot_button_;
    LcdDisplay* display_;

    // I2Cの初期化
    void InitializeI2c() {
        i2c_master_bus_config_t i2c_bus_cfg = {
            .i2c_port = I2C_NUM_0,
            .sda_io_num = AUDIO_CODEC_I2C_SDA_PIN,
            .scl_io_num = AUDIO_CODEC_I2C_SCL_PIN,
            .clk_source = I2C_CLK_SRC_DEFAULT,
            .glitch_ignore_cnt = 7,
            .intr_priority = 0,
            .trans_queue_depth = 0,
            .flags = {
                .enable_internal_pullup = 1,
            },
        };
        ESP_ERROR_CHECK(i2c_new_master_bus(&i2c_bus_cfg, &codec_i2c_bus_));
    }

    // SPIの初期化（ディスプレイ用）
    void InitializeSpi() {
        spi_bus_config_t buscfg = {};
        buscfg.mosi_io_num = DISPLAY_SPI_MOSI_PIN;
        buscfg.miso_io_num = GPIO_NUM_NC;
        buscfg.sclk_io_num = DISPLAY_SPI_SCK_PIN;
        buscfg.quadwp_io_num = GPIO_NUM_NC;
        buscfg.quadhd_io_num = GPIO_NUM_NC;
        buscfg.max_transfer_sz = DISPLAY_WIDTH * DISPLAY_HEIGHT * sizeof(uint16_t);
        ESP_ERROR_CHECK(spi_bus_initialize(SPI2_HOST, &buscfg, SPI_DMA_CH_AUTO));
    }

    // ボタンの初期化
    void InitializeButtons() {
        boot_button_.OnClick([this]() {
            auto& app = Application::GetInstance();
            if (app.GetDeviceState() == kDeviceStateStarting && !WifiStation::GetInstance().IsConnected()) {
                ResetWifiConfiguration();
            }
            app.ToggleChatState();
        });
    }

    // ディスプレイの初期化（ST7789を例として）
    void InitializeDisplay() {
        esp_lcd_panel_io_handle_t panel_io = nullptr;
        esp_lcd_panel_handle_t panel = nullptr;
        
        esp_lcd_panel_io_spi_config_t io_config = {};
        io_config.cs_gpio_num = DISPLAY_SPI_CS_PIN;
        io_config.dc_gpio_num = DISPLAY_DC_PIN;
        io_config.spi_mode = 2;
        io_config.pclk_hz = 80 * 1000 * 1000;
        io_config.trans_queue_depth = 10;
        io_config.lcd_cmd_bits = 8;
        io_config.lcd_param_bits = 8;
        ESP_ERROR_CHECK(esp_lcd_new_panel_io_spi(SPI2_HOST, &io_config, &panel_io));

        esp_lcd_panel_dev_config_t panel_config = {};
        panel_config.reset_gpio_num = GPIO_NUM_NC;
        panel_config.rgb_ele_order = LCD_RGB_ELEMENT_ORDER_RGB;
        panel_config.bits_per_pixel = 16;
        ESP_ERROR_CHECK(esp_lcd_new_panel_st7789(panel_io, &panel_config, &panel));
        
        esp_lcd_panel_reset(panel);
        esp_lcd_panel_init(panel);
        esp_lcd_panel_invert_color(panel, true);
        esp_lcd_panel_swap_xy(panel, DISPLAY_SWAP_XY);
        esp_lcd_panel_mirror(panel, DISPLAY_MIRROR_X, DISPLAY_MIRROR_Y);
        
        // ディスプレイオブジェクトの作成
        display_ = new SpiLcdDisplay(panel_io, panel,
                                    DISPLAY_WIDTH, DISPLAY_HEIGHT, 
                                    DISPLAY_OFFSET_X, DISPLAY_OFFSET_Y, 
                                    DISPLAY_MIRROR_X, DISPLAY_MIRROR_Y, DISPLAY_SWAP_XY,
                                    {
                                        .text_font = &font_puhui_16_4,
                                        .icon_font = &font_awesome_16_4,
                                        .emoji_font = font_emoji_32_init(),
                                    });
    }

    // IoTデバイスの初期化
    void InitializeIot() {
        auto& thing_manager = iot::ThingManager::GetInstance();
        thing_manager.AddThing(iot::CreateThing("Speaker"));
        thing_manager.AddThing(iot::CreateThing("Screen"));
        // さらに多くのIoTデバイスを追加できます
    }

public:
    // コンストラクタ
    MyCustomBoard() : boot_button_(BOOT_BUTTON_GPIO) {
        InitializeI2c();
        InitializeSpi();
        InitializeDisplay();
        InitializeButtons();
        InitializeIot();
        GetBacklight()->SetBrightness(100);
    }

    // オーディオコーデックの取得
    virtual AudioCodec* GetAudioCodec() override {
        static Es8311AudioCodec audio_codec(
            codec_i2c_bus_, 
            I2C_NUM_0, 
            AUDIO_INPUT_SAMPLE_RATE, 
            AUDIO_OUTPUT_SAMPLE_RATE,
            AUDIO_I2S_GPIO_MCLK, 
            AUDIO_I2S_GPIO_BCLK, 
            AUDIO_I2S_GPIO_WS, 
            AUDIO_I2S_GPIO_DOUT, 
            AUDIO_I2S_GPIO_DIN,
            AUDIO_CODEC_PA_PIN, 
            AUDIO_CODEC_ES8311_ADDR);
        return &audio_codec;
    }

    // ディスプレイの取得
    virtual Display* GetDisplay() override {
        return display_;
    }
    
    // バックライト制御の取得
    virtual Backlight* GetBacklight() override {
        static PwmBacklight backlight(DISPLAY_BACKLIGHT_PIN, DISPLAY_BACKLIGHT_OUTPUT_INVERT);
        return &backlight;
    }
};

// 開発ボードの登録
DECLARE_BOARD(MyCustomBoard);
```

### 4. README.mdの作成

README.mdで、開発ボードの特性、ハードウェア要件、コンパイルおよび書き込み手順を説明します。


## 一般的な開発ボードコンポーネント

### 1. ディスプレイ

プロジェクトは、次のようなさまざまなディスプレイドライバをサポートしています：
- ST7789 (SPI)
- ILI9341 (SPI)
- SH8601 (QSPI)
- など...

### 2. オーディオコーデック

サポートされているコーデックには以下が含まれます：
- ES8311 (一般的)
- ES7210 (マイクアレイ)
- AW88298 (パワーアンプ)
- など...

### 3. 電源管理

一部の開発ボードでは、電源管理チップを使用しています：
- AXP2101
- その他の利用可能なPMIC

### 4. MCPデバイス制御

AIが使用できるように、さまざまなMCPツールを追加できます：
- Speaker (スピーカー制御)
- Screen (画面の明るさ調整)
- Battery (バッテリー残量の読み取り)
- Light (ライト制御)
- など...

## 開発ボードのクラス継承関係

- `Board` - 基本的なボードレベルのクラス
  - `WifiBoard` - Wi-Fi接続の開発ボード
  - `Ml307Board` - 4Gモジュールを使用する開発ボード
  - `DualNetworkBoard` - Wi-Fiと4Gネットワークの切り替えをサポートする開発ボード

## 開発のヒント

1. **類似の開発ボードを参考にする**：新しい開発ボードが既存の開発ボードと類似している場合は、既存の実装を参考にすることができます。
2. **段階的なデバッグ**：まず基本的な機能（表示など）を実装し、次に複雑な機能（オーディオなど）を追加します。
3. **ピンマッピング**：config.hですべてのピンマッピングが正しく構成されていることを確認します。
4. **ハードウェアの互換性を確認する**：すべてのチップとドライバの互換性を確認します。

## 発生する可能性のある問題

1. **ディスプレイの異常**：SPI構成、ミラー設定、およびカラー反転設定を確認します。
2. **オーディオ出力がない**：I2S構成、PA有効化ピン、およびコーデックアドレスを確認します。
3. **ネットワークに接続できない**：Wi-Fiの認証情報とネットワーク構成を確認します。
4. **サーバーと通信できない**：MQTTまたはWebSocketの構成を確認します。

## 参考資料

- ESP-IDF ドキュメント: https://docs.espressif.com/projects/esp-idf/
- LVGL ドキュメント: https://docs.lvgl.io/
- ESP-SR ドキュメント: https://github.com/espressif/esp-sr
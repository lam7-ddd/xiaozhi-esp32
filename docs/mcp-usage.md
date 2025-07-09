# MCPプロトコルによるIoT制御の使用法

> このドキュメントでは、MCPプロトコルに基づいてESP32デバイスのIoT制御を実装する方法について説明します。詳細なプロトコルフローについては、[`mcp-protocol.md`](./mcp-protocol.md)を参照してください。

## 概要

MCP（Model Context Protocol）は、IoT制御に推奨される新世代のプロトコルです。標準のJSON-RPC 2.0形式を使用して、バックエンドとデバイス間で「ツール」を発見および呼び出し、柔軟なデバイス制御を実現します。

## 一般的な使用フロー

1. デバイスは起動後、基本プロトコル（WebSocket/MQTTなど）を介してバックエンドとの接続を確立します。
2. バックエンドは、MCPプロトコルの`initialize`メソッドを介してセッションを初期化します。
3. バックエンドは、`tools/list`を介して、デバイスがサポートするすべてのツール（機能）とそのパラメータの説明を取得します。
4. バックエンドは、`tools/call`を介して特定のツールを呼び出し、デバイスの制御を実現します。

詳細なプロトコル形式とインタラクションについては、[`mcp-protocol.md`](./mcp-protocol.md)を参照してください。

## デバイス側でのツール登録方法の説明

デバイスは`McpServer::AddTool`メソッドを介して、バックエンドから呼び出すことができる「ツール」を登録します。その一般的な関数シグネチャは次のとおりです。

```cpp
void AddTool(
    const std::string& name,           // ツール名。一意で階層的な名前を推奨します（例：self.dog.forward）。
    const std::string& description,    // ツールの説明。大規模モデルが理解しやすいように機能を簡潔に説明します。
    const PropertyList& properties,    // 入力パラメータのリスト（空でも可）。サポートされる型：ブール値、整数、文字列。
    std::function<ReturnValue(const PropertyList&)> callback // ツールが呼び出されたときのコールバック実装。
);
```
- `name`：ツールの一意の識別子。「モジュール.機能」の命名スタイルを推奨します。
- `description`：AI/ユーザーが理解しやすい自然言語の説明。
- `properties`：パラメータのリスト。ブール値、整数、文字列の型をサポートし、範囲とデフォルト値を指定できます。
- `callback`：呼び出しリクエストを受信したときの実際の実行ロジック。戻り値はbool/int/stringが可能です。

## 一般的な登録例（ESP-Hiを例として）

```cpp
void InitializeTools() {
    auto& mcp_server = McpServer::GetInstance();
    // 例1：パラメータなし、ロボットを前進させる
    mcp_server.AddTool("self.dog.forward", "ロボットを前進させる", PropertyList(), [this](const PropertyList&) -> ReturnValue {
        servo_dog_ctrl_send(DOG_STATE_FORWARD, NULL);
        return true;
    });
    // 例2：パラメータあり、ライトのRGBカラーを設定する
    mcp_server.AddTool("self.light.set_rgb", "RGBカラーを設定する", PropertyList({
        Property("r", kPropertyTypeInteger, 0, 255),
        Property("g", kPropertyTypeInteger, 0, 255),
        Property("b", kPropertyTypeInteger, 0, 255)
    }), [this](const PropertyList& properties) -> ReturnValue {
        int r = properties["r"].value<int>();
        int g = properties["g"].value<int>();
        int b = properties["b"].value<int>();
        led_on_ = true;
        SetLedColor(r, g, b);
        return true;
    });
}
```

## 一般的なツール呼び出しのJSON-RPCの例

### 1. ツールリストの取得
```json
{
  "jsonrpc": "2.0",
  "method": "tools/list",
  "params": { "cursor": "" },
  "id": 1
}
```

### 2. シャーシを前進させる
```json
{
  "jsonrpc": "2.0",
  "method": "tools/call",
  "params": {
    "name": "self.chassis.go_forward",
    "arguments": {}
  },
  "id": 2
}
```

### 3. ライトモードの切り替え
```json
{
  "jsonrpc": "2.0",
  "method": "tools/call",
  "params": {
    "name": "self.chassis.switch_light_mode",
    "arguments": { "light_mode": 3 }
  },
  "id": 3
}
```

### 4. カメラの反転
```json
{
  "jsonrpc": "2.0",
  "method": "tools/call",
  "params": {
    "name": "self.camera.set_camera_flipped",
    "arguments": {}
  },
  "id": 4
}
```

## 備考
- ツール名、パラメータ、および戻り値は、デバイス側の`AddTool`登録に準拠してください。
- すべての新しいプロジェクトで、IoT制御にMCPプロトコルを統一して使用することを推奨します。
- 詳細なプロトコルと高度な使用法については、[`mcp-protocol.md`](./mcp-protocol.md)を参照してください。
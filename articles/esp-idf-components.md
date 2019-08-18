Title: M5Stack-IDF = ESP-IDF + arduino の仕掛け
Date: 2018-10-18
Save_as: esp-idf-components.html
Tags: esp32
Slug: esp-idf-components

## M5Stack-IDFの仕掛け

[M5Stack-IDF](https://github.com/m5stack/M5Stack-IDF)はESP-IDFでArduinoとM5Stackのライブラリを使う環境を提供しています。実は[ESP-IDF](https://github.com/espressif/esp-idf)は外部ライブラリを追加して使うメカニズムを持っていてM5Stack-IDFはそれを利用します。
使える外部ライブラリは内部ライブラリとの相性である程度制限されます。

ESP-IDFはビルド(make)時に決め打ちのディレクトリとそのサブディレクトリを見に行きそこにcomponent.mkという名前のファイルがあるかどうか調べます。 もしあるとそれぞれのcomponent.mkの中にある指示をビルドへの指示として追加します。
ESP-IDFのサンプルプログラムを見ると必ずmain/component.mkがあってたいていは中身は空です。 この場合は単にここにもソースがあるよという意味になります。 少し複雑なライブラリでよく使われるのは

```
COMPONENT_ADD_INCLUDEDIRS := include src
COMPONENT_SRCDIRS := src
CXXFLAGS += -fno-rtti -Wno-error=unused-value
```

の3つの変数あたりでこの例の場合component.mkのあるディレクトリ下のsrcの中にソースがあり、includeとsrcの2つのディレクトリをインクルードのサーチパスに加え、c++でのコンパイル時にオプション -fno-rtti -Wno-error=unused-value を追加することを指示しています。 [arduino-esp32](https://github.com/espressif/arduino-esp32)にはさらに複雑な指示のあるcomponent.mkがあります。

決め打ちのディレクトリとしてmakeするディレクトリの中にあるmainとcomponentsがあります。 なのでcomponentsにいろいろなライブラリを並べライブラリのルートとなるところに各ライブラリに対応する
component.mkを置いてやればライブラリ・コンポーネントの追加ができるというわけです。 例えば

```
.
├── Makefile
├── README.md
├── components
│   ├── arduino -> /B/arduino-esp32
│   ├── btstack
│   └── m5stack -> /A/M5Stack-IDF/components/m5stack
├── main
│   ├── component.mk
│   ├── hid_host_demo.c
│   └── main.cpp
└── sdkconfig
```
というようなツリーを作っておいてarduino-esp32, btstack, m5stackにcomponent.mkを書けばこの3つのライブラリをESP-IDF環境下で使えるということになります。 arduino-esp32やm5stackにはすでにcomponent.mkがあるのでそれが使えます。 M5Stack-IDFのサンプルプログラムだと最初から

```
.
├── Makefile
├── README.md
├── components
│   ├── arduino
│   └── m5stack
├── main
│   ├── component.mk
│   └── main.cpp
└── sdkconfig
```
の形になっていてM5Stack-IDFの用意するarduino, m5stackにはそれぞれcomponent.mkが書いてあるというわけです。

## Arduinoのloopとsetup

Arduinoはちょっと特別なライブラリというかプロセスなので1つのコアにピン止めして動かします。 またsetup()で準備してあとはメインループをぐるぐる回る構造なのでESP-IDFで動かす時には次のような手順になります。 ESP-IDFでユーザにコントロールがくるのはapp_main()関数なのでそこから何かの方法で

```
extern "C" void m5_arduino_main()
{
    initArduino();

    xTaskCreatePinnedToCore(loopTask, "loopTask", 8192, NULL, 1, NULL, ARDUINO_RUNNING_CORE);
}
```
といったような関数を呼出してARDUINO_RUNNING_CORE(通常はcore 0[^注1])にピン止めしたloopTask()タスクを起動します。 loopTask()は

```
void loopTask(void *pvParameters)
{
    setup();

    while(1) {
        loop ();
    }
}
```
とArduinoではお馴染みのsetup()とloop()を呼出します。

いくつか[外部ライブラリをつかった例](https://github.com/kazkojima/m5stack-app)を試したのでご参考までに。


### \[追記\] はまりポイント

Arduinoでは動いてたのにESP-IDFのコンポーネント化をしたら動かなくなってはまってしまうことがあります。原因は様々なようでなかなか特定できないのですがいくつかはArduinoのプログラムがESP-IDF側のシステムの初期化やinitArduino()より先に動いてしまうケースでした。 普通にはおきないのですがコンストラクタでArduino側の機能を呼んでしまうようなクラスがあって、その静的なオブジェクトが使われると実行のごく初期に静的イニシャライザでコンストラクタからその機能が呼ばれてしまいます。 純粋なArduino環境ではその時すでにある程度のArduino環境がセットアップされていて問題がなくてもESP-IDF環境だと呼ぶのが早過ぎておかしなことになるようです。

そういうオブジェクトについては例えばsetup()内で動的に確保するとか、コンストラクタ内で呼ぶようなArduino側の機能を別のイニシャライズ用メソッドを作って移し、setup()で呼び出すとかしてなんとかごまかしています。

[^注1]:CONFIG_FREERTOS_UNICOREがセットされている場合。make menuconfig[^注2]でComponent config ---> FreeRTOS ---> [*] Run FreeRTOS only on first core

[^注2]:Arduinoコンポーネントを使う場合、M5Stack-IDF/にあるsdkconfigがArduino用にカスタマイズされているのでそれをコピーして修正していくのが吉

### \[追記2\] 例

もう少し複雑な場合としてBTのスタックBstackをM5Stackで使った例です。
その例ではソースツリーは

```
.
├── Makefile
├── README.md
├── components
│   ├── arduino -> /git/arduino-esp32
│   ├── btstack
│   ├── m5stack -> /git/M5Stack-IDF/components/m5stack
│   └── micro-ecc
├── main
│   ├── Kconfig.projbuild
│   ├── component.mk
│   ├── hid_host_demo.c
│   ├── main.cpp
│   └── spi.c
└── sdkconfig
```
のようになっていてbtstack(https://github.com/bluekitchen/btstack.git)とmicro-ecc(https://github.com/kmackay/micro-ecc.git)をcomponentの下にcloneしています。
btstackの版によってはすでにmicro-eccがbtstackの下に取り込まれているのですがこの例で使っている版では別に置くことが必要でした。

micro-ecc/component.mkの内容は
```
COMPONENT_DEPENDS += micro-ecc
COMPONENT_ADD_INCLUDEDIRS := .
COMPONENT_SRCDIRS := .
CFLAGS += -Wno-format
```
と単純ですがbtstack/component.mkの内容は
```
COMPONENT_DEPENDS += micro-ecc

COMPONENT_ADD_INCLUDEDIRS := \
	3rd-party/bluedroid/decoder/include \
	3rd-party/bluedroid/encoder/include \
	3rd-party/hxcmod-player \
	3rd-party/hxcmod-player/mods \
	3rd-party/md5 \
	3rd-party/yxml \
	src/classic \
	src/ble/gatt-service \
	src/ble \
	src/classic \
	src \
	platform/embedded \
	platform/freertos \
	include \

COMPONENT_SRCDIRS := \
	3rd-party/bluedroid/decoder/srce \
	3rd-party/bluedroid/encoder/srce \
	3rd-party/hxcmod-player \
	3rd-party/hxcmod-player/mods \
	3rd-party/md5 \
	src/ble/gatt-service \
	src/ble \
	src/classic \
	src/ \
	platform/freertos \
	. \

CFLAGS += -Wno-format
```
とちょっと複雑になってしまいました。
最初は適当にCOMPONENT_ADD_INCLUDEDIRS/COMPONENT_SRCDIRSを書いておいて関数がないと言われてはCOMPONENT_SRCDIRSに追加しincludeできないと言われてはCOMPONENT_ADD_INCLUDEDIRSを追加、コンパイル時の警告がエラー扱いされるときにはCFLAGSに警告を出さないようにオプション追加、といった試行錯誤が必要になったケースです。

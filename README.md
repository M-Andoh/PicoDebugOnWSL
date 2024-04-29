# Raspbery Pi pico デバッグ環境構築

## 動作環境
- Wndows10
- wsl2  
  - ubuntu-24.04
- VSCODE

 - デバッグ用pico
   - PCとUSB接続
 - デバッグ対象pico

![alt text](デバッグ構成-1.png)   
(出典：https://datasheets.raspberrypi.com/pico/getting-started-with-pico-JP.pdf)

## 準備
　デバッグ用picoにファームウェア書き込み
上記のUSBケーブルがついている側がSWDデバッグ用のpico になります。picoprobe というプログラムを書き込むことで使えるようになります。

　picoprobe
https://www.raspberrypi.com/documentation/microcontrollers/raspberry-pi-pico.html
https://www.raspberrypi.com/documentation/microcontrollers/raspberry-pi-pico.html
https://github.com/raspberrypi/debugprobe/releases/tag/debugprobe-v2.0.1

debugprobe_on_pico.uf2 をダウンロードし配置

## WSL上にSDKインストール
getting-started-with-pico-JP.pdf(https://datasheets.raspberrypi.com/pico/getting-started-with-pico-JP.pdf)に従ってインストールします。
第一章と第二章は同じ内容ですが、Raspberry Pi上での動作を想定しているため別のOS上で動作を行う場合は注意が必要です。

* vscodeをaptでインストールしようとするがリポジトリがない
  - pico_setup.sh実行時に環境変数SKIP_VSCODE=1を指定
  - vscodeを開き、以下の拡張機能を追加
    * marus25.cortex-debug
    * ms-vscode.cmake-tools
    * ms-vscode.cpptools
  ```
  $ code
  ```

* raspi-configコマンドがない
  - pico_setup.sh実行時に環境変数SKIP_UART=1を指定
  - vscodeに以下の拡張機能を追加
  ```
  $ sudo apt install minicom
  ```

* CMake Error at CMakeLists.txt:34 (message):
  picotool cannot be built because libUSB is not found
  - pkg-configがインストールされていない
  
  ```
  $ sudo apt install pkg-config
  ```

インストール開始
```
$ wget https://raw.githubusercontent.com/raspberrypi/pico-setup/master/pico_setup.sh
$ chmod +x pico_setup.sh
$ SKIP_VSCODE=1 SKIP_UART=1 ./pico_setup.sh
```

##  プログラムの作成

```
$ mkdir -p ~/pico/workspace/pico_test
$ cd ~/pico/workspace/pico_test 
```
以下ファイルを準備

* blink.c
  ```
  #include "pico/stdlib.h"

  int main() {
      const uint LED_PIN = PICO_DEFAULT_LED_PIN;
      gpio_init(LED_PIN);
      gpio_set_dir(LED_PIN, GPIO_OUT);

      bool LED_ON = false;
  　　while (true) {
          gpio_put(LED_PIN, LED_ON);
          LED_ON = !LED_ON;
          sleep_ms(1000);
      }
  }
  ```

* CMakeLists.txt 
  ```
  cmake_minimum_required(VERSION 3.13)

  include(pico_sdk_import.cmake)

  project(blink C CXX ASM)
  set(CMAKE_C_STANDARD 11)
  set(CMAKE_CXX_STANDARD 17)

  pico_sdk_init()

  add_executable(blink blink.c)

  target_link_libraries(blink pico_stdlib)

  pico_add_extra_outputs(blink)
  ```
* pico_sdk_import.cmake
  ```
  $ cp ~/pico/pico-sdk/external/pico_sdk_import.cmake ~/pico/workspace/pico_test/
  ```

## Picoprobeのビルド(pico_setup.shで実施済み)
デバッガ用のPicoに書き込む
```
$ cd ~/pico
$ git clone https://github.com/raspberrypi/$ picoprobe.git
$ cd picoprobe
$ mkdir build
$ cd build
$ cmake ..
$ make -j4
```
picoprobeに書き込み

## gdb-multiarchのインストール(pico_setup.shで実施済み)
gdbでarmをデバッグできるようにする
```
$ sudo apt install gdb-multiarch
```

## OpenOCDのビルド・インストール(pico_setup.shで実施済み)
RP2040、Picoprobeを使う都合上、公式リポジトリからブランチを指定して落とす
ビルドしてインストール
```
cd ~/pico
sudo apt install automake autoconf build-essential texinfo libtool libftdi-dev libusb-1.0-0-dev pkg-config
git clone https://github.com/raspberrypi/openocd.git --branch rp2040 --depth=1 --no-single-branch
$ cd openocd
$ ./bootstrap
$ ./configure --enable-picoprobe
$ make -j4
$ sudo make install
```

## OpenOCDの設定
cmsis-dap環境を更新してインストール
```
$ cd ~/pico/openocd
$ sed -i "/adapter driver cmsis-dap/a adapter speed 5000" ~/pico/openocd/tcl/interface/cmsis-dap.cfg
$ sudo make install
```

## usbipdのインストール
- 参考
  - USB デバイスを接続する | Microsoft Learn
- ネイティブ環境からWSL2にUSBデバイスをブリッジする
- まずWinsows側にUSBIPD-WINをインストール
  - PowerShellで実行
  ```
  > winget install --interactive --exact dorssel.usbipd-win
  ```

## usbアタッチ
- PicoにPicoprobeを接続、PicoprobeをPCに接続
- WindowsからWSLにPicoprobeをブリッジする
  - wslを立ちあげた状態で行う
  - Picoprobeのbusidを確認し、そのbusidをattachする
  ```
  > usbipd list
  > usbipd bind -b 2-2
  > usbipd attach --wsl --busid 2-1
  ```
- WSLのアップデートを求められた場合はそれに従う
  ```
  > wsl --update
  ```

  ```
  $ sudo apt install usbutils
  $ lsusb
  ```
  
## udevの設定
- usbデバイスの権限設定
  - usev.ruleの設定
  ```
  $ sudo cp ~/pico/openocd/contrib/60-openocd.rules /etc/udev/rules.d/99-openocd.rules
  $ sudo gpasswd -a  $(whoami) plugdev
  $ sudo udevadm control --reload && sudo udevadm trigger
  ```

## デバッグ
- OpenOCDを起動
もしかしたら/dev/memのアクセス権が必要かもしれません。
Ubuntu22.04ではrootで起動が必要でした。
権限付けたらOKでした  
(動いたらCtrl＋Cで停止)
  ```
  $ openocd -f interface/cmsis-dap.cfg -f target/rp2040.cfg
  ```

- デバッグ対象のディレクトリに移動
  ```
  $ cd ~/pico/workspace/pico_test 
  ```

- vscodeの設定ファイルを準備
  ```
  $ mkdir .vscode
  $ cp ~/pico/pico-examples/ide/vscode/launch-probe-swd.json .vscode/launch.json
  $ cp ~/pico/pico-examples/ide/vscode/settings.json .vscode/settings.json
  ```
  
- vscodeの起動
  ```
  $ code .
  ```
  「実行とデバッグ」(Ctrl＋Shift＋D)からデバッグの開始(F5)でデバッグを開始

## プロジェクトの作成
- pico-project-generatorのインストール
  ```
  $ cd ~/pico
  $ git clone https://github.com/raspberrypi/pico-project-generator.git --branch master
  $ sudo apt install python3-tk
  ```

- pico-project-generatorの起動
  ```
  $ cd ~/pico/workspace
  $ ../pico-project-generator/pico_project.py --gui
  ```
![alt text](project-1.png)

- minicomの設定
```
$ minicom -s
```
 Serial port setup を選んでenter
 Aを押してSerial Device　を変更
 Eを押してSerial Device　を変更
```
    +-----------------------------------------------------------------------+
    | A -    Serial Device      : /dev/ttyACM0                              |
    | B - Lockfile Location     : /var/lock                                 |
    | C -   Callin Program      :                                           |
    | D -  Callout Program      :                                           |
    | E -    Bps/Par/Bits       : 115200 8N1                                |
    | F - Hardware Flow Control : No                                        |
    | G - Software Flow Control : No                                        |
    | H -     RS485 Enable      : No                                        |
    | I -   RS485 Rts On Send   : No                                        |
    | J -  RS485 Rts After Send : No                                        |
    | K -  RS485 Rx During Tx   : No                                        |
    | L -  RS485 Terminate Bus  : No                                        |
    | M - RS485 Delay Rts Before: 0                                         |
    | N - RS485 Delay Rts After : 0                                         |
    |                                                                       |
    +-----------------------------------------------------------------------+
```

- minicomの起動
別窓で
```
$ minicom
```

デバッグを実行

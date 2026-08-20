# 交接包：CAM-C2-SW v1.2 — 完整版（Claude Web 判讀 ⇄ Claude Code 執行 橋接）

> **版本：2026-08-14 v1.2**
> **設計原則：本文件必須自足。接手的 Claude Code 不應該需要再問使用者任何問題就能開始執行 W03b/W03c。**
> **用途**：(1) 跨 Pro 帳號接續用；(2) 直接貼給 Claude Code（Windows App，本機有 bash + git 權限）執行；
>          (3) 同步進 GitHub repo `ilyrenchi/sync_CC_A-B-Cloud/sync_cam_2_sw_v1_1_cc_bridge/` 做雙邊橋接
> **前置文件**：CAM-C2 v3.4（硬體採購線）、CAM-C2-SW v1.0/v1.1（軟體線前兩版，本版整合並取代）

---

## §0 Claude Code 30 秒上手（先看這裡，不用再問）

```
專案：RK3568 LPR 停車場系統，新增 IMX415 遠距車牌相機（CAM-C2 主題）
你現在的角色：在編譯主機上，把 IMX415 sensor 的 DTS 節點寫進去、編譯驗證過
環境：這是「編譯主機」不是「板端」——目前手上沒有實體 IMX415 模組（樣品在海運途中）
      所以你能做：改 DTS、編 kernel/DTB、驗證編譯通過
      你不能做：燒錄、上機測試、dmesg/v4l2-ctl 實測（這些是 W04，要等樣品到貨）

編譯主機路徑：~/Downloads/MYD-LR3568   （Ubuntu 22.04）
目標 defconfig：rockchip_rk3568_myd_lr3568x_full_defconfig
                （對應出貨 image：myir-image-lr3568-debian，已用使用者提供的
                 image 檔名反推確認，不是 core 版）
目標 dts 檔案：kernel-6.1/arch/arm64/boot/dts/rockchip/myd-lr3568x-camera.dtsi
IQ 檔已就位：external/camera_engine_rkaiq/rkaiq/iqfiles/isp21/
             imx415_CMK-OT2022-PX1_IR0147-50IRC-8M-F20.json（已複製完成，不用再找）

直接執行 §5 的批次清單，DTS 內容直接用 §6 骨架，不用回頭問使用者「要用哪個 defconfig」
「dts 骨架長怎樣」這類問題——這份文件已經有答案。

真正還沒有答案、你也查不到的，只有 §7 列的兩項（CSI_PWDN/MCLK 實際 GPIO），
這兩項本來就要等樣品到貨或原理圖延伸頁才能確認，不是你现在該卡住的地方——
先用候選值把 DTS 寫好、能編譯過即可，候選值已經在 §6 骨架裡標好來源。
```

---

## §1 專案背景（完整脈絡，跨主題共用）

```
專案：RK3568 車牌辨識停車場系統
板型：MYIR MYD-LR3568X
核心軟體：rk3568_lpr_clean（板端運行，V4L2Capture → RGA → YOLOv8 → LPRNet → RTSP/HTTP/WS）
本主題定位：新增第二顆相機（CAM-C2），解決「3–5m 遠距離車牌辨識率不足」問題
           現有 CAM003（OV5640, 640×480）在 3–5m 距離車牌僅 32–54px，
           低於辨識所需的 ≥60px 門檻，物理上無法達標，因此換高解析度感測器
選定方案：IMX415（8.4MP，4K），取代原規劃的 IMX307/IMX335
         選擇理由：社群公認唯一有穩定公版 IQ 可用的主流 IMX 系列、
         Rockchip 官方 defconfig 已標配驅動、模組供應商選擇多（不綁定單一廠商）
本主題（CAM-C2-SW）範圍：純軟體準備（kernel config + IQ 檔 + dts），
                        趁硬體打樣期間同步推進，樣品到貨即可直接測試，不留空窗期
明確不處理：現有 OV5640 近距離畫質問題（那是獨立主題 CAM-C1，不要混進來）
```

### 1.1 板端硬體環境（已確認）

| 項目 | 值 |
|---|---|
| SoC | RK3568 |
| 板型 | MYIR MYD-LR3568X |
| Kernel | 6.1.99（MYIR BSP）|
| SDK manifest | `rk3566_rk3568_linux6.1_release_v1.1.0_20241220.xml` |
| ISP 硬體版本 | **isp21**（決定 IQ 檔世代，不可混用 isp20/isp30/isp39/isp3x）|
| RKAIQ runtime | v6.0x9.0（release 2024-12-13）|
| RKISP driver | v02.09.00 |
| 出貨 rootfs | Debian 12（image 檔名 `myir-image-lr3568-debian`）|
| MIPI CSI 接口（原理圖標籤）| J10（標示 "MIPI-CSI1"），24-pin，i2c4，4-lane，GPIO0_B0 電源致能 |
| MIPI CSI 接口（kernel dts 實際命名）| `csi2_dphy0`（**不是** `csi2_dphy1`——原理圖標籤與 dts 命名系統不同，已交叉驗證見 §4）|
| 現有 dts 既有 sensor 節點框架 | `ov5640@3c`、`ov5645@3c`、`ov13855@36`（掛在 `myd-lr3568x-camera.dtsi`）|

### 1.2 J10 完整 24-pin 定義（原理圖讀出，已確認）

| Pin | 信號 | 電壓 |
|:---:|---|:---:|
| 1 | GND | — |
| 2 | 3.3V | 3.3V |
| 3 | CSI1_PWR_IO1 ← GPIO0_B0 | 3.3V |
| 4 | I2C4_SDA_M0（R75 22Ω）| 1.8V |
| 5 | I2C4_SCL_M0（R76 22Ω）| 1.8V |
| 6 | CSI_PWDN（R77 22Ω）| 1.8V |
| 7 | MCLK | 1.8V |
| 8 | GND | — |
| 9–10 | MIPI_CSI1_D3_P/N | 1.8V |
| 11 | GND | — |
| 12–13 | MIPI_CSI1_D2_P/N | 1.8V |
| 14 | GND | — |
| 15–16 | MIPI_CSI1_CLK_P/N | 1.8V |
| 17 | GND | — |
| 18–19 | MIPI_CSI1_D1_P/N | 1.8V |
| 20 | GND | — |
| 21–22 | MIPI_CSI1_D0_P/N | 1.8V |
| 23–24 | GND | — |

### 1.3 感測器模組規格（已定案，硬體採購線 CAM-C2 v3.4）

| 項目 | 定案值 |
|---|---|
| Sensor | IMX415（Sony 原廠 datasheet 已核對）|
| 晶振 | 24MHz |
| I2C 地址 | 0x1a |
| 鏡頭 | 5m 定焦，6mm EFL，M12，實物調焦 |
| 連接器 | 0.5mm 間距、24-pin，與板端 J10 原生相符 |
| IR-CUT | 不配（本輪跳過）|
| 電壓（Sony datasheet）| AVDD 2.9V / OVDD 1.8V / DVDD 1.1V |
| 供應商 | 深圳創騰興光光電科技（線上暱稱：韋辰奶xin），實際生產：仁恒（石先生）|
| 採購狀態 | 已下單，¥1500/2顆，訂單號 3315446978853009651，2026-08-14 已出庫，順豐單號 SF5154194145918，經廈門集運倉海運至台灣桃園，預估到貨 8月中下旬 |

---

## §2 編譯主機實際環境（與原始假設有落差，已修正，直接採用）

**重要**：原始交接包（v1.0）假設的 SDK 目錄結構與這台實際編譯主機（Ubuntu 22.04，`~/Downloads/MYD-LR3568`）不完全一致，以下是**實測修正後**的正確路徑，之後任何操作直接用這份，不要用 v1.0 原始假設路徑：

```
SDK 根目錄：~/Downloads/MYD-LR3568
kernel 原始碼：kernel-6.1/
IQ 檔目錄（注意比原假設多一層 rkaiq/）：
  external/camera_engine_rkaiq/rkaiq/iqfiles/isp21/
  （原假設是 external/camera_engine_rkaiq/iqfiles/isp21/ ，是錯的，已用實測修正）
dts 目錄：kernel-6.1/arch/arm64/boot/dts/rockchip/
```

**這台編譯主機是「首次建置」**：跑 `./build.sh` 任何選項時出現過 `WARN: output/defconfig not exists`，代表這台機器的 SDK tree 從未被完整配置編譯過。這意味著：
- 板子上目前實際跑的韌體，不確定是不是這份 SDK tree 編出來的（可能原廠出貨燒錄、或別台機器編的）
- 第一次跑 `./build.sh` 時系統會跳出互動式選單要求選 defconfig 編號，**不是報錯，是正常流程**，直接選 `rockchip_rk3568_myd_lr3568x_full_defconfig`（對應下方指令會用非互動方式直接帶參數，不會卡住等你選）

**`build.sh` 可用指令（已用 `build.sh` 內建 usage 輸出核對過，不用再問）：**

```
defconfig[:<config>]         選 defconfig，非互動用法：./build.sh rockchip_rk3568_myd_lr3568x_full_defconfig
kernel-6.1[:dry-run]         編 kernel 6.1（含 DTB，DTB 沒有獨立指令，包在這裡面一起編）
kernel-config[:dry-run]      修改 kernel defconfig（menuconfig）
kernel-make[:<arg1>:<arg2>]  直接呼叫 kernel make，適合只重編 dtb：
                              ./build.sh kernel-make dtbs
all                           編整套 image（kernel+dtb+rootfs+u-boot 等全部）
clean-kernel                  清 kernel 編譯產物
```

---

## §3 W01/W02/W03 查證結果（Claude Web 已完成，不用重查，直接引用結論）

### W01 — Kernel Config：✅ 已確認，不需改動

```
kernel-6.1/arch/arm64/configs/myd_lr3568x_defconfig:377:CONFIG_VIDEO_IMX415=y
```
driver 原始碼確認存在：`kernel-6.1/drivers/media/i2c/imx415.c`（86801 bytes）
VCM 備用 driver 也在：`kernel-6.1/drivers/media/i2c/dw9714.c`（38424 bytes）
**結論：不需要 backport，不需要開 config，kernel 這關直接過。**

### W02 — IQ 檔：✅ 已找到並複製進 SDK，可直接驗證

- 目標檔名：`imx415_CMK-OT2022-PX1_IR0147-50IRC-8M-F20.json`
- 本地 SDK 原本只有 `isp3x`、`isp39` 世代的 imx415（**世代不符，isp39 是 RK3576 世代，isp3x 也非 isp21，兩者都不能用在本板，硬套會導致 3A 演算法 segfault**）
- **正確檔案來源**：`gitlab.com/firefly-linux/external/camera_engine_rkaiq`，commit `da2677af`
  （commit message："iqfiles: isp21: add imx415_CMK-OT2022-PX1_IR0147-50IRC-8M-F20.json"）
- **注意**：該 repo 若用 `--depth 1` 淺層 clone 只會抓到最新 HEAD（HEAD 版本沒有 isp21 目錄、且該版本所有 imx415 檔案都是 isp20 世代的 XML 格式，不能用），必須完整 clone 後 `git checkout da2677af` 才能找到目標檔案
- **已複製完成並驗證**：
  ```
  ~/Downloads/MYD-LR3568/external/camera_engine_rkaiq/rkaiq/iqfiles/isp21/imx415_CMK-OT2022-PX1_IR0147-50IRC-8M-F20.json
  ```
  （`ls -la` 驗證過，331137 bytes，2026-07-28 16:42 建立）
  **這步已完成，Claude Code 不用重跑 git clone，檔案已經在正確位置。**

### W03 — DTS 範本：✅ 找到且已交叉驗證，尚未寫入節點

- 板廠原生範本：`kernel-6.1/arch/arm64/boot/dts/rockchip/myd-lr3568x-camera.dtsi`
- 主板 `myd-lr3568x.dts` 用 `#include "myd-lr3568x-camera.dtsi"` 引用（已用 `grep` 確認是真正在編譯路徑中的檔案）
- **排除項**：`rk3568-evb1-dual-camera.dtsi` 這個檔案雖然也搜到、內容也有 camera 定義，但它用的是 `csi2_dphy1`/`csi2_dphy2` 的 split mode 架構，是 Rockchip 官方 EVB 通用範例，**不是這片板子實際在用的**，已排除
- 範本走 `csi2_dphy0` + `i2c4`，裡面已有 `ov13855`（4-lane，跟 J10 規格一致，其他兩顆 ov5645/ov5640 是 2-lane）、`ov5645`、`ov5640` 三顆可替換鏡頭節點
- **電源腳位交叉驗證**（決定性證據，證明範本選對了）：範本裡 `vcc_camera` regulator 掛 `gpio0 RK_PB0`（=GPIO0_B0），與 §1.2 原理圖 J10 pin3 `CSI1_PWR_IO1 ← GPIO0_B0` 完全吻合

---

## §4 dts 範本原始內容（完整貼出，供比對用，不用再去讀檔案）

```dts
#include <dt-bindings/gpio/gpio.h>
#include <dt-bindings/pinctrl/rockchip.h>
#include <dt-bindings/display/media-bus-format.h>

/ {
	vcc_camera: vcc-camera-regulator {
		compatible = "regulator-fixed";
		gpio = <&gpio0 RK_PB0 GPIO_ACTIVE_HIGH>;
		pinctrl-names = "default";
		pinctrl-0 = <&camera_pwr>;
		regulator-name = "vcc_camera";
		enable-active-high;
		regulator-always-on;
		regulator-boot-on;
	};

	clk_ov5640_fixed: clock {
		compatible = "fixed-clock";
		#clock-cells = <0>;
		clock-frequency = <24000000>;
		clock-output-names = "CLK_CAMERA_24MHZ";
	};

	clk_ov5645_fixed: clock {
		compatible = "fixed-clock";
		#clock-cells = <0>;
		clock-frequency = <24000000>;
		clock-output-names = "CLK_CAMERA_24MHZ";
	};

	clk_ov13855_fixed: clock {
		compatible = "fixed-clock";
		#clock-cells = <0>;
		clock-frequency = <24000000>;
		clock-output-names = "CLK_CAMERA_24MHZ";
	};
};

&csi2_dphy_hw {
	status = "okay";
};

&csi2_dphy0 {
	status = "okay";

	ports {
		#address-cells = <1>;
		#size-cells = <0>;
		port@0 {
			reg = <0>;
			#address-cells = <1>;
			#size-cells = <0>;

			mipi_in_ucam0: endpoint@1 {
				reg = <1>;
				remote-endpoint = <&ov13855_out>;
				data-lanes = <1 2 3 4>;
			};
			mipi_in_ucam1: endpoint@2 {
				reg = <2>;
				remote-endpoint = <&ov5645_out>;
				data-lanes = <1 2>;
			};
			mipi_in_ucam2: endpoint@3 {
				reg = <3>;
				remote-endpoint = <&ov5640_out>;
				data-lanes = <1 2>;
			};
		};
		port@1 {
			reg = <1>;
			#address-cells = <1>;
			#size-cells = <0>;

			csidphy_out: endpoint@0 {
				reg = <0>;
				remote-endpoint = <&isp0_in>;
			};
		};
	};
};

&i2c4 {
	status = "okay";

	ov13855: ov13855@36 {
		status = "okay";
		compatible = "ovti,ov13855";
		reg = <0x36>;
		clocks = <&clk_ov13855_fixed>;
		clock-names = "xvclk";
		clock-frequency = <24000000>;
		power-domains = <&power RK3568_PD_VI>;
		reset-gpios = <&gpio2 RK_PC2 GPIO_ACTIVE_HIGH>;
		pwdn-gpios = <&gpio4 RK_PC1 GPIO_ACTIVE_HIGH>;
		rockchip,camera-module-index = <0>;
		rockchip,camera-module-facing = "back";
		rockchip,camera-module-name = "NC";
		rockchip,camera-module-lens-name = "NC";

		port {
			ov13855_out: endpoint {
				remote-endpoint = <&mipi_in_ucam0>;
				data-lanes = <1 2 3 4>;
			};
		};
	};

	/* ov5645@3c、ov5640@3c 節點省略（2-lane，非本次修改對象，不動） */
};

&pinctrl {
	cam {
		camera_pwr: camera-pwr {
			rockchip,pins =
				<0 RK_PB0 RK_FUNC_GPIO &pcfg_pull_none>;
		};
	};
};

&rkisp { status = "okay"; };
&rkisp_mmu { status = "okay"; };
&rkisp_vir0 {
	status = "okay";
	port {
		#address-cells = <1>;
		#size-cells = <0>;
		isp0_in: endpoint@0 {
			reg = <0>;
			remote-endpoint = <&csidphy_out>;
		};
	};
};
```

---

## §5 CC 待辦批次清單

| 批次 | 任務 | 複雜度 | 模型 | 環境 | 驗證方式 | 狀態 |
|:---:|---|:---:|:---:|:---:|---|:---:|
| W03b | 依 §6 骨架修改 `myd-lr3568x-camera.dtsi`：新增 `clk_imx415_fixed`、新增 `imx415@1a` 節點、改 `csi2_dphy0` port@0 的 endpoint@1 指向 imx415 | 中 | Sonnet | CC | `git diff` 對照 §6 | ✅ 2026-08-20 |
| W03c | 選 defconfig：`./build.sh rockchip_rk3568_myd_lr3568x_full_defconfig`，接著 `./build.sh kernel-6.1` 驗證 DTS 語法過、DTB 編出來 | 中 | Sonnet | CC | 編譯無錯誤，`output` 目錄下有對應 `.dtb` | ✅ 2026-08-20 |
| W03d | 若 W03c 失敗，依報錯訊息除錯（常見：`/delete-node/` 順序錯、endpoint reg 衝突、label 重複） | 中～高 | Sonnet/Opus 視錯誤複雜度 | CC | 重編通過 | N/A（W03c 一次過）|
| W03e | 全套編譯 `./build.sh all`，產出完整 image，為板端 bring-up 預備（可選） | 低 | Haiku | CC | build 無錯誤結束 | ⏳ |
| W03f | 把本次修改 + 編譯結果同步進 GitHub repo（見 §8）| 低 | Haiku | CC | push 成功，repo 可見新檔案 | ⏳ |
| W04~W07 | 板端 bring-up（**等樣品到貨才開始，目前不要嘗試**）：燒錄、`dmesg`/`v4l2-ctl --list-devices`/`media-ctl -p` 驗證、CSI_PWDN/MCLK 實測確認 | 高 | Opus | 板端 | dmesg 無 error、v4l2 能列出設備 | ⏳ 等貨 |

**執行順序**：W03b → W03c →（成功）W03e（可選）→ W03f；（失敗）W03d 除錯後回 W03c。

---

## §6 dts 骨架（W03b 直接套用，逐段對應要改的位置）

修改目標檔案：`kernel-6.1/arch/arm64/boot/dts/rockchip/myd-lr3568x-camera.dtsi`

**改動 1 —— 在 `/ { ... }` 區塊內，仿照 `clk_ov13855_fixed` 新增：**

```dts
clk_imx415_fixed: clock-imx415 {
	compatible = "fixed-clock";
	#clock-cells = <0>;
	clock-frequency = <24000000>;
	clock-output-names = "CLK_CAMERA_24MHZ";
};
```

**改動 2 —— 在 `&i2c4 { ... }` 區塊內，新增 imx415 節點（與既有 ov13855 並存，不刪除舊節點）：**

```dts
imx415: imx415@1a {
	status = "okay";
	compatible = "sony,imx415";
	reg = <0x1a>;
	clocks = <&clk_imx415_fixed>;
	clock-names = "xvclk";
	clock-frequency = <24000000>;
	power-domains = <&power RK3568_PD_VI>;

	/* ⚠️ 候選值，抄自 ov13855 節點，未經原理圖/實測驗證，
	   等 §7 CSI_PWDN/MCLK 走線確認後可能需要修正 */
	reset-gpios = <&gpio2 RK_PC2 GPIO_ACTIVE_HIGH>;
	pwdn-gpios = <&gpio4 RK_PC1 GPIO_ACTIVE_HIGH>;

	rockchip,camera-module-index = <0>;
	rockchip,camera-module-facing = "back";
	rockchip,camera-module-name = "CMK-OT2022-PX1";
	rockchip,camera-module-lens-name = "IR0147-50IRC-8M-F20";

	port {
		imx415_out: endpoint {
			remote-endpoint = <&mipi_in_ucam0>;
			data-lanes = <1 2 3 4>;
		};
	};
};
```

**改動 3 —— 在 `&csi2_dphy0 { ports { port@0 { ... } } }` 內，把 `mipi_in_ucam0` 改接到 imx415（取代原本接 ov13855_out）：**

```dts
/delete-node/ endpoint@1;   /* 移除原本 mipi_in_ucam0 掛點（接 ov13855_out）*/

mipi_in_ucam0: endpoint@1 {
	reg = <1>;
	remote-endpoint = <&imx415_out>;
	data-lanes = <1 2 3 4>;
};
```

> **決定性提醒**：`rockchip,camera-module-name` + `rockchip,camera-module-lens-name` 兩個字串組合起來必須完全等於 §3 W02 已放進 SDK 的 IQ 檔名（去掉 `imx415_` 前綴和 `.json` 後綴）。目前寫的 `"CMK-OT2022-PX1"` + `"IR0147-50IRC-8M-F20"` 組合起來正好對應 `imx415_CMK-OT2022-PX1_IR0147-50IRC-8M-F20.json`，照抄不要手動改字，拼錯一個字 3A 演算法就載入不到、不會報錯但畫面會不動。

> **關於 ov13855 節點去留**：若板子上物理上不會同時裝 ov13855 和 imx415（同一個接口只能接一顆），W03d 除錯階段可能需要決定是保留 ov13855 節點但 status 改 "disabled"，還是直接刪除，這個判斷留給 W03c 編譯報錯時再處理，不用現在猜。

---

## §7 尚未解決項目（誠實列出，不是「查不到就算了」）

| 項目 | 現況 | 需要什麼才能解決 | 是否卡住 W03b/W03c |
|---|---|---|---|
| CSI_PWDN 實際 GPIO | 候選值 `gpio4 RK_PC1`（抄自 ov13855，未驗證專屬 J10）| 原理圖 CSI_PWDN 走線延伸頁，或樣品到貨後 `dmesg`/示波器實測 | 否，先用候選值編譯過即可 |
| MCLK pinmux | 範本裡 ov13855/ov5645/ov5640 都沒有獨立 MCLK pinctrl 定義，可能透過共用時脈路徑，具體 pinmux 名稱未知 | 同上 | 否，`clock-frequency` 已用固定時脈節點頂替，能編譯通過 |
| `vcc_camera` regulator 與 imx415 節點電源的關係 | 範本用獨立 regulator-fixed 統一致能 GPIO0_B0，imx415 節點本身沒寫 `enable-gpios`，不確定是否需要額外加 | 樣品到貨後實測開機是否已由 regulator 自動供電 | 否 |
| Firefly FAE 對 dts/IQ 的追問 | 已發問，未回覆（CAM-C2 硬體線 §10 記錄）| 等對方回覆，若回覆則優先採用其提供的數值取代 §6 候選值 | 否 |

**結論：以上四項都不阻擋 W03b/W03c 執行**，用候選值先把 DTS 寫好、編譯驗證過，這是本次 CC 該做的事。真正的最終確認要等 W04（樣品到貨後）才能做，這是計畫內的正常分工，不是卡關。

---

## §8 GitHub 橋接同步（Claude Code 執行，本機已有 git 認證）

目標 repo：`https://github.com/ilyrenchi/sync_CC_A-B-Cloud`
子資料夾：`sync_cam_2_sw_v1_1_cc_bridge/`

```bash
# 第一次使用，若本機還沒 clone：
cd ~/Downloads   # 或你偏好的位置，跟 MYD-LR3568 分開放即可
git clone https://github.com/ilyrenchi/sync_CC_A-B-Cloud.git
cd sync_CC_A-B-Cloud

# 已 clone 過，先同步最新狀態：
git pull origin main   # 分支名稱若是 master 請對應調整

mkdir -p sync_cam_2_sw_v1_1_cc_bridge

# 把這份交接包本身存進去（同步 Claude Web 產出的完整背景文件）：
cp [這份檔案的實際路徑，使用者下載後告知] \
   sync_cam_2_sw_v1_1_cc_bridge/HANDOFF_CAM-C2-SW_v1_2.md

# W03b/W03c 做完後，把改動與結果也存進去：
cd ~/Downloads/MYD-LR3568
git diff kernel-6.1/arch/arm64/boot/dts/rockchip/myd-lr3568x-camera.dtsi \
   > ~/Downloads/sync_CC_A-B-Cloud/sync_cam_2_sw_v1_1_cc_bridge/W03b_dts_diff.patch

cd ~/Downloads/sync_CC_A-B-Cloud
git add sync_cam_2_sw_v1_1_cc_bridge/
git commit -m "CAM-C2-SW v1.2: W03b dts 節點寫入 + W03c 編譯驗證結果"
git push origin main
```

**橋接運作方式**：
- Claude Web 產出的判讀/規劃文件 → 存進 `sync_cam_2_sw_v1_1_cc_bridge/`
- Claude Code 執行完的 diff / 編譯 log / 更新版 HANDOFF → 同樣存進同一資料夾
- 任何一邊要接續工作，先 `git pull` 這個 repo 拿到最新狀態，不用靠聊天記錄回溯
- **repo 若為 public**：建議把 §1.3 供應商聯絡細節、訂單號這類非技術資訊排除在 commit 之外，或改設 private repo；技術內容（DTS/kernel config/IQ 路徑）公開風險低

---

## §9 主題記錄

- **主題代號**：CAM-C2-SW
- **本版更新時間**：2026-08-20（W03b/W03c 完成）
- **git HEAD（kernel-6.1 submodule）**：`3d0580aa0d50`（branch `myd-lr3568-l601`，詳見 §10）
- **前置主題**：CAM-C2 v3.4（硬體採購，已定案，樣品已出貨）
- **下一步**：W03f push diff → W04 等樣品到貨後板端 bring-up

---

## §10 W03b/W03c 執行記錄（2026-08-20，勿踩同樣的坑）

### 10.1 W03b DTS 實際修改內容

修改檔案：`kernel-6.1/arch/arm64/boot/dts/rockchip/myd-lr3568x-camera.dtsi`

共四處變動（依照 §6 骨架執行，加上一個 §6 未預見的額外修正）：

**變動 1** — 在 `/ { }` 區塊新增 `clk_imx415_fixed` 時脈節點（仿照 `clk_ov13855_fixed`）

**變動 2** — 在 `&i2c4 { }` 新增 `imx415: imx415@1a` 完整節點（含 `imx415_out` endpoint）

**變動 3** — `&csi2_dphy0 port@0 mipi_in_ucam0` 的 `remote-endpoint` 從 `<&ov13855_out>` 改為 `<&imx415_out>`

**變動 4（§6 未列、執行時才發現必要）** — 把 `ov13855@36` 節點的 `status` 從 `"okay"` 改為 `"disabled"`

> **為什麼要加 變動 4？**
> `ov13855_out` 仍然有 `remote-endpoint = <&mipi_in_ucam0>`，
> 但 `mipi_in_ucam0` 已改接 `imx415_out`，
> DTC 會報 endpoint ref 不一致（或 kernel 啟動時 CSI subsystem 互搶 endpoint）。
> 把 ov13855 整個節點 disable 掉，讓 kernel 完全忽略它，避免衝突。
> **下一版 §6 要加這條，或說明「ov13855 endpoint 改為 self-pointing 也行，但 disable 最乾淨」。**

備份：`myd-lr3568x-camera.dtsi.bak2`（disable 前）存在同目錄。

---

### 10.2 W03c 編譯環境：必裝套件（完整清單）

這台 Ubuntu 22.04（Nobel，`~/Downloads/MYD-LR3568`）是**首次跑編譯**，幾乎從零裝。
完整指令（已測試通過，一次貼完執行，不用分批）：

```bash
sudo apt-get install -y \
  git make gcc g++ gcc-multilib g++-multilib \
  libssl-dev libelf-dev libncurses-dev \
  flex bison bc \
  lz4 liblz4-tool \
  device-tree-compiler \
  u-boot-tools \
  cpio rsync kmod \
  python3 python3-pip python-is-python3 \
  patchelf chrpath gawk texinfo \
  fakeroot cmake unzip expect \
  qemu-user-static binfmt-support \
  gcc-aarch64-linux-gnu binutils-aarch64-linux-gnu \
  sshpass
```

**check-kernel.sh 逐項通過條件：**

| check-kernel.sh 檢查項 | 套件 | 特別注意 |
|---|---|---|
| `python` 指令存在 | `python-is-python3` | 不是 `python3`，是 `python-is-python3`，少這個直接失敗 |
| `flex` | `flex` | 少了會在 kconfig/lexer.lex.c 報 Error 127 |
| `openssl/ssl.h` | `libssl-dev` | |
| `gmp.h` | `libgmp-dev`（gcc-multilib 附帶）| |
| `mpc.h` | `libmpc-dev`（gcc-multilib 附帶）| |
| `ncurses.h` | `libncurses-dev` | 單獨安裝不夠，要明確指定 libncurses-dev |
| `lz4 favor-decSpeed` | `lz4` | version ≥ 1.9.3 才有這個旗標，apt 預設版本足夠 |

**SDK 自帶 cross-compiler**（不需要手動裝 aarch64-linux-gnu-gcc）：
```
~/Downloads/MYD-LR3568/prebuilts/gcc/linux-x86/aarch64/
  gcc-arm-10.3-2021.07-x86_64-aarch64-none-linux-gnu/bin/
  aarch64-none-linux-gnu-gcc
```
build.sh 自動抓這個，不用手動 export CROSS_COMPILE。

---

### 10.3 W03c 編譯指令（已驗證成功）

```bash
cd /home/nobel/Downloads/MYD-LR3568

# 步驟 1：選 defconfig（只需做一次，之後重編不用重跑）
./build.sh rockchip_rk3568_myd_lr3568x_full_defconfig

# 步驟 2：編 kernel + DTB
./build.sh kernel-6.1 2>&1 | tee /tmp/kernel_build_full.log
echo "BUILD EXIT CODE: ${PIPESTATUS[0]}"
```

**成功判斷標準**：
- 輸出末尾出現 `Running 10-kernel.sh - build_kernel-6.1 succeeded.`
- `BUILD EXIT CODE: 0`
- DTB 確認：`ls -lh output/kernel/arch/arm64/boot/dts/rockchip/myd-lr3568x.dtb`
  （也可在 `output/firmware/boot.img` 裡，FIT image 已打包進去）

**本次成功輸出片段（供比對）：**
```
Image:  resource.img (with myd-lr3568x.dtb logo.bmp logo_kernel.bmp) is ready
Image:  boot.img (with Image  resource.img) is ready
Image:  zboot.img (with Image.lz4  resource.img) is ready
...
FIT Image 0 (fdt): myd-lr3568x.dtb  188.08 KiB
FIT Image 1 (kernel): 45.41 MiB
Running 10-kernel.sh - build_kernel-6.1 succeeded.
BUILD EXIT CODE: 0
```

**估時**：4 核機器首次冷編譯約 30–50 分鐘；有 `.o` 快取後重編約 5–15 分鐘。

---

### 10.4 已知無害警告（不用理會，但每次都會出現）

**警告 1**：
```
grep: /tmp/tmp.MJqx7GI6Em/tmp.pRVp4KZQDv: exceeded PCRE's backtracking limit
```
（連續出現 7–8 次）
原因：SDK 的 `mk-fitimage.sh` 用 grep 掃大型 .ko 內容，超過 PCRE 限制。不影響 build 結果，直接忽略。

**警告 2**：
```
 PLEASE CHECK BOARD GPIO POWER DOMAIN CONFIGURATION !!!!!
 <<< ESPECIALLY Wi-Fi/Flash/Ethernet IO power domain >>> !!!!!
```
原因：MYIR SDK 固定印出的提醒文字，每次 kernel build 都會出現。不是錯誤。

---

### 10.5 踩坑記錄（下次的人要注意）

**坑 1：tee 權限問題**

問題：`tee: /tmp/kernel_build_full.log: 拒絕不符權限的操作`
原因：之前以 root 建立過 `/tmp/kernel_build_full.log`，但 build.sh 內部以 nobel（uid 1000）身份跑 make，tee 也以同樣身份執行，寫不進 root 建的檔案。

修正：`rm -f /tmp/kernel_build_full.log` 再重跑，或整個 session 都用 nobel（不切到 root）執行。

**坑 2：build.sh 的 -j 參數不能直接傳**

問題：`./build.sh kernel-6.1 -j2` 沒有用，build.sh 忽略此參數，自行用 `$(nproc)` 決定。
（本機 4 核 → nproc=4，build.sh 實際跑 -j5，如果機器不穩定可能 OOM 或 crash）

正確限核方式（用 taskset 欺騙 nproc）：
```bash
taskset -c 0,1 ./build.sh kernel-6.1   # 只讓 build 看到 CPU 0 和 1，等同 -j2
```

**坑 3：`git add kernel-6.1/arch/...` 在 SDK 根目錄會失敗**

問題：`fatal: 路徑規格 '...' 在子模組 'kernel-6.1' 中`
原因：`kernel-6.1/` 是 SDK 的 git submodule，父 repo 不能直接 `git add` 子模組內的檔案。

正確做法：進子模組目錄 commit：
```bash
cd /home/nobel/Downloads/MYD-LR3568/kernel-6.1
git config user.email "renchi.teng@gmail.com"
git config user.name "renchi"
git add arch/arm64/boot/dts/rockchip/myd-lr3568x-camera.dtsi
git status        # 確認只有這一個檔案 staged
git commit -m "W03b: add IMX415 sensor to DTS, disable ov13855 to avoid endpoint conflict"
git log --oneline -3   # 確認 commit 存在
```

**✅ 2026-08-20 已執行結果：**
```
[myd-lr3568-l601 3d0580aa0d50] W03b: add IMX415 sensor to DTS, disable ov13855 to avoid endpoint conflict
 1 file changed, 35 insertions(+), 2 deletions(-)
3d0580aa0d50 (HEAD -> myd-lr3568-l601) W03b: add IMX415 sensor to DTS, disable ov13855 to avoid endpoint conflict
f9f556d92842 FIX: update new mipi101c configuration
b61f6161bafe FIX: modify myd-lr3568 compatible name
```
kernel-6.1 的工作分支為 `myd-lr3568-l601`（MYIR BSP 原始分支，不要切換）。

然後回到 SDK 父 repo 更新 submodule 指針（選做，若有需要推整套 SDK）：
```bash
cd /home/nobel/Downloads/MYD-LR3568
git add kernel-6.1   # 這樣才合法：父 repo 只記 submodule commit hash，不記子模組檔案內容
git commit -m "update kernel-6.1 submodule: add IMX415 DTS"
```

---

*交接包版本 v1.2 | 2026-08-20 W03b/W03c 完成更新*
*橋接：Claude Web（查證/判讀）⇄ GitHub sync_CC_A-B-Cloud ⇄ Claude Code（DTS 修改 + 編譯 + push）*
*模型建議：W03b/W03c/W03d → Sonnet（複雜除錯視情況升 Opus）；W03e/W03f 機械執行 → Haiku；W04 板端 bring-up → Opus*

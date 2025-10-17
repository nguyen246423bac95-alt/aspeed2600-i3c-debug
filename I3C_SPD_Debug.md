# ASPEED AST2600 I3C SPD 调试指南

本文档面向在 ASPEED AST2600 平台上通过 I3C 接口读取 DIMM SPD 数据时遇到 "未发现设备"、"总线初始化失败" 等问题的调试场景。以下步骤基于实际 Linux 系统日志：

```
root@sakway-pkg:~# ls -l /sys/bus/i
 i2c/  i3c/  iio/
root@sakway-pkg:~# ls -l /sys/bus/i3c/devices/
total 0
root@sakway-pkg:~#
root@sakway-pkg:~# ls -l /sys/bus/platform/devices/*i3c*
lrwxrwxrwx 1 root root 0 Apr  4 04:59 /sys/bus/platform/devices/1e7a0000.i3cg -> ../../../devices/platform/ahb/ahb:apb/ahb:apb:bus@1e7a0000/1e7a0000.i3cg
lrwxrwxrwx 1 root root 0 Apr  4 04:59 /sys/bus/platform/devices/1e7a2000.i3c0 -> ../../../devices/platform/ahb/ahb:apb/ahb:apb:bus@1e7a0000/1e7a2000.i3c0
lrwxrwxrwx 1 root root 0 Apr  4 04:59 /sys/bus/platform/devices/1e7a3000.i3c1 -> ../../../devices/platform/ahb/ahb:apb/ahb:apb:bus@1e7a0000/1e7a3000.i3c1
lrwxrwxrwx 1 root root 0 Apr  4 04:59 /sys/bus/platform/devices/1e7a4000.i3c2 -> ../../../devices/platform/ahb/ahb:apb/ahb:apb:bus@1e7a0000/1e7a4000.i3c2
lrwxrwxrwx 1 root root 0 Apr  4 04:59 /sys/bus/platform/devices/1e7a5000.i3c3 -> ../../../devices/platform/ahb/ahb:apb/ahb:apb:bus@1e7a0000/1e7a5000.i3c3
root@sakway-pkg:~#
root@sakway-pkg:~# dmesg |grep -i i3c
[    2.449515] dw-i3c-master 1e7a2000.i3c0: HW DAT supports only 8 target devices - enabling SW DAT to support 32 devices
[    2.461856] dw-i3c-master: probe of 1e7a2000.i3c0 failed with error -75
[    2.479671] dw-i3c-master 1e7a3000.i3c1: HW DAT supports only 8 target devices - enabling SW DAT to support 32 devices
[    2.491810] dw-i3c-master: probe of 1e7a3000.i3c1 failed with error -75
[    2.509514] dw-i3c-master 1e7a4000.i3c2: HW DAT supports only 8 target devices - enabling SW DAT to support 32 devices
[    2.521673] dw-i3c-master: probe of 1e7a4000.i3c2 failed with error -75
[    2.539367] dw-i3c-master 1e7a5000.i3c3: HW DAT supports only 8 target devices - enabling SW DAT to support 32 devices
[    2.551474] dw-i3c-master: probe of 1e7a5000.i3c3 failed with error -75
root@sakway-pkg:~#
```

另一个环境中 `journalctl` 提供的内核版本信息如下，可用于确认 AST2600 EVB 平台及 6.1.61 内核：

```
Apr 04 05:01:01 huakun-bhs kernel: Booting Linux on physical CPU 0xf00
Apr 04 05:01:01 huakun-bhs kernel: Linux version 6.1.61-46d54a6-dirty-c69d472 (oe-user@oe-host) (arm-openbmc-linux-gnueabi-gcc (GCC) 13.1.1 20230520, GNU ld (GNU Binutils) 2.40.0.20230419) #1 SMP Fri Jul  5 04:16:56 UTC 2024
Apr 04 05:01:01 huakun-bhs kernel: CPU: ARMv7 Processor [410fc075] revision 5 (ARMv7), cr=10c5387d
Apr 04 05:01:01 huakun-bhs kernel: OF: fdt: Machine model: AST2600 EVB
Apr 04 05:01:01 huakun-bhs kernel: Kernel command line: console=ttyS4,115200n8 root=/dev/ram rw
...
```

## 1. 基本确认

1. **确认内核版本与驱动补丁**：
   - AST2600 的 I3C 控制器使用 Synopsys DesignWare I3C IP（`dw-i3c-master`）。通过 `uname -a` 或 `journalctl -k | head` 确认当前内核版本（例如 6.1.61-46d54a6），并确保集成了最新的 AST2600 补丁，尤其是 SPD 相关驱动（`i3c-dimm`、`i3c-hub` 等）。
   - 检查 `CONFIG_I3C`、`CONFIG_I3CDEV`、`CONFIG_SENSORS_I3C_HUB`、`CONFIG_EEPROM_SPD`、`CONFIG_I3C_SLAVE_EEPROM` 等选项是否启用，必要时查看 `/proc/config.gz` 或构建输出中的 `.config`。

2. **确认设备树配置**：
   - 在 `arch/arm/boot/dts/aspeed/` 对应的 BMC 设备树中，确保 I3C 控制器节点（`1e7a2000`, `1e7a3000`, `1e7a4000`, `1e7a5000`）被启用 (`status = "okay"`)。
   - DIMM SPD 一般通过 I3C Hub 或直接接入 I3C 控制器，需要在设备树中描述 hub、端口、和 SPD 从设备的静态地址。参考 `i3c-hub`, `i3c-bus` 节点配置文档。

3. **确认硬件连线与电源**：
   - 确保 DIMM 板卡已上电，SPD EEPROM（I3C target）具有 1.2V 电源。
   - 检查 I3C_SCL/I3C_SDA 线路阻值是否匹配规范（典型 49.9 Ω 串联电阻 + 4.7 kΩ 上拉）。
   - SPD 支持 I3C 需要 DIMM 版本满足 DDR5 SPD5118 或 DDR4 SPD5116 规范；若是旧版 SPD 只支持 I2C，需要考虑退回 I2C 模式读取（见章节「退回 I2C 模式读取流程」）。

## 2. 驱动探测失败 (-75) 排查

`dw-i3c-master: probe ... failed with error -75` 对应 `-EOVERFLOW`，一般表示控制器在初始化时收到意外的总线响应。排查步骤如下：

1. **确认 I3C 主机的时钟和复位**：
   - 查看 `clk summary`：`cat /sys/kernel/debug/clk/clk_summary | grep i3c`，确认 `i3c` 控制器时钟已开启。
   - 检查 `reset` 控制器：`devmem` 或 `debugfs` 确认 `SCU` 中 I3C 模块未处于复位状态。

2. **检查 I3C DAT (Dynamic Address Table)**：
   - 错误日志中提到 HW DAT 仅支持 8 个 target，驱动自动切换为 SW DAT。确认内核中 `CONFIG_I3C_SLAVE_MQUEUE` 或相关 SW DAT 支持开启。
   - 若系统有多个 I3C 设备，尝试减少总线从设备数量以确认是否为 DAT 配置问题。

3. **验证 I3C 线路是否空闲**：
   - 使用示波器查看 SCL/SDA 是否处于高电平、是否有噪声。
   - 复位 DIMM 电源或通过 GPIO 控制 I3C Hub 的复位脚。

4. **重新加载驱动**：
   - `echo 1e7a2000.i3c0 > /sys/bus/platform/drivers/dw-i3c-master/unbind`
   - `echo 1e7a2000.i3c0 > /sys/bus/platform/drivers/dw-i3c-master/bind`
   - 观察 `dmesg` 是否仍然报错。

## 3. 进一步调试步骤

1. **启用更详细的内核日志**：
   - 在启动参数中加入 `dynamic_debug.verbose=1 dyndbg="file drivers/i3c/* +p"`，或使用 `echo 'file drivers/i3c/* +p' > /sys/kernel/debug/dynamic_debug/control` 动态开启调试日志。
   - 重新探测后查看 `dmesg`，获取初始化过程中更多细节。

2. **使用 `i3c` 用户态工具**：
   - 确认 `i3c-tools` 已编译安装，使用 `i3c detect -b 0`（总线号视情况而定）检测总线设备。
   - 若 `i3c-tools` 不可用，可参考 `i3cdump`, `i3cgetcfgr` 等命令验证总线状态。

3. **切换为 I2C 兼容模式**：
   - 若 SPD 芯片不支持 I3C，可通过在设备树中将相应 DIMM 节点切换至 I2C 控制器，并使用 `i2cdump -y <bus> 0x50` 验证。
   - 确保 I3C/I2C 复用引脚配置正确（Pinmux）。

4. **核对 Bootloader 初始化**：
   - U-Boot 可能已经对 I3C 控制器做初始化。确认 U-Boot 中是否启用了 I3C 驱动，如果存在冲突可尝试在 U-Boot 禁用相关功能或在内核启动前复位控制器。

5. **固件与 SPD 通信协议**：
   - 对于 DDR5 SPD，可能需要在 hub 上配置 VR/PMIC 等器件后 SPD 才能响应。检查 DIMM 参考设计是否要求额外的初始化步骤。
   - 如果平台使用 PIM（Power Management IC）进行仲裁，需要同步调试 PIM 的 I3C 通信。

## 4. 常见问题与解决方案

| 现象 | 可能原因 | 调试建议 |
| ---- | -------- | -------- |
| `/sys/bus/i3c/devices/` 为空 | 控制器 probe 失败、设备树未启用、线路问题 | 依次确认 dmesg、设备树、硬件连接 |
| `probe failed with error -75` | 总线握手异常、DAT 配置不足 | 开启 debug log，检查 DAT 配置、确认目标设备数量 |
| `i3c detect` 无设备 | SPD 不支持 I3C 或未供电 | 切换 I2C 模式验证、检查电源 | 
| SPD 数据读取错误 | DIMM SPD 需要 hub/PMIC 初始化 | 确认初始化顺序、驱动补丁 |

## 5. 参考资料

- ASPEED AST2600 数据手册 - I3C 控制器章节
- Linux Kernel Documentation: `Documentation/i3c/`、`Documentation/memory/spd.rst`
- JEDEC SPD5118 (DDR5) / SPD5116 (DDR4) 规范
- `i3c-tools` 项目：https://github.com/robherring/i3c-tools

## 6. 建议的调试流程

1. **内核配置确认** → 确认所有 I3C 相关驱动和选项已启用。
2. **设备树检查** → 验证 I3C 控制器、Hub、SPD 节点配置。
3. **硬件检查** → 测量 I3C 线路、电源、复位、DIMM 在位情况。
4. **驱动调试** → 通过动态调试日志、重新绑定驱动、使用用户态工具进行验证。
5. **兼容模式回退** → 必要时切换至 I2C 模式以确认 SPD 芯片是否工作正常。
6. **记录日志** → 收集 `dmesg`、`i3c detect`、示波波形等信息，作为进一步分析依据。

遵循上述步骤可以系统化定位 AST2600 上 I3C SPD 无法读取的原因，并逐步缩小问题范围。

## 7. 退回 I2C 模式读取流程

当 DIMM 所搭载的 SPD 芯片未满足 JEDEC DDR5 SPD5118 / DDR4 SPD5116 对 I3C 的支持要求，或实际运行中 I3C 握手始终失败时，可按以下步骤退回至 I2C 模式核查：

1. **确认 SPD 版本与供电**
   - 使用整机物料清单或 DIMM 标签确认 SPD 芯片型号及固件版本，查阅其是否仅支持 I2C。
   - 确认 SPD 侧 3.3V/2.5V 供电良好（旧版 SPD 多使用 2.5V/3.3V 电源），避免 I3C 1.2V 逻辑导致兼容性问题。

2. **调整设备树为 I2C 连接**
   - 在 BMC 设备树中，将原本指向 `i3c@1e7a2xxx` 的 DIMM 节点改为对应的 `i2c@1e78xxxx` 控制器节点，并设置 `compatible = "atmel,24c02"` 等 SPD EEPROM 兼容字符串。
   - 为 SPD 指定固定地址（一般为 `0x50/0x51`），并移除 I3C 专用属性（如 `i3c-hub`、`ibi`、`reset-gpios` 等），确保加载为标准 I2C EEPROM。
   - 若同时存在 I3C Hub，请在设备树中禁用相关 hub 节点，避免驱动抢占引脚。

3. **修改 Pinmux 与时钟**
   - 检查 SCU Pinmux 设置，确保 DIMM SPD 线路切换到 I2C 功能（`I2CxxSCL/I2CxxSDA`），必要时在 U-Boot 或内核 pinctrl 中更新。
   - 在 SCU 时钟控制寄存器中开启相应 I2C 控制器的时钟，确认未被关闭。

4. **重新编译并部署设备树/固件**
   - 重新构建 OpenBMC 镜像或手动更新 `.dtb`，重启 BMC，使更改生效。
   - 启动后通过 `dmesg | grep i2c` 确认 I2C 控制器和 SPD EEPROM 成功注册，例如 `at24 0-0050`。

5. **使用 I2C 工具读取 SPD**
   - 安装 `i2c-tools` 后，执行 `i2cdetect -y <bus>`，确认 `0x50/0x51` 地址存在。
   - 使用 `i2cdump -y <bus> 0x50` 或 `i2cget` 读取 SPD 数据，验证字节内容是否合理（如前两字节为存储容量、SPD 版本号等）。
   - 若读取成功，可将数据保存为二进制并使用 JEDEC SPD 解码工具进一步分析。

6. **评估双模策略**
   - 若确认 SPD 仅支持 I2C，可在量产固件中维持 I2C 模式；若后续升级至支持 I3C 的 SPD，可再按章节 2/3 的流程启用 I3C。
   - 若平台需要兼容两种 SPD，建议在引导流程中增加检测逻辑（例如先尝试 I3C，失败后自动切换至 I2C）。

通过上述流程，可在保持硬件连线不变的前提下验证 SPD 在 I2C 模式下的可用性，帮助判断问题源自 SPD 规格还是 I3C 控制器/驱动实现。

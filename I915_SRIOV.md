# Intel i915 SR-IOV 配置指南

通过 i915 SR-IOV 将 Intel 核显虚拟化为多个 VF，分配给虚拟机/容器使用。

## 关键资源

1. **支持列表**（确认 CPU 核显是否支持 SR-IOV）：
   <https://www.intel.cn/content/www/cn/zh/support/articles/000093216/graphics/processor-graphics.html>
2. **Host 驱动**（DKMS）：<https://github.com/strongtz/i915-sriov-dkms>
3. **Guest 驱动**：TODO（Windows Guest 需要特定版本驱动）
4. **EFI 固件**（部分主板需开启相关选项，可参考 system76 开源固件）：
   <https://github.com/system76/firmware-open>
   ***（用 sha256sum 检索以找到与设备匹配的版本）***

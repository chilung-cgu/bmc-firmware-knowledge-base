# OEM 擴充開發指南

本文件說明如何為 PLDM 新增 OEM 廠商擴充。

---

## 步驟概覽

1. 建立 OEM 目錄結構
2. 實作 OEM Handler
3. 新增 Meson 選項
4. 更新建置系統

---

## 1. 建立目錄結構

```bash
mkdir -p oem/<vendor>/libpldmresponder
mkdir -p oem/<vendor>/configurations/bios
mkdir -p oem/<vendor>/pldmtool
```

---

## 2. 實作 OEM Handler

```cpp
// oem/<vendor>/libpldmresponder/oem_handler.hpp
namespace pldm::responder::oem_<vendor> {

class Handler : public oem_platform::Handler {
public:
    int setOemEffecter(
        uint16_t effecterId,
        uint8_t count,
        const std::vector<set_effecter_state_field>& stateField,
        uint16_t effecterIntf) override;
        
    void buildOemPDR(pdr_utils::Repo& repo) override;
};

} // namespace
```

---

## 3. 新增 Meson 選項

```meson
# meson.options
option('oem-<vendor>', 
       type: 'feature', 
       value: 'disabled',
       description: 'Enable OEM <vendor> support')
```

---

## 4. 更新建置系統

```meson
# meson.build
if get_option('oem-<vendor>').allowed()
    subdir('oem/<vendor>')
endif
```

---

## 建置

```bash
meson setup build -Doem-<vendor>=enabled
meson compile -C build
```

---

## 相關文件

- [TypeOEM](TypeOEM.md) - OEM Type 說明
- [CodeOrganization](CodeOrganization.md) - 程式碼組織

---

*返回 [Home](Home.md)*

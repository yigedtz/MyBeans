# 拼豆管理系统（BeadManager）产品需求文档

> 版本：v1.3
> 目标平台：H5（适配 PC + 移动端）
> 交付对象：前端开发（React/Vue 技术栈）

---

## 1. 项目概述

### 1.1 产品背景
拼豆爱好者需要管理大量不同颜色的拼豆库存（Mard 品牌，共 221 色）。现有管理方式多为 Excel 或纸质记录，效率低下，且缺乏图纸关联、智能识图、库存预警、图纸制作等能力。本产品旨在提供一个轻量化的 H5 库存管理工具，方便随时随地管理拼豆。

### 1.2 产品目标
- 实现 221 色 Mard 拼豆的数字化库存管理
- 支持手动 + 智能识图两种库存变更方式
- 提供库存预警，避免拼豆断货
- 关联存储拼豆图纸，便于追溯
- 支持上传图片自动转换为拼豆像素图纸

### 1.3 目标用户
拼豆爱好者（个人用户），单用户场景，无需登录/多账号体系（可预留扩展）。

### 1.4 技术栈建议
- **前端**：React 18 + Vite + TailwindCSS（或 Vue 3 + Vite + UnoCSS）
- **状态管理**：Zustand / Pinia
- **数据持久化**：IndexedDB（dexie.js）
- **图像识别**：Tesseract.js（浏览器端 OCR，无需后端）
- **图像处理**：Canvas API（像素化、颜色匹配）
- **PWA**：支持离线使用、可添加到主屏幕

---

## 2. 数据模型

### 2.1 拼豆基础数据（BeadColor）
```typescript
interface BeadColor {
  id: string;           // 唯一标识，如 "A1"
  code: string;         // Mard 品牌编号，如 "A1"
  name: string;         // 颜色名称（英文）
  hex: string;          // 颜色色值，如 "#FAF4C8"
  letterGroup: string;  // 字母分组，如 "A"
}
```

### 2.2 库存记录（BeadInventory）
```typescript
interface BeadInventory {
  beadId: string;       // 关联 BeadColor.id
  currentQty: number;   // 当前库存量（单位：粒）
  threshold: number;    // 告警阈值，默认 50
  updatedAt: string;    // ISO 8601 时间戳
}
```

### 2.3 变更记录（BeadChangeLog）
```typescript
interface BeadChangeLog {
  id: string;           // 唯一标识
  beadId: string;       // 关联 BeadColor.id
  type: 'in' | 'out';   // 入库 / 出库
  qty: number;          // 变更数量（正数）
  reason: string;       // 变更原因：'manual' | 'image_recognition' | 'batch'
  note: string;         // 备注，识图时存储图纸名称/ID
  snapshot: {           // 变更后的快照
    qtyAfter: number;
    timestamp: string;
  };
  createdAt: string;    // ISO 8601 时间戳
}
```

### 2.4 拼豆图纸（BeadPattern）
```typescript
interface BeadPattern {
  id: string;           // 唯一标识
  name: string;         // 图纸名称
  images: string[];     // 图片 Base64 或 Blob URL
  createdAt: string;    // ISO 8601 时间戳
  relatedLogIds: string[]; // 关联的变更记录 ID（识图出库时自动关联）
}
```

### 2.5 像素图纸（PixelPattern）
```typescript
interface PixelPattern {
  id: string;           // 唯一标识
  name: string;         // 图纸名称
  sourceImage: string;  // 原图 Base64
  width: number;        // 图纸宽度（格数）
  height: number;       // 图纸高度（格数）
  gridData: string[][]; // 二维数组，每个格子的编号，如 gridData[y][x] = "A1"
  beadCounts: Record<string, number>; // 每种颜色的粒数统计
  createdAt: string;    // ISO 8601 时间戳
}
```

---

## 3. Mard 221 色基础数据

### 3.1 编号体系（9 个色系）
| 色系 | 编号范围 | 数量 |
|------|---------|------|
| A系·黄橙系 | A1 ~ A26 | 26色 |
| B系·绿色系 | B1 ~ B32 | 32色 |
| C系·蓝青系 | C1 ~ C29 | 29色 |
| D系·蓝紫系 | D1 ~ D26 | 26色 |
| E系·粉玫系 | E1 ~ E24 | 24色 |
| F系·红色系 | F1 ~ F25 | 25色 |
| G系·棕肤系 | G1 ~ G21 | 21色 |
| H系·黑白系 | H1 ~ H23 | 23色 |
| M系·大地系 | M1 ~ M15 | 15色 |
| **合计** | | **221色** |

### 3.2 编号格式
- 格式：`字母 + 数字`（无前导零），如 `A1`, `A26`, `B32`, `M15`
- 字母分组：`A`, `B`, `C`, `D`, `E`, `F`, `G`, `H`, `M`

### 3.3 完整色值表

```json
[
  {"id":"A1","code":"A1","name":"","hex":"#FAF4C8","letterGroup":"A"},
  {"id":"A2","code":"A2","name":"","hex":"#FFFFD5","letterGroup":"A"},
  {"id":"A3","code":"A3","name":"","hex":"#FEFF8B","letterGroup":"A"},
  {"id":"A4","code":"A4","name":"","hex":"#FBED56","letterGroup":"A"},
  {"id":"A5","code":"A5","name":"","hex":"#F4D738","letterGroup":"A"},
  {"id":"A6","code":"A6","name":"","hex":"#FEAC4C","letterGroup":"A"},
  {"id":"A7","code":"A7","name":"","hex":"#FE8B4C","letterGroup":"A"},
  {"id":"A8","code":"A8","name":"","hex":"#FFDA45","letterGroup":"A"},
  {"id":"A9","code":"A9","name":"","hex":"#FF995B","letterGroup":"A"},
  {"id":"A10","code":"A10","name":"","hex":"#F77C31","letterGroup":"A"},
  {"id":"A11","code":"A11","name":"","hex":"#FFDD99","letterGroup":"A"},
  {"id":"A12","code":"A12","name":"","hex":"#FE9F72","letterGroup":"A"},
  {"id":"A13","code":"A13","name":"","hex":"#FFC365","letterGroup":"A"},
  {"id":"A14","code":"A14","name":"","hex":"#FD543D","letterGroup":"A"},
  {"id":"A15","code":"A15","name":"","hex":"#FFF365","letterGroup":"A"},
  {"id":"A16","code":"A16","name":"","hex":"#FFFF9F","letterGroup":"A"},
  {"id":"A17","code":"A17","name":"","hex":"#FFE36E","letterGroup":"A"},
  {"id":"A18","code":"A18","name":"","hex":"#FEBE7D","letterGroup":"A"},
  {"id":"A19","code":"A19","name":"","hex":"#FDC72F","letterGroup":"A"},
  {"id":"A20","code":"A20","name":"","hex":"#FFD568","letterGroup":"A"},
  {"id":"A21","code":"A21","name":"","hex":"#FFE395","letterGroup":"A"},
  {"id":"A22","code":"A22","name":"","hex":"#F4E57D","letterGroup":"A"},
  {"id":"A23","code":"A23","name":"","hex":"#E6C9B7","letterGroup":"A"},
  {"id":"A24","code":"A24","name":"","hex":"#F7BA22","letterGroup":"A"},
  {"id":"A25","code":"A25","name":"","hex":"#FFD6BD","letterGroup":"A"},
  {"id":"A26","code":"A26","name":"","hex":"#FFC830","letterGroup":"A"},

  {"id":"B1","code":"B1","name":"","hex":"#E6EE31","letterGroup":"B"},
  {"id":"B2","code":"B2","name":"","hex":"#63F347","letterGroup":"B"},
  {"id":"B3","code":"B3","name":"","hex":"#9EF780","letterGroup":"B"},
  {"id":"B4","code":"B4","name":"","hex":"#5DE035","letterGroup":"B"},
  {"id":"B5","code":"B5","name":"","hex":"#35E352","letterGroup":"B"},
  {"id":"B6","code":"B6","name":"","hex":"#65E2A6","letterGroup":"B"},
  {"id":"B7","code":"B7","name":"","hex":"#3DAF80","letterGroup":"B"},
  {"id":"B8","code":"B8","name":"","hex":"#1C9C4F","letterGroup":"B"},
  {"id":"B9","code":"B9","name":"","hex":"#27523A","letterGroup":"B"},
  {"id":"B10","code":"B10","name":"","hex":"#95D3C2","letterGroup":"B"},
  {"id":"B11","code":"B11","name":"","hex":"#5D722A","letterGroup":"B"},
  {"id":"B12","code":"B12","name":"","hex":"#166F41","letterGroup":"B"},
  {"id":"B13","code":"B13","name":"","hex":"#CAE87B","letterGroup":"B"},
  {"id":"B14","code":"B14","name":"","hex":"#ADE946","letterGroup":"B"},
  {"id":"B15","code":"B15","name":"","hex":"#2E5132","letterGroup":"B"},
  {"id":"B16","code":"B16","name":"","hex":"#C5ED9C","letterGroup":"B"},
  {"id":"B17","code":"B17","name":"","hex":"#9BB13A","letterGroup":"B"},
  {"id":"B18","code":"B18","name":"","hex":"#E6E649","letterGroup":"B"},
  {"id":"B19","code":"B19","name":"","hex":"#24BB8C","letterGroup":"B"},
  {"id":"B20","code":"B20","name":"","hex":"#C2F0CC","letterGroup":"B"},
  {"id":"B21","code":"B21","name":"","hex":"#156A68","letterGroup":"B"},
  {"id":"B22","code":"B22","name":"","hex":"#0B3C43","letterGroup":"B"},
  {"id":"B23","code":"B23","name":"","hex":"#303A21","letterGroup":"B"},
  {"id":"B24","code":"B24","name":"","hex":"#EEFCA5","letterGroup":"B"},
  {"id":"B25","code":"B25","name":"","hex":"#4E84CD","letterGroup":"B"},
  {"id":"B26","code":"B26","name":"","hex":"#807A35","letterGroup":"B"},
  {"id":"B27","code":"B27","name":"","hex":"#CCE1AF","letterGroup":"B"},
  {"id":"B28","code":"B28","name":"","hex":"#9EE5B9","letterGroup":"B"},
  {"id":"B29","code":"B29","name":"","hex":"#C5E254","letterGroup":"B"},
  {"id":"B30","code":"B30","name":"","hex":"#E2FCB1","letterGroup":"B"},
  {"id":"B31","code":"B31","name":"","hex":"#BDE792","letterGroup":"B"},
  {"id":"B32","code":"B32","name":"","hex":"#9CAB5A","letterGroup":"B"},

  {"id":"C1","code":"C1","name":"","hex":"#E8FFE7","letterGroup":"C"},
  {"id":"C2","code":"C2","name":"","hex":"#A9F9FC","letterGroup":"C"},
  {"id":"C3","code":"C3","name":"","hex":"#A0E2FB","letterGroup":"C"},
  {"id":"C4","code":"C4","name":"","hex":"#41CCFF","letterGroup":"C"},
  {"id":"C5","code":"C5","name":"","hex":"#01ACEB","letterGroup":"C"},
  {"id":"C6","code":"C6","name":"","hex":"#50AAF0","letterGroup":"C"},
  {"id":"C7","code":"C7","name":"","hex":"#3677D2","letterGroup":"C"},
  {"id":"C8","code":"C8","name":"","hex":"#0F54C0","letterGroup":"C"},
  {"id":"C9","code":"C9","name":"","hex":"#3248CA","letterGroup":"C"},
  {"id":"C10","code":"C10","name":"","hex":"#3EBCE2","letterGroup":"C"},
  {"id":"C11","code":"C11","name":"","hex":"#28DDDE","letterGroup":"C"},
  {"id":"C12","code":"C12","name":"","hex":"#1C334D","letterGroup":"C"},
  {"id":"C13","code":"C13","name":"","hex":"#CDE8FF","letterGroup":"C"},
  {"id":"C14","code":"C14","name":"","hex":"#D5FDFF","letterGroup":"C"},
  {"id":"C15","code":"C15","name":"","hex":"#22C4C6","letterGroup":"C"},
  {"id":"C16","code":"C16","name":"","hex":"#1557A8","letterGroup":"C"},
  {"id":"C17","code":"C17","name":"","hex":"#04D1F6","letterGroup":"C"},
  {"id":"C18","code":"C18","name":"","hex":"#103344","letterGroup":"C"},
  {"id":"C19","code":"C19","name":"","hex":"#1887A2","letterGroup":"C"},
  {"id":"C20","code":"C20","name":"","hex":"#176DAF","letterGroup":"C"},
  {"id":"C21","code":"C21","name":"","hex":"#BEDDFE","letterGroup":"C"},
  {"id":"C22","code":"C22","name":"","hex":"#67B4BE","letterGroup":"C"},
  {"id":"C23","code":"C23","name":"","hex":"#C8E2FF","letterGroup":"C"},
  {"id":"C24","code":"C24","name":"","hex":"#7CC4FF","letterGroup":"C"},
  {"id":"C25","code":"C25","name":"","hex":"#A9E5E5","letterGroup":"C"},
  {"id":"C26","code":"C26","name":"","hex":"#3CAED8","letterGroup":"C"},
  {"id":"C27","code":"C27","name":"","hex":"#3D8FFA","letterGroup":"C"},
  {"id":"C28","code":"C28","name":"","hex":"#BBCFED","letterGroup":"C"},
  {"id":"C29","code":"C29","name":"","hex":"#34488E","letterGroup":"C"},

  {"id":"D1","code":"D1","name":"","hex":"#AEB4F2","letterGroup":"D"},
  {"id":"D2","code":"D2","name":"","hex":"#85BEDD","letterGroup":"D"},
  {"id":"D3","code":"D3","name":"","hex":"#2F54AF","letterGroup":"D"},
  {"id":"D4","code":"D4","name":"","hex":"#182A84","letterGroup":"D"},
  {"id":"D5","code":"D5","name":"","hex":"#BB43C5","letterGroup":"D"},
  {"id":"D6","code":"D6","name":"","hex":"#AC78DE","letterGroup":"D"},
  {"id":"D7","code":"D7","name":"","hex":"#8854B3","letterGroup":"D"},
  {"id":"D8","code":"D8","name":"","hex":"#E2D3FF","letterGroup":"D"},
  {"id":"D9","code":"D9","name":"","hex":"#D5B9F8","letterGroup":"D"},
  {"id":"D10","code":"D10","name":"","hex":"#361851","letterGroup":"D"},
  {"id":"D11","code":"D11","name":"","hex":"#B9BAE1","letterGroup":"D"},
  {"id":"D12","code":"D12","name":"","hex":"#DE9AD4","letterGroup":"D"},
  {"id":"D13","code":"D13","name":"","hex":"#B90095","letterGroup":"D"},
  {"id":"D14","code":"D14","name":"","hex":"#8B279B","letterGroup":"D"},
  {"id":"D15","code":"D15","name":"","hex":"#2F1F90","letterGroup":"D"},
  {"id":"D16","code":"D16","name":"","hex":"#E3E1EE","letterGroup":"D"},
  {"id":"D17","code":"D17","name":"","hex":"#C4D4F6","letterGroup":"D"},
  {"id":"D18","code":"D18","name":"","hex":"#A45EC7","letterGroup":"D"},
  {"id":"D19","code":"D19","name":"","hex":"#DBC3D7","letterGroup":"D"},
  {"id":"D20","code":"D20","name":"","hex":"#9C32B2","letterGroup":"D"},
  {"id":"D21","code":"D21","name":"","hex":"#9A009B","letterGroup":"D"},
  {"id":"D22","code":"D22","name":"","hex":"#8785A1","letterGroup":"D"},
  {"id":"D23","code":"D23","name":"","hex":"#EBDAFC","letterGroup":"D"},
  {"id":"D24","code":"D24","name":"","hex":"#7786E5","letterGroup":"D"},
  {"id":"D25","code":"D25","name":"","hex":"#494FC7","letterGroup":"D"},
  {"id":"D26","code":"D26","name":"","hex":"#DFC2F8","letterGroup":"D"},

  {"id":"E1","code":"E1","name":"","hex":"#FDD3CC","letterGroup":"E"},
  {"id":"E2","code":"E2","name":"","hex":"#FEC0DF","letterGroup":"E"},
  {"id":"E3","code":"E3","name":"","hex":"#FFB7E7","letterGroup":"E"},
  {"id":"E4","code":"E4","name":"","hex":"#E8649E","letterGroup":"E"},
  {"id":"E5","code":"E5","name":"","hex":"#F551A2","letterGroup":"E"},
  {"id":"E6","code":"E6","name":"","hex":"#F13D74","letterGroup":"E"},
  {"id":"E7","code":"E7","name":"","hex":"#C63478","letterGroup":"E"},
  {"id":"E8","code":"E8","name":"","hex":"#FFDBE9","letterGroup":"E"},
  {"id":"E9","code":"E9","name":"","hex":"#E970CC","letterGroup":"E"},
  {"id":"E10","code":"E10","name":"","hex":"#D33793","letterGroup":"E"},
  {"id":"E11","code":"E11","name":"","hex":"#FCDDD2","letterGroup":"E"},
  {"id":"E12","code":"E12","name":"","hex":"#F78FC3","letterGroup":"E"},
  {"id":"E13","code":"E13","name":"","hex":"#B5006D","letterGroup":"E"},
  {"id":"E14","code":"E14","name":"","hex":"#FFD1BA","letterGroup":"E"},
  {"id":"E15","code":"E15","name":"","hex":"#F8C7C9","letterGroup":"E"},
  {"id":"E16","code":"E16","name":"","hex":"#FFF3EB","letterGroup":"E"},
  {"id":"E17","code":"E17","name":"","hex":"#FFE2EA","letterGroup":"E"},
  {"id":"E18","code":"E18","name":"","hex":"#FFC7D8","letterGroup":"E"},
  {"id":"E19","code":"E19","name":"","hex":"#FEBAD5","letterGroup":"E"},
  {"id":"E20","code":"E20","name":"","hex":"#DBC7D1","letterGroup":"E"},
  {"id":"E21","code":"E21","name":"","hex":"#BD9DA1","letterGroup":"E"},
  {"id":"E22","code":"E22","name":"","hex":"#B785A1","letterGroup":"E"},
  {"id":"E23","code":"E23","name":"","hex":"#937A8D","letterGroup":"E"},
  {"id":"E24","code":"E24","name":"","hex":"#E1BCE8","letterGroup":"E"},

  {"id":"F1","code":"F1","name":"","hex":"#FD957B","letterGroup":"F"},
  {"id":"F2","code":"F2","name":"","hex":"#FC3D46","letterGroup":"F"},
  {"id":"F3","code":"F3","name":"","hex":"#F74941","letterGroup":"F"},
  {"id":"F4","code":"F4","name":"","hex":"#FC283C","letterGroup":"F"},
  {"id":"F5","code":"F5","name":"","hex":"#E7002F","letterGroup":"F"},
  {"id":"F6","code":"F6","name":"","hex":"#943630","letterGroup":"F"},
  {"id":"F7","code":"F7","name":"","hex":"#971937","letterGroup":"F"},
  {"id":"F8","code":"F8","name":"","hex":"#BC0028","letterGroup":"F"},
  {"id":"F9","code":"F9","name":"","hex":"#E2677A","letterGroup":"F"},
  {"id":"F10","code":"F10","name":"","hex":"#8A4526","letterGroup":"F"},
  {"id":"F11","code":"F11","name":"","hex":"#5A2121","letterGroup":"F"},
  {"id":"F12","code":"F12","name":"","hex":"#FD4E6A","letterGroup":"F"},
  {"id":"F13","code":"F13","name":"","hex":"#F35744","letterGroup":"F"},
  {"id":"F14","code":"F14","name":"","hex":"#FF9A4D","letterGroup":"F"},
  {"id":"F15","code":"F15","name":"","hex":"#D30022","letterGroup":"F"},
  {"id":"F16","code":"F16","name":"","hex":"#FEC2A6","letterGroup":"F"},
  {"id":"F17","code":"F17","name":"","hex":"#E69C79","letterGroup":"F"},
  {"id":"F18","code":"F18","name":"","hex":"#D37C46","letterGroup":"F"},
  {"id":"F19","code":"F19","name":"","hex":"#E65C79","letterGroup":"F"},
  {"id":"F20","code":"F20","name":"","hex":"#CD9391","letterGroup":"F"},
  {"id":"F21","code":"F21","name":"","hex":"#F7B4C6","letterGroup":"F"},
  {"id":"F22","code":"F22","name":"","hex":"#FDCD00","letterGroup":"F"},
  {"id":"F23","code":"F23","name":"","hex":"#F67666","letterGroup":"F"},
  {"id":"F24","code":"F24","name":"","hex":"#E698AA","letterGroup":"F"},
  {"id":"F25","code":"F25","name":"","hex":"#E5484F","letterGroup":"F"},

  {"id":"G1","code":"G1","name":"","hex":"#FFE2CE","letterGroup":"G"},
  {"id":"G2","code":"G2","name":"","hex":"#FFC4AA","letterGroup":"G"},
  {"id":"G3","code":"G3","name":"","hex":"#F4C3A5","letterGroup":"G"},
  {"id":"G4","code":"G4","name":"","hex":"#E1B383","letterGroup":"G"},
  {"id":"G5","code":"G5","name":"","hex":"#EDB045","letterGroup":"G"},
  {"id":"G6","code":"G6","name":"","hex":"#E99C17","letterGroup":"G"},
  {"id":"G7","code":"G7","name":"","hex":"#9D5B3E","letterGroup":"G"},
  {"id":"G8","code":"G8","name":"","hex":"#753832","letterGroup":"G"},
  {"id":"G9","code":"G9","name":"","hex":"#E5B843","letterGroup":"G"},
  {"id":"G10","code":"G10","name":"","hex":"#D98C39","letterGroup":"G"},
  {"id":"G11","code":"G11","name":"","hex":"#E0C593","letterGroup":"G"},
  {"id":"G12","code":"G12","name":"","hex":"#FFC890","letterGroup":"G"},
  {"id":"G13","code":"G13","name":"","hex":"#B7714A","letterGroup":"G"},
  {"id":"G14","code":"G14","name":"","hex":"#8D614C","letterGroup":"G"},
  {"id":"G15","code":"G15","name":"","hex":"#FCF9E0","letterGroup":"G"},
  {"id":"G16","code":"G16","name":"","hex":"#F2D9BA","letterGroup":"G"},
  {"id":"G17","code":"G17","name":"","hex":"#C2A98A","letterGroup":"G"},
  {"id":"G18","code":"G18","name":"","hex":"#FFE4CC","letterGroup":"G"},
  {"id":"G19","code":"G19","name":"","hex":"#E07935","letterGroup":"G"},
  {"id":"G20","code":"G20","name":"","hex":"#A94023","letterGroup":"G"},
  {"id":"G21","code":"G21","name":"","hex":"#B88558","letterGroup":"G"},

  {"id":"H1","code":"H1","name":"","hex":"#FDFBFF","letterGroup":"H"},
  {"id":"H2","code":"H2","name":"","hex":"#FEFFFF","letterGroup":"H"},
  {"id":"H3","code":"H3","name":"","hex":"#B6B1BA","letterGroup":"H"},
  {"id":"H4","code":"H4","name":"","hex":"#89858C","letterGroup":"H"},
  {"id":"H5","code":"H5","name":"","hex":"#48464E","letterGroup":"H"},
  {"id":"H6","code":"H6","name":"","hex":"#2F2B2F","letterGroup":"H"},
  {"id":"H7","code":"H7","name":"","hex":"#000000","letterGroup":"H"},
  {"id":"H8","code":"H8","name":"","hex":"#E7D6DB","letterGroup":"H"},
  {"id":"H9","code":"H9","name":"","hex":"#EDEDED","letterGroup":"H"},
  {"id":"H10","code":"H10","name":"","hex":"#EEE9EA","letterGroup":"H"},
  {"id":"H11","code":"H11","name":"","hex":"#CECDD5","letterGroup":"H"},
  {"id":"H12","code":"H12","name":"","hex":"#FFF5ED","letterGroup":"H"},
  {"id":"H13","code":"H13","name":"","hex":"#F5ECED","letterGroup":"H"},
  {"id":"H14","code":"H14","name":"","hex":"#CFD7D3","letterGroup":"H"},
  {"id":"H15","code":"H15","name":"","hex":"#9BA6A8","letterGroup":"H"},
  {"id":"H16","code":"H16","name":"","hex":"#1D1414","letterGroup":"H"},
  {"id":"H17","code":"H17","name":"","hex":"#FFEFF2","letterGroup":"H"},
  {"id":"H18","code":"H18","name":"","hex":"#FFFDFF","letterGroup":"H"},
  {"id":"H19","code":"H19","name":"","hex":"#FEFEF2","letterGroup":"H"},
  {"id":"H20","code":"H20","name":"","hex":"#949FA3","letterGroup":"H"},
  {"id":"H21","code":"H21","name":"","hex":"#FFFBE1","letterGroup":"H"},
  {"id":"H22","code":"H22","name":"","hex":"#CACAD4","letterGroup":"H"},
  {"id":"H23","code":"H23","name":"","hex":"#9A9D94","letterGroup":"H"},

  {"id":"M1","code":"M1","name":"","hex":"#BCC6B8","letterGroup":"M"},
  {"id":"M2","code":"M2","name":"","hex":"#BAA39B","letterGroup":"M"},
  {"id":"M3","code":"M3","name":"","hex":"#697D80","letterGroup":"M"},
  {"id":"M4","code":"M4","name":"","hex":"#E3D2BC","letterGroup":"M"},
  {"id":"M5","code":"M5","name":"","hex":"#D0CCAA","letterGroup":"M"},
  {"id":"M6","code":"M6","name":"","hex":"#B0A782","letterGroup":"M"},
  {"id":"M7","code":"M7","name":"","hex":"#B4A497","letterGroup":"M"},
  {"id":"M8","code":"M8","name":"","hex":"#A38281","letterGroup":"M"},
  {"id":"M9","code":"M9","name":"","hex":"#B38182","letterGroup":"M"},
  {"id":"M10","code":"M10","name":"","hex":"#A5828C","letterGroup":"M"},
  {"id":"M11","code":"M11","name":"","hex":"#C77362","letterGroup":"M"},
  {"id":"M12","code":"M12","name":"","hex":"#C74449","letterGroup":"M"},
  {"id":"M13","code":"M13","name":"","hex":"#D19066","letterGroup":"M"},
  {"id":"M14","code":"M14","name":"","hex":"#C79362","letterGroup":"M"},
  {"id":"M15","code":"M15","name":"","hex":"#757D78","letterGroup":"M"}
]
```

> **说明**：以上 221 色 HEX 值从官方色值对照图精确读取（9 张色系对照表交叉核对）。`name` 字段暂为空，后续可补充颜色名称。

---

## 4. 页面结构（4 个底部 Tab）

```
/                       → 库存 Tab
  ├── 豆列表（展示、筛选、排序）
  ├── 单豆快捷修改
  ├── 批量操作
  ├── 识图出库入口
  └── 低库存告警横幅

/record                 → 变更记录 Tab
  ├── 变更记录列表
  ├── 筛选（类型、时间、编号）
  └── 回溯功能

/gallery                → 图库 Tab
  ├── 子 Tab：拼豆图库（上传、重命名、删除）
  └── 子 Tab：图纸制作（图片转像素图）

/me                     → 我的 Tab
  ├── 全局阈值设置
  └── 重置数据

--- 子页面（不在底部导航）---
/bead/:id               → 豆详情页
/scan                   → 识图出库流程页
/pattern/:id            → 图纸详情页
/pixel/:id              → 像素图纸详情页
/make                   → 图纸制作流程页
```

---

## 5. 功能需求

### 5.1 库存 Tab（首页）

#### 5.1.1 豆列表展示
- 以卡片形式展示所有 221 色拼豆
- 每个卡片展示：颜色色块、编号（code）、颜色名称、当前余量（单位：粒）
- **字母筛选**：顶部字母导航栏 `A | B | C | D | E | F | G | H | M`，点击快速筛选
- **搜索框**：支持按编号或颜色名称模糊搜索
- **排序**：
  - 按编号升序/降序（默认升序 A1 → M15）
  - 按余量升序/降序
  - 点击排序按钮切换，有明确排序状态指示（↑ ↓）

#### 5.1.2 单豆快捷修改
- 卡片上直接显示 `+` `-` 按钮，点击增减（默认步长 10 粒，可在设置中配置）
- 点击卡片进入**豆详情页**，可精确输入数量、查看历史记录、设置该豆独立阈值
- 操作后 Toast 提示：「A1 出库 10粒，剩余 120粒」

#### 5.1.3 批量操作
- 顶部「批量」按钮进入批量模式：
  - 卡片变为可多选状态
  - 底部固定操作栏，显示已选数量
  - 支持「批量入库」「批量出库」
  - 输入统一变更数量（粒），一键确认
  - 为每个选中的豆生成独立变更记录

#### 5.1.4 识图出库入口
- 首页 FAB（悬浮按钮）或顶部「识图出库」按钮
- 点击进入识图出库流程页

#### 5.1.5 低库存告警横幅
- 如有豆量低于阈值，首页顶部显示红色告警横幅：「X 种拼豆库存不足」
- 点击横幅展开/跳转到低库存豆列表
- 低库存卡片显示红色边框或警告角标

### 5.2 识图出库（核心功能）

#### 5.2.1 流程
1. 用户上传一张拼豆图纸（支持拍照或相册选择）
2. 前端进行图像识别（OCR），提取图纸上所有编号并统计频次
3. 识别结果展示：列出识别到的编号、颜色名称、统计数量（粒）
4. 用户确认或修正识别结果（可编辑数量、删除误识别项、手动添加遗漏项）
5. 点击「确认出库」，弹出二次确认弹窗（展示总出库粒数和涉及颜色数）
6. 确认后自动扣减库存，生成变更记录（reason = 'image_recognition'）
7. 自动将图纸保存到「拼豆图库」（弹窗提示已保存）

#### 5.2.2 OCR 策略
- **引擎**：Tesseract.js（浏览器端，无需网络）
- **预处理**：上传后先对图像进行放大、灰度化、二值化、去网格线处理，提高小字识别率
- **识别逻辑**：
  - 全图 OCR 识别所有文字
  - 正则过滤：匹配 `/^[A-HM]\d{1,2}$/` 格式的编号
  - 统计每个有效编号的出现频次，即为该颜色所需粒数
- **后处理**：
  - 过滤不在 221 色库中的编号（如识别错误产生的乱码）
  - 对识别结果按编号排序展示
- **兜底**：如 OCR 置信度过低，提示用户「识别结果可能不准确，请仔细核对」

#### 5.2.3 确认与修正
- 识别结果页每行展示：编号 | 颜色色块 | 颜色名称 | 数量（可编辑输入框）| 删除按钮
- 底部有「添加颜色」按钮，手动输入编号补充遗漏
- 确认出库前必须二次弹窗确认

### 5.3 变更记录 Tab

- 按时间倒序展示所有变更记录
- 每条记录展示：时间、编号、颜色色块、类型（入库/出库）、数量、原因、变更后余量
- 支持筛选：按类型（入库/出库）、按时间范围、按编号搜索
- 支持回溯：点击记录旁「恢复」按钮，二次确认后将该豆库存恢复到 snapshot.qtyAfter，并生成新记录（reason = 'restore'）

### 5.4 图库 Tab

#### 5.4.1 子 Tab：拼豆图库
- 网格展示所有上传的图纸缩略图 + 名称 + 上传时间
- 支持单张/多张手动上传，上传时弹窗输入名称（可选，兜底自动命名：`图纸_YYYYMMDD_HHmmss`）
- 支持重命名、删除
- 点击图纸进入图纸详情页，查看大图 + 关联出库记录
- 识图出库时自动保存的图纸，名称默认用识图时间命名

#### 5.4.2 子 Tab：图纸制作（新增功能）

**功能描述**：用户上传一张普通图片，系统自动将其转换为拼豆像素图纸，输出带编号的像素格图和用量清单。

**流程**：
1. 用户上传一张图片（支持拍照或相册选择）
2. 选择图纸尺寸：
   - 提供多个预设选项：`29×29` | `50×50` | `80×80` | `100×100`
   - 支持自定义尺寸：输入宽度和高度（格数），范围 10~200
   - 保持原图宽高比，等比例缩放
3. 系统处理：
   - 使用 Canvas API 将图片缩放为所选尺寸
   - 提取每个像素的颜色（RGB）
   - 计算该颜色与 221 色 Mard 色库中每种颜色的欧几里得距离
   - 选择距离最近的 Mard 编号作为该格子的颜色
   - 生成二维编号数组（gridData）
   - 统计每种颜色的使用数量（beadCounts）
4. 结果展示：
   - 像素图纸预览：网格图，每个格子显示对应颜色色块和编号
   - 支持缩放查看（双指捏合 / 滚轮缩放）
   - 用量清单：列出本次图纸使用的所有颜色编号、色块、粒数
   - 支持一键「按此图纸出库」（直接跳转识图出库确认页）
5. 保存：
   - 点击「保存图纸」，输入名称（可选），保存到像素图纸库
   - 保存后可在图库 Tab 的「图纸制作」子 Tab 中查看历史

**颜色匹配算法**：
```typescript
function findClosestBead(r: number, g: number, b: number): BeadColor {
  let minDist = Infinity;
  let closest = null;
  for (const bead of beadColors) {
    const [br, bg, bb] = hexToRgb(bead.hex);
    const dist = Math.sqrt(
      Math.pow(r - br, 2) + Math.pow(g - bg, 2) + Math.pow(b - bb, 2)
    );
    if (dist < minDist) {
      minDist = dist;
      closest = bead;
    }
  }
  return closest;
}
```

**优化选项（可选，P2）**：
- 颜色数量限制：用户可设置最多使用 N 种颜色，系统做颜色聚类（k-means）后再匹配
- 抖动算法：使用 Floyd-Steinberg 等抖动算法减少色带，提升视觉效果
- 边框检测：自动识别图片主体，去除背景

### 5.5 我的 Tab（设置）

#### 5.5.1 全局阈值设置
- 设置低库存告警的默认阈值（默认 50 粒）
- 说明：单个豆可在豆详情页设置独立阈值，覆盖全局默认值

#### 5.5.2 快捷步长设置
- 设置首页卡片 `+` `-` 按钮的默认步长（默认 10 粒）

#### 5.5.3 重置数据
- 一键重置所有库存为 0（保留 221 色基础数据）
- 二次确认弹窗，防止误操作

---

## 6. 响应式适配

| 断点 | 布局 |
|------|------|
| 移动端（< 768px） | 单列卡片列表，底部 4 Tab 导航 |
| 平板（768px - 1024px） | 双列卡片网格，底部 4 Tab 导航 |
| PC（> 1024px） | 三列卡片网格，左侧边栏导航（或顶部导航） |

---

## 7. 交互状态设计

| 场景 | 交互 |
|------|------|
| 库存变更 | 数字滚动动画 + Toast 提示 |
| 识图识别中 | 全屏 Loading，提示「正在识别图纸...预计 3-5 秒」 |
| 识图完成 | 结果页展示，置信度低的项标黄提示 |
| 图纸制作处理中 | 全屏 Loading，提示「正在生成像素图纸...」 |
| 图纸制作完成 | 展示像素图预览 + 用量清单 |
| 低库存告警 | 首页顶部红色横幅，列表卡片红色边框 |
| 批量模式 | 卡片出现复选框，底部操作栏上滑出现 |

---

## 8. 确认弹窗清单

- 识图出库确认（展示总览）
- 恢复到某状态确认
- 批量操作确认
- 删除图纸确认
- 重置数据确认
- 图纸制作保存确认

---

## 9. 非功能需求

### 9.1 性能
- 首次加载 < 2s（221 色数据本地存储）
- 列表滚动流畅
- 图纸制作处理：100×100 像素图处理时间 < 3s
- 图片压缩：上传图纸前压缩至 1920px 宽，控制 IndexedDB 体积

### 9.2 存储
- 使用 IndexedDB 存储所有数据（dexie.js）
- 提供数据导出功能（JSON），方便备份和迁移
- 提供数据导入功能，支持合并或全量覆盖

### 9.3 离线可用
- PWA 化，支持离线访问和操作
- 所有数据本地存储，无需网络
- OCR 使用浏览器端 Tesseract.js

### 9.4 扩展性
- 预留多品牌支持（当前只有 Mard）
- 预留多用户支持（当前单用户）

---

## 10. 开发优先级

### P0（核心 MVP）
1. 221 色基础数据初始化与展示（含完整 HEX 色值）
2. 单豆手动增减库存（粒）
3. 变更记录与快照
4. 低库存告警（默认阈值 50 粒）

### P1（增强体验）
5. 字母筛选与排序
6. 批量进出库
7. 阈值自定义 + 快捷步长设置
8. 拼豆图手动上传与管理

### P2（高级功能）
9. 识图出库（Tesseract.js OCR + 频次统计）
10. 识图自动保存图纸
11. 图纸制作（图片转像素图 + 颜色匹配）
12. 数据导入/导出
13. PWA 离线支持
14. 状态回溯功能

---

## 11. 附录

### 11.1 命名规范
- 组件：PascalCase（如 `BeadCard.tsx`）
- 页面：kebab-case（如 `bead-detail.vue`）
- 接口/类型：PascalCase（如 `BeadColor`）
- 常量：UPPER_SNAKE_CASE

### 11.2 图纸制作算法说明
- 颜色匹配使用 RGB 空间的欧几里得距离
- 可扩展 Lab 颜色空间匹配，更符合人眼感知（P2 优化）
- 处理大尺寸图片时，建议先用 Canvas 缩放到目标尺寸后再逐像素处理

# 仓库管理程序（对接领星 WMS / OMS OpenAPI）

一个 Node.js + Express 后端，封装 OMS OpenAPI 的签名和调用逻辑，
配一个简单网页前端，可以管理：仓库、产品、入库单、出库单、库存查询。

## ⚠️ 上线前必须核对一件事：签名算法

我在公开网络上只找到了 **同公司另一个文档站**（`apidoc-wms.xlwms.com`，智能设备接口）
公开的加签说明，OMS 这边（`apidoc-oms.xlwms.com`）的"认证/签名"文档页面需要登录才能看，
我这边抓取不到，所以 `src/sign.js` 里的实现是**按同公司同类接口的规则推测**的：

```
sign = HMAC_SHA256(
  key = appSecret,
  message = appSecret + path + 按字典序拼接的"参数名+参数值"字符串 + appSecret
)
```

**在正式联调前，请打开你们 OMS 后台 → API 文档 → 开发指南/认证说明，**
**把"签名生成规则"那一页的截图或文字发给我，我立刻按实际规则改 `src/sign.js`。**
如果规则不对，所有请求都会返回签名错误（通常是 401/403 或 "sign不匹配"），
和网络问题看起来很像，容易走弯路。

## 目录结构

```
warehouse-app/
  src/
    sign.js         // 签名算法（需核对，见上）
    xlwmsClient.js  // 封装的 OMS API 调用（入库/出库/库存/产品/仓库）
    server.js       // Express 后端，暴露给前端的简化路由
  public/
    index.html      // 前端页面
    app.js          // 前端逻辑
    style.css
  .env.example      // 环境变量模板
```

## 使用步骤

1. 安装依赖
   ```bash
   cd warehouse-app
   npm install
   ```

2. 配置密钥：复制 `.env.example` 为 `.env`，填入你的 appKey 和 appSecret
   ```bash
   cp .env.example .env
   ```
   ```
   XLWMS_APP_KEY=你的appKey
   XLWMS_APP_SECRET=你的appSecret
   XLWMS_BASE_URL=https://api.xlwms.com
   PORT=3000
   ```

3. 启动
   ```bash
   npm start
   ```
   打开浏览器访问 http://localhost:3000

## 目前已封装的接口

| 模块 | 接口 | 说明 |
|---|---|---|
| 仓库 | `getWarehouseList` | 仓库列表查询 |
| 产品 | `getProductPage` / `batchCreateProduct` | 产品列表 / 批量创建产品 |
| 入库单 | `createInboundOrder` / `getInboundOrderPage` / `getInboundOrderDetail` | 创建/查询入库单 |
| 出库单 | `createOutboundOrder` / `getOutboundOrderPage` / `getOutboundOrderDetail` | 创建/查询小包出库单 |
| 库存 | `getInventoryPage` | 分页查询综合库存 |

其余接口（退件单、工单、FBA退货等）在 `xlwmsClient.js` 里预留了同样的写法，
需要哪个我可以直接加，模式都是一样的（拼 path、拼 data、调用 `callApi`）。

## 关于 CORS 和安全

appSecret 绝对不能放到前端 JS 里。这也是为什么必须有这个 Node 后端：
浏览器只请求你自己的后端（比如 `/api/inbound`），后端再用 appSecret 签名后
去请求 `api.xlwms.com`，appSecret 全程不会出现在浏览器里。

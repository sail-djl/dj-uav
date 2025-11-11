
### UavRouteController API 文档

#### `POST /updateKmz`
- **功能**：根据传入的航线信息编辑已有的 KMZ 文件。
- **请求体**：`UavRouteReq`（航线基础信息、航点、载荷等）。
- **响应**：无返回体；成功时 HTTP 200。
- **备注**：若编辑失败，将由全局异常处理器返回错误信息。

#### `POST /buildKmz`
- **功能**：根据传入的航线数据生成新的 KMZ 文件。
- **请求体**：`UavRouteReq`。
- **响应**：无返回体；成功时 HTTP 200。
- **备注**：生成的 KMZ 文件位置由业务层确定。

#### `POST /parseKmz`
- **功能**：解析指定 URL 的 KMZ 文件，返回航线详情。
- **请求参数**：
  - `fileUrl` (String)：KMZ 文件的访问地址。
- **响应**：`KmzInfoVO`，包含解析后的航线信息。
- **异常**：解析失败时抛出 `IOException`，由全局异常处理器统一封装。

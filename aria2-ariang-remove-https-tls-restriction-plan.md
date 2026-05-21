# AriaNg 移除 HTTPS/TLS 限制 改造方案

## 背景

AriaNg 当前存在一个安全限制：当用户通过 HTTPS 访问 AriaNg 页面时，RPC 协议选择中的 `http` 和 `ws` 选项会被禁用，且新建 RPC 配置时协议会被强制设为 `https`。

这是因为浏览器混合内容（Mixed Content）安全策略会阻止从 HTTPS 页面发起不安全的 HTTP/WS 请求。

## 改造目标

取消此限制，即使 AriaNg 通过 HTTPS 访问，也允许用户自由选择 RPC 协议（http/https/ws/wss）。

## 涉及文件

| 文件 | 改动内容 |
|------|----------|
| `src/scripts/services/ariaNgSettingService.js` | 修改核心检测函数 |
| `src/views/settings-ariang.html` | 移除视图中的禁用逻辑和提示 |
| `src/scripts/controllers/settings-ariang.js` | 移除不再需要的状态注入 |

## 详细修改步骤

### 1. `src/scripts/services/ariaNgSettingService.js`

**函数 `isInsecureProtocolDisabled()`（第 44-48 行）**

修改前：
```javascript
var isInsecureProtocolDisabled = function () {
    var protocol = $location.protocol();
    return protocol === 'https';
};
```

修改后：
```javascript
var isInsecureProtocolDisabled = function () {
    return false;
};
```

此改动是核心。`initRpcSettingWithDefaultHostAndProtocol()`（第 207 行）中的判断条件自然跳过，不再强制覆盖协议为 https。

### 2. `src/views/settings-ariang.html`

**RPC 协议选择器（第 370-382 行）**

修改前：
```html
<div class="setting-key setting-key-without-desc col-sm-4">
    <span translate>Aria2 RPC Protocol</span>
    <span class="asterisk">*</span>
    <i class="icon-primary fa fa-question-circle" ng-tooltip-container="body" ng-tooltip-placement="top"
       ng-tooltip="{{'Http and WebSocket would be disabled when accessing AriaNg via Https.' | translate}}"></i>
</div>
<div class="setting-value col-sm-8">
    <select class="form-control" style="width: 100%;" ng-model="setting.protocol" ng-change="updateRpcSetting(setting, 'protocol')">
        <option value="http" ng-disabled="::(context.isInsecureProtocolDisabled)" ng-bind="('Http' + (context.isInsecureProtocolDisabled ? ' (Disabled)' : '')) | translate">Http</option>
        <option value="https" translate>Https</option>
        <option value="ws" ng-disabled="::(context.isInsecureProtocolDisabled)" ng-bind="('WebSocket' + (context.isInsecureProtocolDisabled ? ' (Disabled)' : '')) | translate">WebSocket</option>
        <option value="wss" translate>WebSocket (Security)</option>
    </select>
</div>
```

修改后：
```html
<div class="setting-key setting-key-without-desc col-sm-4">
    <span translate>Aria2 RPC Protocol</span>
    <span class="asterisk">*</span>
</div>
<div class="setting-value col-sm-8">
    <select class="form-control" style="width: 100%;" ng-model="setting.protocol" ng-change="updateRpcSetting(setting, 'protocol')">
        <option value="http">Http</option>
        <option value="https" translate>Https</option>
        <option value="ws">WebSocket</option>
        <option value="wss" translate>WebSocket (Security)</option>
    </select>
</div>
```

变更说明：
- 移除问号提示图标及 tooltip 文本
- `http` 选项：移除 `ng-disabled` 和条件显示的 `(Disabled)` 文字
- `ws` 选项：同上

### 3. `src/scripts/controllers/settings-ariang.js`

**状态注入（第 76 行）**

修改前：
```javascript
isInsecureProtocolDisabled: ariaNgSettingService.isInsecureProtocolDisabled(),
```

修改后：直接删除此行。

## 不改动的文件

| 文件 | 说明 |
|------|------|
| `src/scripts/config/constants.js` | `defaultSecureProtocol: 'https'` 保留，仅作为新建 RPC 时的初始默认值 |
| `src/scripts/controllers/command.js` | URL 命令参数验证不涉及此限制逻辑 |
| `src/scripts/services/aria2WebSocketRpcService.js` | 底层通信服务无需修改 |
| `src/scripts/services/aria2HttpRpcService.js` | 底层通信服务无需修改 |
| `src/scripts/services/aria2RpcService.js` | RPC 调度器无需修改 |

## 浏览器兼容性注意事项

HTTPS 页面直接调用 HTTP/WS 接口会受到浏览器混合内容策略限制：

| 场景 | 浏览器行为 |
|------|-----------|
| HTTPS → `http://` RPC | 现代浏览器默认阻止 XHR 请求 |
| HTTPS → `ws://` RPC | 明确阻止 |
| HTTPS → `http://localhost:6800` | 部分浏览器允许（安全上下文例外） |

如需在生产环境使用，建议：
- 确保用户了解混合内容的风险
- 或在前端代理层面做协议转换
- 或将 AriaNg 和 RPC 部署在同一台机器的 localhost 上

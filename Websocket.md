### 一、WebSocket 典型应用场景

| 场景类型     | 具体案例                     | 特点说明               |
| :----------- | :--------------------------- | :--------------------- |
| 实时通信     | 聊天应用、在线客服           | 低延迟消息传递         |
| 实时数据推送 | 股票行情、赛事比分、监控系统 | 服务器主动推送更新     |
| 协同编辑     | 在线文档、设计协作工具       | 多人实时同步操作       |
| 在线游戏     | 多人在线游戏、实时对战       | 高频双向通信           |
| IoT 设备控制 | 智能家居控制、工业物联网     | 设备状态实时监控与控制 |
| 实时通知     | 订单状态更新、系统报警       | 即时触达用户           |

### 二、断线问题全面解决方案

#### 1. 如何检测断线

**心跳检测机制 (Heartbeat)：**

```typescript
// 开始心跳检测
private startHeartbeat(): void {
    if (this.stopWs) return;
    if (this.heartbeatTimer !== undefined) {
        this.closeHeartbeat();
    }
this.heartbeatTimer = window.setInterval(() => {
    if (this.socket && this.socket.readyState === WebSocket.OPEN) {
        this.socket.send(JSON.stringify({ type: 'heartBeat', data: {} }));
        this.log('WebSocket', '发送心跳数据...');
    } else {
        console.error('[WebSocket] 未连接');
    }
}, this.heartbeatInterval);
}

// 关闭心跳检测
private closeHeartbeat(): void {
    if (this.heartbeatTimer !== undefined) {
        clearInterval(this.heartbeatTimer);
        this.heartbeatTimer = undefined;
    }
}

this.socket.onmessage = (event: MessageEvent) => {
    this.dispatchEvent('message', event);
    // 收到消息时重置心跳计时器
    this.startHeartbeat();
};
```

**服务端心跳处理：**

```js
// Node.js 服务端示例
ws.on('message', (message) => {
  const data = JSON.parse(message);
  if (data.type === 'heartbeat') {
    // 立即响应心跳
    ws.send(JSON.stringify({ 
      type: 'pong', 
      serverTime: Date.now() 
    }));
  }
});
```

### 三、完整代码

[LvImrynu - 码上掘金](https://code.juejin.cn/pen/7538978395267596326)

```typescript
// 定义事件监听器类型
type EventListener = (data?: any) => void;
// Log类实现
class Log {
  static console: boolean = true;
  log(title: string, text: string): void {
    if (!Log.console) return;
    if (import.meta.env?.MODE === 'production') return;
    const color = '#ff4d4f';
    console.log(
      `%c ${title} %c ${text} %c`,
      `background:${color};border:1px solid ${color}; padding: 1px; border-radius: 2px 0 0 2px; color: #fff;`,
      `border:1px solid ${color}; padding: 1px; border-radius: 0 2px 2px 0; color: ${color};`,
      'background:transparent',
    );
  }
  closeConsole(): void {
    Log.console = false;
  }
}
// 事件分发器类
class EventDispatcher extends Log {
  private listeners: Record<string, EventListener[]> = {};
  addEventListener(type: string, listener: EventListener): void {
    if (!this.listeners[type]) {
      this.listeners[type] = [];
    }
    if (this.listeners[type].indexOf(listener) === -1) {
      this.listeners[type].push(listener);
    }
  }
  removeEventListener(type: string): void {
    this.listeners[type] = [];
  }
  dispatchEvent(type: string, data?: any): void {
    const listenerArray = this.listeners[type] || [];
    if (listenerArray.length === 0) return;
    listenerArray.forEach((listener) => {
      listener.call(this, data);
    });
  }
}
// WebSocket客户端类
export class WebSocketClient extends EventDispatcher {
  // Socket URL
  private url: string;
  // Socket实例
  private socket: WebSocket | null = null;
  // 重连次数
  private reconnectAttempts: number = 0;
  // 最大重连次数
  private maxReconnectAttempts: number = 5;
  // 重连间隔（毫秒）
  private reconnectInterval: number = 10000;
  // 心跳间隔（毫秒）
  private heartbeatInterval: number = 30000;
  // 心跳计时器ID
  private heartbeatTimer: number | undefined;
  // 是否终止WebSocket
  private stopWs: boolean = false;
  constructor(url: string) {
    super();
    this.url = url;
  }
  // 生命周期钩子
  onopen(callback: (event?: Event) => void): void {
    this.addEventListener('open', callback);
  }
  onmessage(callback: (event?: MessageEvent) => void): void {
    this.addEventListener('message', callback);
  }
  onclose(callback: (event?: CloseEvent) => void): void {
    this.addEventListener('close', callback);
  }
  onerror(callback: (event?: Event) => void): void {
    this.addEventListener('error', callback);
  }
  // 发送消息
  send(message: string | ArrayBuffer | Blob | ArrayBufferView): void {
    if (this.socket && this.socket.readyState === WebSocket.OPEN) {
      this.socket.send(message);
    } else {
      console.error('[WebSocket] 未连接');
    }
  }
  // 初始化连接
  connect(): void {
    if (this.reconnectAttempts === 0) {
      this.log('WebSocket', `初始化连接中...          ${this.url}`);
    }
    if (this.socket && this.socket.readyState === WebSocket.OPEN) {
      return;
    }
    this.socket = new WebSocket(this.url);
    // WebSocket连接成功
    this.socket.onopen = (event: Event) => {
      this.stopWs = false;
      // 重置重连计数器
      this.reconnectAttempts = 0;
      // 启动心跳检测
      this.startHeartbeat();
      this.log(
        'WebSocket',
        `连接成功,等待服务端数据推送[onopen]...     ${this.url}`,
      );
      this.dispatchEvent('open', event);
    };
    this.socket.onmessage = (event: MessageEvent) => {
      this.dispatchEvent('message', event);
      // 收到消息时重置心跳计时器
      this.startHeartbeat();
    };
    this.socket.onclose = (event: CloseEvent) => {
      console.log(`WebSocket closed: ${event.code} - ${event.reason}`);
      if (this.reconnectAttempts === 0) {
        this.log('WebSocket', `连接断开[onclose]...    ${this.url}`);
      }
      if (!this.stopWs) {
        this.handleReconnect();
      }
      this.dispatchEvent('close', event);
    };
    this.socket.onerror = (event: Event) => {
      if (this.reconnectAttempts === 0) {
        this.log('WebSocket', `连接异常[onerror]...    ${this.url}`);
      }
      this.closeHeartbeat();
      this.dispatchEvent('error', event);
    };
  }
  // 断网重连逻辑
  private handleReconnect(): void {
    if (this.reconnectAttempts < this.maxReconnectAttempts) {
      this.reconnectAttempts++;
      this.log(
        'WebSocket',
        `尝试重连... (${this.reconnectAttempts}/${this.maxReconnectAttempts})       ${this.url}`,
      );
      setTimeout(() => {
        this.connect();
      }, this.reconnectInterval);
    } else {
      this.closeHeartbeat();
      this.log('WebSocket', `最大重连失败，终止重连: ${this.url}`);
    }
  }
  // 关闭连接
  close(): void {
    if (this.socket) {
      this.stopWs = true;
      this.socket.close();
      this.socket = null;
      this.removeEventListener('open');
      this.removeEventListener('message');
      this.removeEventListener('close');
      this.removeEventListener('error');
    }
    this.closeHeartbeat();
  }
  // 开始心跳检测
  private startHeartbeat(): void {
    if (this.stopWs) return;
    if (this.heartbeatTimer !== undefined) {
      this.closeHeartbeat();
    }
    this.heartbeatTimer = window.setInterval(() => {
      if (this.socket && this.socket.readyState === WebSocket.OPEN) {
        this.socket.send(JSON.stringify({ type: 'heartBeat', data: {} }));
        this.log('WebSocket', '发送心跳数据...');
      } else {
        console.error('[WebSocket] 未连接');
      }
    }, this.heartbeatInterval);
  }
  // 关闭心跳检测
  private closeHeartbeat(): void {
    if (this.heartbeatTimer !== undefined) {
      clearInterval(this.heartbeatTimer);
      this.heartbeatTimer = undefined;
    }
  }
  // 获取当前连接状态
  get readyState(): number | null {
    return this.socket ? this.socket.readyState : null;
  }
  // 获取当前重连次数
  get currentReconnectAttempts(): number {
    return this.reconnectAttempts;
  }
}
```

### 四、代码调用

```typescript
// 使用示例
export const setupWebSocketDemo = () => {
  // 创建WebSocket客户端实例
  const wsClient = new WebSocketClient('wss://echo.websocket.org');
  // 添加事件监听器
  wsClient.onopen((event) => {
    console.log('WebSocket连接已建立');
    // 发送测试消息
    wsClient.send(JSON.stringify({
      message: 'Hello WebSocket!',
      timestamp: new Date().toISOString()
    }));
  });
  wsClient.onmessage((event) => {
    const message = event?.data ? JSON.parse(event.data) : null;
    if (message && message.type === 'heartBeat') {
      console.log('收到心跳响应');
    } else {
      console.log('收到消息:', message);
    }
  });
  wsClient.onclose((event) => {
    console.log('WebSocket连接已关闭');
  });
  wsClient.onerror((event) => {
    console.error('WebSocket发生错误:', event);
  });
  // 连接WebSocket
  wsClient.connect();
  // 返回客户端实例以便在UI中使用
  return wsClient;
};
```

```react
// 完整的WebSocket Demo组件
export const WebSocketDemo = () => {
  const [client, setClient] = React.useState<WebSocketClient | null>(null);
  const [messages, setMessages] = React.useState<any[]>([]);
  React.useEffect(() => {
    const ws = setupWebSocketDemo();
    setClient(ws);
    // 添加消息监听器
    ws.onmessage((event) => {
      if (event?.data) {
        const msg = JSON.parse(event.data);
        if (msg.type !== 'heartBeat') {
          setMessages(prev => [...prev, {
            ...msg,
            direction: 'incoming',
            time: new Date().toLocaleTimeString()
          }]);
        }
      }
    });
    return () => {
      ws.close();
    };
  }, []);
  const handleSend = (msg: string) => {
    if (client && msg.trim()) {
      const messageObj = {
        message: msg,
        timestamp: new Date().toISOString()
      };
      client.send(JSON.stringify(messageObj));
      setMessages(prev => [...prev, {
        ...messageObj,
        direction: 'outgoing',
        time: new Date().toLocaleTimeString()
      }]);
    }
  };
  return (
    <div className="websocket-demo">
      <h2>WebSocket客户端演示</h2>
      {client && (
        <div className="status-panel">
          <WebSocketStatus client={client} />
          <WebSocketControls client={client} />
        </div>
      )}
      <div className="message-container">
        {messages.map((msg, index) => (
          <div key={index} className={`message ${msg.direction}`}>
            <div className="message-content">
              {msg.message}
            </div>
            <div className="message-time">{msg.time}</div>
          </div>
        ))}
      </div>
    </div>
  );
};
```


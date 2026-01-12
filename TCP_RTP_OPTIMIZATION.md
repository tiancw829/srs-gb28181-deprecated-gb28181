# GB28181 TCP RTP 数据处理优化说明

## 优化概述

本次优化借鉴了新版本SRS中GB28181 TCP RTP数据处理的实现方式，显著提升了TCP传输模式下的可靠性和性能。

## 主要改进

### 1. 采用RFC4571标准协议

**改进前：**
- 支持两种不兼容的封装格式（0x24标记和无标记）
- 格式识别逻辑复杂且容易出错
- 不符合标准协议规范

**改进后：**
```cpp
// RFC4571: RTP over TCP uses 2 bytes length prefix
uint16_t length = 0;
uint8_t lbuffer[2];
conn_->read_fully(lbuffer, sizeof(lbuffer), NULL);
length = ((uint16_t)lbuffer[0]) << 8 | (uint16_t)lbuffer[1];
```
- 统一使用RFC4571标准的2字节长度前缀
- 代码更简洁、可维护性更好
- 与标准协议兼容

### 2. 改进缓冲区管理

**改进前：**
```cpp
uint32_t left_data_len = 0;
// 频繁使用memmove操作
memmove(mbuffer, buf + index, left_data_len);
```

**改进后：**
```cpp
// 限制剩余字节数量，避免内存问题
uint32_t reserved = 0;
const uint32_t MAX_RESERVED = 128;

if (reserved > MAX_RESERVED) {
    srs_warn("gb28181: drop excessive reserved=%d bytes", reserved);
    reserved = 0;
}
```
- 添加保留字节数量上限
- 减少不必要的内存操作
- 防止内存累积导致的问题

### 3. 增强错误处理

**改进前：**
```cpp
if (packet_len > MAX_PACKAGE_SIZE) {
    srs_error("abnormal RTP packet length:%d", packet_len);
    return err;  // err仍是srs_success！
}
```

**改进后：**
```cpp
if (length > MAX_PACKAGE_SIZE) {
    srs_error("gb28181: abnormal RTP packet length=%d, closing connection", length);
    err = srs_error_new(ERROR_GB28181_PACKET_INVALID, "invalid packet length=%d", length);
    return err;  // 正确返回错误
}

// 记录详细的错误信息
srs_warn("gb28181: process packet error %s, reserved=%d, length=%d", 
         srs_error_desc(err).c_str(), reserved, length);
```
- 异常时正确返回错误码
- 添加详细的错误日志
- 提供更多上下文信息用于调试

### 4. 添加包大小告警

```cpp
// 对大包进行告警
if (length > 1500) {
    srs_warn("gb28181: large RTP packet length=%d from %s", length, ip.c_str());
}
```
- 及时发现异常大小的数据包
- 有助于问题诊断

### 5. 改进TCP和UDP处理分离

**改进前：**
```cpp
srs_error_t on_tcp_packet(...) {
    return on_udp_packet(...);  // 直接复用UDP逻辑
}
```

**改进后：**
```cpp
srs_error_t on_tcp_packet(...) {
    // TCP packet already de-framed by SrsGb28181Conn, directly process as RTP
    // Note: TCP framing is different from UDP, but RTP payload is the same
    return on_rtp_packet_jitter(...);
}
```
- 明确TCP和UDP的处理差异
- 添加清晰的注释说明
- 为未来扩展预留空间

### 6. 优化RTP包错误处理

**改进前：**
```cpp
if ((err = pkt->decode(&stream)) != srs_success) {
    srs_freep(pkt);
    srs_warn("gb28181 ps rtp: decode error");
    srs_freep(err);
    return srs_success;  // 丢弃错误信息
}
```

**改进后：**
```cpp
if ((err = pkt->decode(&stream)) != srs_success) {
    srs_freep(pkt);
    srs_warn("gb28181 ps rtp: decode error %s, peer=%s:%d, size=%d",
             srs_error_desc(err).c_str(), address_string, peer_port, nb_buf);
    srs_freep(err);
    return srs_success;
}
```
- 记录完整的错误描述
- 包含源地址和包大小信息
- 便于问题追踪

### 7. 新增错误码定义

```cpp
#define ERROR_GB28181_PACKET_INVALID        6021
#define ERROR_GB28181_PACKET_LENGTH         6022
#define ERROR_GB28181_RTP_DECODE            6023
```
- 为TCP RTP处理添加专用错误码
- 提供更精确的错误分类

## 性能提升

1. **减少内存操作**：优化缓冲区管理，减少memmove调用
2. **提前错误检测**：在读取完整数据包前进行长度校验
3. **资源清理**：确保异常情况下的内存正确释放

## 可靠性提升

1. **标准化协议**：使用RFC4571标准，提高互操作性
2. **完善错误处理**：异常时正确返回错误码并关闭连接
3. **防止内存累积**：限制保留字节数量
4. **详细日志**：记录完整的错误上下文

## 未来扩展方向

为了进一步提升可靠性，可以考虑以下改进：

### 1. RTP序列号校验
```cpp
// 检测丢包和乱序
uint16_t expected_seq = last_seq + 1;
if (pkt->sequence_number != expected_seq) {
    srs_warn("gb28181: RTP sequence gap, expected=%u, got=%u", 
             expected_seq, pkt->sequence_number);
}
```

### 2. 恢复机制
参考新版本SRS的`SrsRecoverablePsContext`实现：
- 进入恢复模式丢弃数据直到找到PS包头
- 限制最大恢复尝试次数
- 记录恢复统计信息

### 3. 连接状态管理
- 添加连接超时检测
- 实现心跳机制
- 优雅断开连接

## 测试建议

1. **单元测试**
   - 测试各种包长度（空包、正常包、大包、超大包）
   - 测试错误处理路径
   - 测试缓冲区边界条件

2. **集成测试**
   - 长时间稳定性测试
   - 网络异常场景测试
   - 多设备并发测试

3. **性能测试**
   - CPU使用率
   - 内存占用
   - 丢包率和延迟

## 兼容性说明

本次优化保持了与现有UDP处理的兼容性：
- UDP数据包处理流程不变
- RTP payload处理逻辑不变
- 仅优化了TCP framing层

## 参考资料

1. RFC4571 - Framing Real-time Transport Protocol (RTP) and RTP Control Protocol (RTCP) Packets over Connection-Oriented Transport
2. SRS新版本GB28181实现（trunk/src/app/srs_app_gb28181.cpp）
3. GB/T 28181-2016 公共安全视频监控联网系统信息传输、交换、控制技术要求

## 变更文件清单

- `trunk/src/app/srs_app_gb28181.cpp` - 核心TCP RTP处理逻辑
- `trunk/src/kernel/srs_kernel_error.hpp` - 错误码定义
- `TCP_RTP_OPTIMIZATION.md` - 本文档

## 第二轮修复（稳定性增强）

在代码审查后发现并修复了以下问题：

### 1. 内存泄漏修复
**问题：** `peer_sockaddr`使用`malloc`分配，但异常退出时不会被释放。

**修复：** 使用栈分配替代堆分配，确保自动释放。
```cpp
// 修复前
sockaddr_in *peer_sockaddr = (sockaddr_in*)malloc(addr_len);
// ... 异常时可能泄漏

// 修复后
sockaddr_in peer_sockaddr_storage;
sockaddr_in *peer_sockaddr = &peer_sockaddr_storage;
memset(peer_sockaddr, 0, sizeof(sockaddr_in));
// 栈变量自动释放，无泄漏风险
```

### 2. SrsAutoFree使用优化
**问题：** `SrsAutoFree`放在函数末尾，且手动`srs_freep`和自动释放混用，可能导致双重释放。

**修复：** 在`new`之后立即声明`SrsAutoFree`，移除手动释放。
```cpp
// 修复后
SrsPsRtpPacket *pkt = new SrsPsRtpPacket();
SrsAutoFree(SrsPsRtpPacket, pkt); // 立即接管所有权

if ((err = pkt->decode(&stream)) != srs_success) {
    // 不需要手动 srs_freep(pkt)
    return srs_success; // SrsAutoFree 自动释放
}
```

### 3. 缓冲区溢出保护
**问题：** 未检查`length`是否超过`mbuffer`容量。

**修复：** 添加双重检查。
```cpp
// 检查逻辑最大值
if (length > MAX_PACKAGE_SIZE) {
    return srs_error_new(ERROR_GB28181_PACKET_INVALID, ...);
}

// 检查实际缓冲区容量
if (length > SRS_RTSP_BUFFER) {
    return srs_error_new(ERROR_GB28181_PACKET_LENGTH, ...);
}
```

### 4. RTP最小包大小验证
**问题：** 未验证接收数据是否满足RTP头最小长度要求。

**修复：** 添加最小长度检查。
```cpp
// RTP header最少12字节
if (nb_buf < 12) {
    srs_warn("gb28181 ps rtp: packet too small %d bytes", nb_buf);
    return srs_success;
}
```

### 5. 移除死代码
**问题：** `reserved = 0`后又检查`reserved > MAX_RESERVED`，逻辑无效。

**修复：** 简化代码，移除无效逻辑。

### 6. 添加必要的头文件
**修复：** 添加`<cstring>`确保`memset`可用。

---

**优化日期：** 2026年1月12日  
**优化作者：** GitHub Copilot  
**参考版本：** SRS 6.0+ GB28181实现

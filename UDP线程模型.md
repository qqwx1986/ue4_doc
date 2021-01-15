# UDP线程模型分析
- 发送没有独立的线程，直接UIpConnection::LowLevelSend发送，LowLevelSend会调用FSocketBSD::SendTo，因为设置是nonblock，发送是无阻塞模式
- 接收 在UIpNetDriver::InitBase时，判定CVarNetIpNetDriverUseReceiveThread的值决定是否启用多线程来接收，默认是不启用，如果配置>0，那么调用 FRunnableThread::Create 创建接收线程<br>
  FRunnableThread::Create会判定FPlatformProcess::SupportsMultithreading()是否支持多线程模型，<br>
  这个的配置是命令行参数 -nothreading 未传入时调用，但是很多项目因为性能优化原因会打开-nothreading 这个参数，此处需要注意下
  
 ## 给自己埋个TODO
  -nothreading 开启后能提高哪些性能，又会降低哪些性能，怎么权衡，后续研究

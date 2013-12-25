### 测试环境
##### 1.1 服务端测试环境

<table class="table table-bordered table-striped table-condensed">
  <th width="15%">服务</th>
  <th width="30%">机器</th>
  <th width="55%">硬件配置</th>
  <tr>
    <td>GameServer
    </td>
    <td>pomelo3.server.163.org</td>
    <td>
      云主机<br>
      1CPU 8核心<br>
      CPU型号 GenuineIntel QEMU Virtual CPU version 1.1.2@2.0GHz<br>
      16G 内存<br>
      1网卡<br>
      linux/64位 OS<br>
    </td>
  </tr>
</table>


##### 1.2 客户端测试环境

<table class="table table-bordered table-striped table-condensed">
  <th width="15%">服务</th>
  <th width="30%">机器</th>
  <th width="55%">硬件配置</th>
  <tr>
    <td>Clients</td>
    <td>pomelo16~18.server.163.org</td>
    <td>
      云主机<br>
      1CPU 1核心<br>
      CPU型号 GenuineIntel Westmere E56xx/L56xx/X56xx (Nehalem-C)@2.6GHz<br>
      1G 内存<br>
      1网卡<br>
      linux/64位 OS<br>
    </td>
  </tr>
</table>

### 测试结果

1. `connector`和`echo`业务进程各`1个`.
2. `2个`客户端并发, 每隔`1ms`发起一次`request`请求, 每个客户端总计发送`1w`次, 服务器对每个`request`回复一个`200`. 
3. 服务器完成`2w`次请求的时间为`14.835s`, 平均`1348次/s`.
4. 服务器完成一次RPC调用的时间约为: `2~8ms`
5. 在服务器运行过程中:
 `connector`进程对CPU的占用为: 92%, 94%, 95%, 87%, 84%, 96%, 93%
      `echo`进程对CPU的占用为: 30%, 20%, 33%, 22%, 25%, 46%, 21%
6. 在客户端运行过程中:
   client进程对CPU的占用为: 18%, 24%, 25%, 40%, 16%, 49%, 39%


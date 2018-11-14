## 如果为syn_sent阶段
 - 如果包含syn && fin 根据是否drop_synfin配置判断是否抛弃
 - 如果包含 ack 则 ack < iss || ac > send_max的话，回复rst 并抛弃 （不会放弃链接）
 - 设置rcv_time 
 - 根据包的win 设置twin局部变量
 - （跳过ecn）
 - （不对option做解析）
 - （不开启ts）
 - 如果包换syn 则初始化send_win
 - 初始化recv_wim
 - 如果包含rst 和ack 两个标记 (ack的合法性已经在之前验过了)，则直接断开连接。
 - 如果包含rst 则抛弃（没有ack的话 并不会断开链接）
 - 如果没有syn， 抛弃
 - 初始化irs 为包的th_seq (到了这里数据报必然会有syn 标记，可能会有 ack 标记，fin标记必然为0)
 - (tp)->rcv_adv = (tp)->rcv_nxt = (tp)->irs + 1
 - 如果包含ack 则为单方向链接，如果包含rcv_scale 则重新初始化rcv_win
   - snd_una ++ 来响应对端ack
   - 不考虑delay ack
   - 不考虑ece
   - 初始化t_starttime 为当前时间
   - 如果t_flags 包含TF_NEEDFIN 表示用户已经关闭了该链接 则直接进入TCPS_FIN_WAIT_1阶段
   - 否则进入 TCPS_ESTABLISHED 阶段同时初始化congestion control， 并激活keepalive 定时器
 - 否则双端同时打开连接
   - 将t_flags 标记为(TF_ACKNOW | TF_NEEDSYN)
   - 禁用重传定时器
   - 进入TCPS_SYN_RECEIVED阶段
 - 将包的seq +1 用来指定正确的数据seq
 - 如果包的数据 size > rcv_win 则抛弃不被接受的部分， 并将fin强制标记为0
 - 设置跟新该snd_win 的包的序号为 snd_wl1 = seq -1
 - 将紧急数据的序号标记为 rcv_up = seq
 - 如果数据包含ack(syn && ack)，这里很可能我们已经发送了数据(TFO?)
   - 进入普通处理ack逻辑 process_ACK
 - 否则进入 step 6 （fuking stepxxxx)

#### 普通处理ack逻辑 process_ACK
 - 计算真实被acked 字节数 acked
 - 如果我们有一次重传并且ack 到来的时间 < RTT /2， 认为是伪重传，提示congestion 。
 - 重新估算rtt
 - 如果 包ack == snd_max取消重传定时器，否则启用重传定时器
 - 如果acked数据为0 则（如果从syn_sent阶段流转过来 则直接进入step6）
 - 提示congestion 收到数据（此时包含数据）
 - 释放对应的acked 字节空间
 - 纠正snd_una 如果snd_una 回环的话， 并判断是否可以退出recovery 状态
 - 将snd_una 设置为包的ack
 - 重置snd_nxt 

#### step 6
 - 纠正snd_wnd，并且如果需要纠正的话讲needoutput 置为1
 - 不考虑urg
 
#### dodata
 - 不考虑TFO
 - 如果needoutput 被置位 或者TF_ACKNOW被置位则调用tfb_tcp_output(tcp_output)发送数据
 
 ### 总结
  在syn_sent阶段， 如果收到seg 包含ack则判断ack是否有效，如果无效则抛弃。如果只有rst 则直接抛弃并返回，如果有rst + ack 且ack 有效则抛弃链接。到了这里只会有syn 或者syn+ack。如果是syn 则为同时创建请求


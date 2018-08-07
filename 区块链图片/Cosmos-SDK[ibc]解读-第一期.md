## 1 IBC是什么
ibc是一种创建了不同的区块链间代币转移的机制，可以在不同的链间提供消息传递。如果把区块链比作是一种微服务，那么ibc就像是连接各个微服务使用的amqp协议RabbitMq的一种实现。在发送IBC数据包时，把数据包以队列的形式存储(merkle树)，并对每个发送和接受的数据包进行有效的安全验证，以保证消息的可靠，按序传递。并能够以异步的方式处理响应数据，整个调用过程类似于异步的rpc调用过程。

## 2 IBC数据包结构化定义
为了更好的说明ibc数据包的结构，下面只描述的一个ibc发送的数据包

```golang

type Packet struct {
    Payload   Payload
    SrcChain  string
    DestChain string
}

type Payload interface {
    Type() string
    ValidateBasic() sdk.Error
}

// Implements Payload
type SendPayload struct {
    SrcAddr  sdk.Address
    DestAddr sdk.Address
    Coins    sdk.Coins
}

// Implements sdk.Msg
type IBCSendMsg struct {
    DestChain string
    SendPayload
}

```

## 3 IBC数据包存储

   * KVstore，每个消息都有一个唯一的，确定的和可预测的key

     ```golang

       // Stores an outgoing IBC packet under "egress/chain_id/index".
     	func EgressKey(destChain string, index int64) []byte {
     		return []byte(fmt.Sprintf("egress/%s/%d", destChain, index))
     	}
     	
     	// Stores the number of outgoing IBC packets under "egress/index".
     	func EgressLengthKey(destChain string) []byte {
     		return []byte(fmt.Sprintf("egress/%s", destChain))
     	}
     	
     	// Stores the sequence number of incoming IBC packet under "ingress/index".
     	func IngressSequenceKey(srcChain string) []byte {
     		return []byte(fmt.Sprintf("ingress/%s", srcChain))
     	} 
     ```

* 双队列模型(ibc::send ，ibc::receipt )
  
     每个基于cosmos-sdk开发的区块链，ibc消息的存储都是由两个使用KVstore实现的队列，即发送队列和接受队列。(队列数量 = 2 * 与之连接的zone数量)。
     
     发送队列主要用于存储发往其他zone的packet消息，每一个消息都有唯一的标识sequence用于防重攻击(接收链验证sequence)
     
     接受队列主要用于存储接收来自其他链的响应结果。
     
 * 支持模块化写入权限 


## 4 IBC消息传递

ibc消息在传送过程中，必须要确保以下问题:

   * 顺序执行
   * 消息不可篡改(merkle证明) 
   * 正确路由(Relay进程)

   ![Successful Transaction](https://raw.githubusercontent.com/zhiqiang-bianjie/image/master/ibc/Receipts.png)

   ![Rejected Transaction](https://github.com/zhiqiang-bianjie/image/blob/master/ibc/ReceiptError.png?raw=true)


## 5 IBC消息堆积

![Successful Transaction](https://github.com/zhiqiang-bianjie/image/blob/master/ibc/CleanUp.png?raw=true) 

## 6 BFT容错处理

* 共识层面(ibc可检测,分叉处理:由于队列和关联状态必须映射到一个远程链，因此IBC协议不能选择遵循两条链)
* 应用层面(ibc不可检测，由应用层的审计处理)

## 7 开发阶段

MVP1,MVP2,MVP3

## 8 代码解读
   * 8.1 以下是发送一个IBC的交易流程(不包含响应阶段)
      ![Ibc tx](https://github.com/zhiqiang-bianjie/image/blob/master/ibc/IBC_Send.png?raw=true)
   * 8.2 mgs的hander注册流程
      ![Ibc reg tx](https://github.com/zhiqiang-bianjie/image/blob/master/ibc/hander_reg.png?raw=true)
   * 8.3 ibcSend发送流程
      ![Ibc send tx](https://github.com/zhiqiang-bianjie/image/blob/master/ibc/ibcsend.png?raw=true)
   * 8.4 Ibc relay处理流程

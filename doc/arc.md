# TopMain

TopMain 是RTL的顶层,用于生成verilog
```
object TopMain extends App {
  // Pass protocol parameter from sbt command line interface
  val protocol = args(0)
  assert(Seq("AXI4", "AXI4-Lite", "AXI4-Stream", "Simple").contains(protocol))

  // Configs
  val network_configs = GetImplicitNetworkConfigs(protocol)

  // Emit Verilog RTL
  protocol match {
    case "AXI4" | "AXI4-Lite" =>
      (new chisel3.stage.ChiselStage).emitVerilog(new NetworkAXI4Wrapper()(network_configs), args)
    case "AXI4-Stream" =>
      (new chisel3.stage.ChiselStage).emitVerilog(new NetworkAXI4StreamWrapper()(network_configs), args)
    case "Simple" => (new chisel3.stage.ChiselStage).emitVerilog(new NetworkSimpleWrapper()(network_configs), args)
  }
}
```

* val protocol = args(0) -> protocol 来自于Makefile
```
PROTOCOL ?= AXI4

all: $(CHISEL_SRC)
	@mkdir -p $(BUILD_DIR)
	sbt "run $(PROTOCOL) -td $(BUILD_DIR)"
```
* network_configs 包含所有参数
返回 NetworkConfigs = ConnectConfig + LibraryConfig + AXI4WrapperConfig/
AXI4LiteWrapperConfig/AXI4StreamWrapperConfig/SimpleWrapperConfig

* Config 基类 三个功能函数，类型转换
```
class Configs(configs: Map[String, Any]) {
  def getBoolean(field: String, default: Boolean = false) = configs.getOrElse(field, default).asInstanceOf[Boolean]
  def getInt(field:     String, default: Int     = 0)     = configs.getOrElse(field, default).asInstanceOf[Int]
  def getString(field:  String, default: String  = "")    = configs.getOrElse(field, default).asInstanceOf[String]
}
```
* NetworkConfigs 拆分成 ConnectConfig + LibraryConfig + AXI4WrapperConfig/
AXI4LiteWrapperConfig/AXI4StreamWrapperConfig/SimpleWrapperConfig负责不同组的参数

# NetworkSimpleWrapper
* Config
NUM_USER_SEND_PORTS = 4
NUM_USER_RECV_PORTS = 4
"PACKET_WIDTH"       -> 72
* 三个模块 mkNetwork send 和 recv 
* mkNetwork 是一个verilog模块，功能未知 recv :output  send :input
* 每个send channel ::: io -> FlitSerializer -> FlitFlowControlSend -> mkNetwork
* 每个recv channel ::: mkNetwork -> FlitFlowControlRecv -> FlitDeserializer -> io

* NetworkSimpleWrapper io: send recv 
* send -> serializer.io.in_packet -> flow_control_send.io.flit(0) -> network.io.send(i)
* network.io.recv(i) -> flow_control_recv.io.recv -> deserializer.io.in_flit -> io.recv(i)

## in packet  ::: device ---> send packet ---> NI ---> network
## FlitSerializer
* packet -> out_flit trans
* 从大包到小包的转变
* in_packet -> fifo -> meta_reg ------ .....  ----> out_flit
                    -> data_reg

                                            len
* state :  s_idle --packet.fire--> s_send -----------> 
            ^                         |      |     |
            |                         --------     | len > LEN
            ----------------------------------------

## FlitFlowControlSend
    outside           interface 
* packet --> flit ---> io.flit ---> fifo --> hub(FlitHubNTo1) --> send_interface(FlitSendInterface) --> 
                       flic 三个通道，目前只用到通道0

* FIFO::InPortFIFO
device_flit --> fifo in --> 
fifo out --> network_flit (在有过 io.network_credit.fire 的情况下)
io.network_credit.fire ??? 

* FlitHubNTo1
device_flit --> arbiter --> network_flit
io.network_credit --> vc --> io.device_credit(i)  选择传入哪个通道

* FlitSendInterface
hub.io.network_flit   ---->     send_interface | put_flit -----> send | ---> io.send  ---> network.send
hub.io.network_credit <----                    |                      |


## credit flow
  保证send 送入到相应的fifo 
  network.send(include credit) -> send_interface(FlitSendInterface) -> hub(FlitHubNTo1)  ->   fifo(InPortFIFO) (fifo check)
                                     check credit N to 1 fifo
  

+--------+            +------------+             +-------------+            +---------+
| Device | <--AXI4--> | AXI4Bridge | <--flits--> | InPortFIFO  | <--flits-->|         |
+--------+            +------------+             +-------------+            |         |
                            ^                                               | Network |
                            |                    +-------------+            |         |
                            +-----------flits--> | OutPortFIFO | <--flits-->|         |
                                                 +-------------+            +---------+



* data flow
* send
+---------+
|         |
|         |   flit    +--------------------+            
| Network |<----------| FlitSendInterface  | <--------
|         |---------->|                    |          
|         |  credit   +--------------------+          
+---------+
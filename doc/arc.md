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
* mkNetwork 是一个verilog模块，功能未知
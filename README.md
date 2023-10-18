# ROS2基础

- [ ] 掌握ROS2中节点概念即相关指令
- [ ] 掌握ROS2中工作空间的相关概念
- [ ] 掌握colcon编译指令来构建一个包
- [ ] 学会通过命令行安装软件和ROS2相关的包
- [x] 了解面向对象编程OOP的基本概念
- [ ] 学会使用Python编写一个ROS2的节点
- [ ] 学会使用C++编写一个ROS2的节点

## 1. ROS2节点

### 1-1什么是节点

ROS2中的每一个节点也是只负责一个**单独**的模块化的功能，每个节点都是一个封装了特定功能或任务的独立实体。**节点通常是一个类的实例**

### 1-2节点的职责

- **发布消息**：向特定的话题**发送**数据。
- **订阅消息**：从特定的话题**接收**数据。
- **提供服务**：**响应**其他节点的请求。
- **调用服务**：向其他节点发送**请求并等待**响应。
- **行为**：一种更复杂的服务模式，通常用于需要多步骤和可中断操作的情况。

### 1-3节点之间如何交互？

- [话题-topics](#发布和订阅消息(topics))

- [服务-services](#服务和客户端(services))

- 动作-Action

- 参数-parameters

  ![Nodes-TopicandService](https://fishros.com/d2lros2foxy/chapt3/3.1ROS2%E8%8A%82%E7%82%B9%E4%BB%8B%E7%BB%8D/imgs/Nodes-TopicandService.gif)

### 1-4话题/服务(Topics/Services)
#### 1-4-1发布和订阅消息(topics)

在节点中，你通常会有一个或多个发布器（Publisher）和订阅器（Subscriber）。发布器用于向特定的话题发送消息，而订阅器则用于从特定的话题接收消息。

是一种异步通信方式。发布者发布消息到某个话题，所有订阅了该话题的订阅者都会收到这个消息。但发布者不知道（也不关心）谁会接收这个消息。

##### 发布消息


1. **创建发布器**：在节点的构造函数中，我们使用 <u>`create_publisher`</u> 方法创建一个发布器。这个方法需要两个参数：话题名（"my_topic"）和队列大小（10）。
   
   ```cpp
   publisher_ = this->create_publisher<std_msgs::msg::String>("my_topic", 10);
   ```
   
2. **创建定时器**：定时器用于周期性地触发消息发布事件。在这个例子中，我们设置了一个每 500 毫秒触发一次的定时器。

   ```cpp
   timer_ = this->create_wall_timer(
     std::chrono::milliseconds(500),
     std::bind(&MyPublisherNode::publishMessage, this)
   );
   ```

3. **发布消息**：在 publishMessage 函数中，我们创建一个 std_msgs::msg::String 类型的消息，并通过 publish 方法发送它。

   ```cpp
   auto message = std_msgs::msg::String();
   message.data = "Hello, ROS 2!";
   publisher_->publish(message);
   ```

##### 订阅消息

1. **创建订阅器**：与发布器类似，我们在节点的构造函数中使用 create_subscription 方法创建一个订阅器。这个方法同样需要话题名、队列大小和一个回调函数作为参数。
   
   ```cpp
   subscriber_ = this->create_subscription<std_msgs::msg::String>(
     "my_topic",
     10,
     std::bind(&MySubscriberNode::messageCallback, this, std::placeholders::_1)
   );
   
   ```
   
2. **处理消息**：每当从 "my_topic" 话题收到新消息时，messageCallback 函数就会被触发。在这个函数中，我们简单地输出接收到的消息。

   ```cpp
   void messageCallback(const std_msgs::msg::String::SharedPtr msg)
   {
     RCLCPP_INFO(this->get_logger(), "Received message: %s", msg->data.c_str());
   }
   
   ```

#### 1-4-2服务和客户端(services)

服务（Services）和客户端（Clients）提供了一种同步通信机制，使一个节点能向另一个节点发送请求并接收响应。在 ROS 2 中，服务和客户端**可以是同步的也可以是异步**的。在服务和客户端的交互中，通常有**一个**“服务提供者”（Service Provider）和一个或**多个**“服务用户”（Service Client）。

是更像远程过程调用（RPC）的通信方式，通常是同步的（也可以是异步的）。客户端发送一个**请求**到服务，服务处理这个请求并返回一个**响应**给客户端。

##### 创建服务

1. **定义服务**：在节点的构造函数中，我们使用 <u>`create_service`</u> 方法定义一个服务。这个方法需要两个参数：服务名（"add_two_ints"）和一个回调函数。

   ```
   cppCopy codeservice_ = this->create_service<example_interfaces::srv::AddTwoInts>(
     "add_two_ints",
     std::bind(&MyServiceNode::handleService, this, std::placeholders::_1, std::placeholders::_2)
   );
   ```

2. **处理请求**：在定义的回调函数 `handleService` 中，我们处理客户端发送的请求并返回一个响应。

   ```
   cppCopy codevoid handleService(
     const std::shared_ptr<example_interfaces::srv::AddTwoInts::Request> request,
     std::shared_ptr<example_interfaces::srv::AddTwoInts::Response> response)
   {
     response->sum = request->a + request->b;
   }
   ```

##### 创建客户端

1. **定义客户端**：在节点的构造函数中，我们使用 <u>`create_client`</u> 方法定义一个客户端。这个方法需要一个参数：服务名（"add_two_ints"）。

   ```
   cppCopy code
   client_ = this->create_client<example_interfaces::srv::AddTwoInts>("add_two_ints");
   ```

2. **发送请求**：在节点的某个函数（比如 `send_request`）中，我们创建一个请求并使用客户端的 `async_send_request` 方法发送它。

   ```
   cppCopy codevoid send_request(int a, int b)
   {
     auto request = std::make_shared<example_interfaces::srv::AddTwoInts::Request>();
     request->a = a;
     request->b = b;
     auto result = client_->async_send_request(request);
     // Handle the result...
   }
   ```

#### 1-4-3(Pub/Sub)与**(Service/Client)**区别

##### 通信模式：

- *发布/订阅（Pub/Sub）:* 是一种异步通信方式。发布者发布消息到某个话题，所有订阅了该话题的订阅者都会收到这个消息。但发布者不知道（也不关心）谁会接收这个消息。
- *服务/客户端（Service/Client）:* 是更像远程过程调用（RPC）的通信方式，通常是同步的（也可以是异步的）。客户端发送一个请求到服务，服务处理这个请求并返回一个响应给客户端。

##### 数据流：

- *发布/订阅（Pub/Sub）:* 数据流是**单向**的。发布者发布消息，订阅者接收消息。
- *服务/客户端（Service/Client）:* 数据流是**双向**的。客户端发送*请求*，服务接收请求并发送*响应*。

##### 实时性和可靠性：

- *发布/订阅（Pub/Sub）*: 通常用于需要高频或实时更新的情况，如传感器数据的**实时流**。
- *服务/客户端（Service/Client）*: 通常用于**请求/响应式**的*交互*，这样的交互通常不需要高频率，但需要更高的**可靠性**。

##### 用途：

- *发布/订阅（Pub/Sub）*: 更适用于**数据的广播**，例如，一个传感器节点发布温度数据，多个其他节点（如数据记录节点、监控节点等）可能需要这些数据。
- *服务/客户端（Service/Client）*: 更适用于**一对一**的交互，例如，一个节点请求另一个节点进行某种计算并返回结果。

##### 复杂性：

- *发布/订阅（Pub/Sub）*: 相对简单，因为你不需要关心谁会接收或处理消息。

- *服务/客户端（Service/Client）*: 可能更复杂一些，因为你需要处理请求的接收、处理和响应。

#### 1-4-4完全限定名

  如 *rclcpp::Subscription<std_msgs::msg::String>::SharedPtr subscriber_*

  1. **命名空间（Namespace）**: 诸如 `rclcpp` 或 `std_msgs` 这样的命名空间通常用于组织代码。当你看到 `::` 符号，可以认为你正在进入一个特定的“区域”或“分类”，在这里你会找到特定的类、函数或变量。
     - 例如：`rclcpp::` 表示我们正在使用 ROS 2 的 C++ 客户端库。
  2. **类名（Class）**: `Subscription` 和 `String` 这样的单词通常是类名。这告诉你这段代码与何种对象类型有关。
     - 例如：`Subscription` 表示这是一个用于订阅话题的对象。
  3. **模板参数（Template Parameters）**: 尖括号 `< >` 中的内容是模板参数，它们用于指定一个通用类的具体类型。
     - 例如：`<std_msgs::msg::String>` 表示这个 `Subscription` 是专门用于订阅 `String` 类型消息的。
  4. **类型修饰符（Type Modifiers）**: 诸如 `SharedPtr` 这样的单词通常用于指定变量的额外属性或行为。
     - 例如：`SharedPtr` 表示这是一个智能指针，它会自动管理内存。
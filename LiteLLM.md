##  使用 LiteLLM 的动机与优势
 1. 多模型支持是现实需求 

企业级应用通常需要同时使用多类模型，例如通用 LLM（GPT‑4／Claude），行业定制微调模型，以及视觉/多模态模型（如图像生成、音频等）。在存在多类模型的情况下，急需一个管理平台对在这些模型进行管控。LiteLLM 能以统一接口支持 100+ 模型供应商（OpenAI、Anthropic、Bedrock、Azure、Hugging Face、本地部署等），可以在不改应用代码的情况下灵活切换或混合调用模型。  

 2. 有效管控宝贵 GPU 与外部 API 调用

LiteLLM 可作为统一代理，支持 **Virtual Keys、团队预算、调用速率限制**（Rate Limit）、Token 限额等策略，不仅能追踪费用与调用，还能避免多个团队或外部请求无序共享 GPU 或 API 配额。对于拥有多个模型使用者的组织而言，这极其重要。

 3. 支持对特殊模型参数的精细控制。

部分模型需要在调用时调整特殊参数或参数组合，如混合模型 qwen3可以通过参数调整切换**推理模式**和**一般模式**。通过 LiteLLM 的 `Model Mappings` 与 alias 功能，可以根据模型访问策略或 Key 限额自动选择不同参数版本，无需更改客户端应用。  

 4. 测试与展示阶段需要资源隔离策略  

在压力测试、用户演示阶段，必须保证特定模型占用的计算资源（如 GPU）是独占或隔离的，避免因资源争抢或噪声导致不稳定。LiteLLM 支持为不同场景定义独立模型部署、独立 Endpoint，并通过 Model Health、Retry 策略与限额机制保障隔离与稳定性。

## LiteLLM 是什么

**LiteLLM 是一个开源框架和代理服务，旨在建立统一的 LLM 接入方式，并集成调度、预算、日志、Guardrails 等企业功能。**它解决了多模型、多提供商、多团队使用的复杂问题，让企业能够快速、安全、高效地管理 LLM 调用流程。

## LiteLLM 的核心配置文件：

###  一、config.yaml：LiteLLM 的中央配置文件

#### 定义与作用

- `config.yaml` 是 LiteLLM Proxy Server 的 **命令式配置源**，决定代理服务启动所需的所有模型、路由策略、限额、安全设置等核心信息。
- 来自官方文档的定义：它是 **LiteLLM Proxy 的 central configuration hub**，包括模型调用、认证、fallback、budget、retry 等多方面设置。

#### 结构与关键模块

- **environment_variables**
   定义环境变量，如 `OPENAI_API_KEY` 等，可通过 `os.environ/VAR_NAME` 在其他字段中引用。

- **model_list**（必需）
   列出每个 Public Model 的 alias 和后端调用配置，包括：

  ```cpp
  yaml
  
  model_list:
    - model_name: gpt-4o
      litellm_params:
        model: azure/gpt-4o-eu
        api_base: ...
        api_key: os.environ/OPENAI_API_KEY
        rpm: 6
      model_info:
        version: "1"
  ```

- **general_settings**
   设置主 Key（master_key）、数据库、日志保留、回调配置（如 Slack 通知）等服务级参数。

- **litellm_settings**
   SDK 行为设置，例如是否启用 verbose、JSON logs、默认超时等。

- **router_settings**
   路由策略与容错配置：包括 fallback、retry_policy、routing_strategy、Redis 协调、多模型分组支持、tag-based routing 等。

- **callback_settings**（可选）

   配置日志输出 plugins，如 Langfuse、Athina、Prometheus 回调等。

#### 二、docker-compose.yaml：容器编排脚本示例

#### 定义与作用

- `docker-compose.yaml` 是用于将 Proxy 与其他服务（如数据库、Open-WebUI）一起部署的容器编排脚本。
- 它描述多个服务如何协同运行，包括 LiteLLM Proxy 镜像、环境变量挂载、端口暴露、 volume 卷挂载等。

#### 常见组合结构

根据多个教程示例：

```cpp
yaml

services:
  litellm:
    image: ghcr.io/berriai/litellm:main-latest
    restart: unless-stopped
    volumes:
      - ./litellm_config.yaml:/litellm_config.yaml
    environment:
      LITELLM_MASTER_KEY: ${LITELLM_MASTER_KEY}
      OPENAI_API_KEY: ${OPENAI_API_KEY}
      ANTHROPIC_API_KEY: ${ANTHROPIC_API_KEY}
    command:
      - "--config=/litellm_config.yaml"
      - "--detailed_debug"
    ports:
      - "4000:4000"
  webui:
    image: ghcr.io/open-webui/open-webui:main
    ports:
      - "8080:8080"
    environment:
      - OPENAI_API_BASE_URL=http://litellm:4000/v1
```

然后通过 `docker-compose up -d` 启动整个服务栈。

#### 可选扩展：

- 添加 PostgreSQL service，用于存储日志、Key、配置等；
- 添加 Redis service（适合分布式部署）；
- 通过 `.env` 文件安全管理 API Key 和 master key 等敏感信息

## LiteLLM DashBoard
### 一、Virtual Keys
#### Virtual Keys页面说明
标准大模型 API Key 通常对一个模型或服务商有效，缺乏细粒度控制能力。而Virtual Key 是 LiteLLM Proxy 使用的 API Key，每个 Virtual Key 与一个 **Internal User** 或 **Team** 关联， 可对 Key 所对应的用户或团队进行权限、预算与日志分配 。**可自定义访问行为**，包括允许使用的模型列表、别名映射（alias）、预算限制（max_budget）、速率限制（requests/tokens 限额）、Key 到期时间等 。

 	在 **LiteLLM Dashboard** 中点击 **Virtual Keys** 后，可以进入管理界面，在此页面你可以：  

+ 浏览当前已创建的所有 Virtual Key；
+ 看到每个 Key 的基础信息，例如名称、关联的用户或团队、权限模型、使用状态等；
+ 查看每个 Key 的当前消费（Spend）、预算上限（Max Budget），以及所允许访问的模型列表；
+ 借助界面右上角的 **Filters** 按钮，对 Key 进行分类筛选，比如按预算状态、模型权限、过期时间等维度进行过滤。
+ 借助界面右上角的** + Create New Key ** 按钮，创建一个新的Virtual Keys 。

![](https://cdn.nlark.com/yuque/0/2025/png/48196500/1753604175759-b6240593-de72-44ca-b936-55cb2eb316ab.png)

#### **Create New Key**页面字段说明
点击页面左上方的**“+ Create New Key”**按钮，弹出创建 Key 的配置弹窗；表单中包含以下字段，每个字段的含义说明如下：![](https://cdn.nlark.com/yuque/0/2025/png/48196500/1753604229250-21558cc8-cc73-4b35-8a47-7cc31b6f5566.png)

+ **<font style="background-color:rgb(255,255,255);">Owned By：</font>**<font style="background-color:rgb(255,255,255);"> 用于指定该 API Key 属于哪个主体，通常可选择：</font>**自己（Yourself）**<font style="background-color:rgb(255,255,255);">、</font>**Service Account**<font style="background-color:rgb(255,255,255);">，也可以指定 </font>**Another User**<font style="background-color:rgb(255,255,255);">（即为其他用户创建 Key）。选择 Service Account 时，该 Key 不依附于个人账户，适用于生产项目或长期服务帐号使用，且不会随个人用户被删除而丢失。  </font>
+ **<font style="background-color:rgb(255,255,255);">Team：</font>**指定该 Key 所属的团队。 团队关联定义了可访问的模型范围以及预算限制等管理策略。  
+ **<font style="background-color:rgb(255,255,255);">Key Name：</font>**<font style="background-color:rgb(255,255,255);"> 自定义可识别该 Key 的名称／别名。有助于日后进行管理、审计与辨识。  </font>
+ **<font style="background-color:rgb(255,255,255);">Models：</font>**<font style="background-color:rgb(255,255,255);"> 可选择 Key 可使用的模型范围。可以选择接入团队下的全部模型（All Team Models），也可手动选定一个或多个模型 。</font>
+ <font style="background-color:rgb(255,255,255);"> </font>**Optional Settings（可选设置）：**
    - <font style="background-color:rgb(255,255,255);">Max Budget（最大预算，单位 USD 或其他货币）；</font>
    - <font style="background-color:rgb(255,255,255);">Budget Duration（预算重置周期，如每天、每月）；</font>
    - **<font style="background-color:rgb(255,255,255);">Requests per Minute / Tokens per Minute（每分钟最大请求数或 token 限制）；</font>**
    - **<font style="background-color:rgb(255,255,255);">Expiry Duration（Key 失效时间／有效期）</font>**<font style="background-color:rgb(255,255,255);">，通过设置，可以控制Virtual Keys的过期时间</font>
    - <font style="background-color:rgb(255,255,255);">Guardrail、Tag Routing、Logging Callback 等高级策略配置</font>

#### **Create New Key**
<font style="background-color:rgb(255,255,255);">填写并配置完成后，点击 </font>**Create Key**即可生成一个新的 Virtual Key。**此 Key 是唯一且一次性的，只在生成时显示一次，请务必妥善保存该值**——之后无法再次查看。 

![](https://cdn.nlark.com/yuque/0/2025/png/48196500/1753607942448-036af674-98e0-4a4d-a263-1f957df58c44.png)![](https://cdn.nlark.com/yuque/0/2025/png/48196500/1753607980202-9767542b-b9ad-43e5-8583-8b16754795cb.png)

### 二、TEST KEY
#### **Test Key**页面说明
**Test Key** 是 LiteLLM Dashboard 中 Virtual Key 的测试辅助入口，用于验证 Key 的配置在 Playground 中是否可正常调用模型；页面中包含以下字段，每个字段的含义说明如下：

 ![](https://cdn.nlark.com/yuque/0/2025/png/48196500/1753608043015-1496e1de-d187-45eb-a44f-c9279e93f2a5.png)

+ **API Key Source有两个选项：**
    - **Current UI Session** 指的是在 Dashboard UI 中以当前登录用户的身份直接进行模型调用测试的方式。这并不是一把实际的 Virtual Key，而是一种临时授权机制，UI 会自动把当前登录的账户或 session 用作调用凭证。  
    - 选择**Virtual Key**，便可将生成的 Key 粘贴至 Playground；
+ **Endpoint Type** 字段通常用于表示这个虚拟 Key 所允许访问的后端调用接口类型。
+ **Tags **用于在请求中携带标签，帮助 LiteLLM Router 根据模型或团队关联的标签进行路由控制（Tag‑Based Routing）。
+ **Vector Store（向量存储／知识库）**  将 Virtual Key 与 Vector Stores（知识库）绑定，允许使用 RAG（Retrieval‑Augmented Generation，即检索增强生成）工作流。 
+ **Guardrails（守护策略）**  是 LiteLLM 提供的一类策略插件机制，用于在模型调用前、中、或后执行 PII 过滤、安全审计、内容策略检查等操作。

#### **测试创建的Virtual Key**
选择**Virtual Key，**在 UI 界面中粘贴生成的 Key，并选择访问模型，便可以进行实际的请求测试操作（如与 Web UI 的即时交互）**。**

![](https://cdn.nlark.com/yuque/0/2025/png/48196500/1753608070441-57d9677e-ae9c-4e06-8d82-a8b2a5882c42.png)

同时在测试的过程中，我们还可以切换当前Virtual Key下的不同模型服务。比如从glm-l1切换到glm4v-9b。

![](https://cdn.nlark.com/yuque/0/2025/png/48196500/1753608092213-48ac71b2-3656-4058-8530-f69e31f05d38.png)

### 三、Models + Endpoints  
此页面用于管理在 LiteLLM Proxy 中定义和控制的模型部署与端点配置，是管理接口与模型访问权限的核心模块。

#### ALL models
ALL models页面显示 Proxy 已加载的所有模型部署实例，包括 Model ID、 Public Model Name   （公开模型名称）  、提供者(provider)、 LiteLLM Model Name （内部 LiteLLM 模型名称） 、成本定价等信息。  ![](https://cdn.nlark.com/yuque/0/2025/png/48196500/1753608134149-79370a0c-7646-4a04-b5c3-41885a620873.png)

+  Public Model Name：这是你在 LiteLLM 前端 Dashboard 或 API 客户端中使用的用户可见模型名称，也就是调用时使用的名字。  
+  LiteLLM Model Name：实际发送给后端 LLM 提供商的模型标识，包括 provider 前缀、部署名、版本号等。  
+  LiteLLM Proxy 会将客户端请求中的 `model = Public Model Name` 映射为此实际的 provider 模型名称执行调用。  

#### Add new model
在 **LiteLLM Dashboard** 的 “Models + Endpoints” 页面，点击 **“Add New Model”** 后，即可通过 UI 向 LiteLLM Proxy 添加新的模型部署。 并且无需修改 config.yaml 或重启服务，可以通过 Admin UI 直接添加模型部署。

但还是优先推荐使用 config.yaml 配置模型 ：

+ UI 添加模型是一种临时操作，**只保存在数据库（若启用）或内存中**，一旦重启或配置迁移可能丢失。  
+ 而写入 config.yaml 的模型配置，是 Proxy 启动时的“真 source of truth”，确保即便重启、升级、或有人误删 UI 中模型，仍能稳定加载。  

下面是该操作中各个字段的详解，配合官方文档说明：

![](https://cdn.nlark.com/yuque/0/2025/png/48196500/1753608202880-88756822-3e2d-4684-9b83-c00c52ce98ac.png)

+  **Provider：** 
    - 指定模型所属的服务商，如 OpenAI, Azure OpenAI, Anthropic, Bedrock 等。   
    - 后续字段将依据所选 Provider 显示对应的认证与调用配置项 。
+ **LiteLLM Model Name(s) ：**
    -  实际在 `litellm.completion()` 或 `litellm.embedding()` 中调用时使用的内部模型标识，例如 `"openai/gpt-3.5-turbo"` 或 `"anthropic/claude-3"`.  
    - 支持为一个 Public 名称绑定多个 LiteLLM Model 名称，实现同一调用路由多个后端部署以做负载均衡或 fallback。  
+ **Model Mappings ：**
    - 定义 Public Model Name 与 LiteLLM Model Name 之间的映射关系。
    - 当多个 LiteLLM Model Name 映射到同一个 Public Model Name 时，系统会按一定策略进行负载均衡路由调用。  
+ **Mode（可选）  :**
    - 用于指定模型类型，如 `chat`、`completion`、`embedding` 等。
    - Proxy 将根据对应 mode 的方式对该模型进行健康检查与调用健康性管理。
+ **Credentials（认证信息）：** 
    - Existing Credentials：从已有 Provider 凭证中选择（如之前添加的 OpenAI Key、Azure Key 等）。
    - 或手动输入新的 Provider 凭证，包括 API Key、Organization ID、API Base URL 等。
+ **Additional Model Info Settings**
    - Team：只有属于该 Team 的 Virtual Key 才有权限调用此模型；
    - Model Access Group：通过访问组控制哪些 Key 有权限使用该模型，适用于动态扩展用户权限场景。

同样，可以在yaml配置文件新增一个模型

```cpp
model_name: gpt-3.5-turbo  # Public 名称
litellm_params:
  model: openai/gpt-3.5-turbo
  api_key: os.environ/OPENAI_API_KEY
  api_base: https://api.openai.com
  mode: "chat"
model_info:
  access_groups: ["beta-testers"]
  team_id: "your-team-id"
```

在 UI 中则是：

+ Provider 选择 “OpenAI”；
+ LiteLLM Model Name 输入 `openai/gpt-3.5-turbo`；
+ Mode 填 `chat`；
+ 从 Existing Credentials 选择对应的 Key；
+ 指定 Team 或 Access Group 管理权限。

#### LLM Credentials
**LLM Credentials **是 LiteLLM 的凭证管理中心 ，你可以添加、删除、编辑或复用不同 Provider 的凭证 ， 添加凭证后可供后续添加模型时使用，提升配置效率与安全性。 

<font style="color:rgb(28, 30, 33);">①Models -> LLM Credentials -> Add Credential</font>

![](https://cdn.nlark.com/yuque/0/2025/png/48196500/1753618217110-ed94c76b-368a-444d-90dd-06feffcc2c13.png)

<font style="color:rgb(28, 30, 33);">②选择您的 LLM 提供商，输入您的 API 密钥并单击“Add Credential”</font>

注意：凭证取决于提供商，如果您选择 Vertex AI，那么您将看到Vertex Project、Vertex Location和Vertex Credentials字段![](https://cdn.nlark.com/yuque/0/2025/png/48196500/1753618312002-86657aee-b86b-4e11-bb6c-e0ef9390dd51.png)

<font style="color:rgb(28, 30, 33);">③当添加模型时，选择凭证</font>

<font style="color:rgb(28, 30, 33);">Add Model -> Existing Credentials -> 在下拉菜单中选择您的凭证</font>![](https://cdn.nlark.com/yuque/0/2025/png/48196500/1753618438766-88731ef0-7749-4621-b9c8-f277e1dba23d.png)

#### Pass-Through Endpoints
Pass‑Through Endpoint 是 LiteLLM 提供的一种机制，允许你通过 Proxy 将请求**原样转发**至第三方服务商 API，比如 Cohere、Bria、Vertex AI、VLLM 和 Bedrock 等。适用于想将已有项目迁移至 LiteLLM，或需要调用 provider 特定接口而 LiteLLM 尚未封装的场景。   这些端点不会对请求进行任何格式转换，完全保留 provider 原生的 request/response 结构，是一种“透明代理”机制：  

+ 客户端向 LiteLLM 发起请求（使用 Virtual Key 验证）；
+ LiteLLM Proxy 转发到外部 API；
+ 返回结果后原样返回给客户端；
+ 同时 LiteLLM 会记录调用日志、计费、预算控制等信息。

配置 Pass‑Through Endpoint？

在 **Models + Endpoints → Pass Through Endpoints** 页面，点击 **“Add Pass Through Endpoint”** 后，你需要填写如下字段：

1. **Path Prefix**：定义 LiteLLM Proxy 的路由路径前缀，例如 `/bria`、`/vllm`、`/vertex_ai`；
2. **Target URL**：真实的第三方服务 API 地址，例如 `https://engine.prod.bria-api.com`；
3. **Headers / Auth 配置**：设置认证头部（如 `Authorization: Bearer <API_KEY>`）、Content-Type、Accept 等；
4. **Pricing / Cost 设置**：设置每次调用的成本，比如 $12/request，以便 LiteLLM 跟踪消费并纳入预算控制。

完成配置后，端点将立即启用，你可以通过对应路径测试调用，确认是否访问成功。

####  Health Status  
 在 **LiteLLM Dashboard** 的 **“Models + Endpoints”** 页面中，**Health Status（健康状态）** 用于表示当前模型部署是否可用及其运行状况：  

该状态分为：

+ **Healthy（健康）**：模型正常响应，处于可被调用状态；
+ **Unhealthy（不健康）**：模型响应失败或超时，不建议继续调用；
+ 有时还会显示 **Degraded（降级）** 等中间健康级别。

通过 Action下的 Re-run Health Check按键，我们可以重新检查模型的健康状态。![](https://cdn.nlark.com/yuque/0/2025/png/48196500/1753619375348-149699da-4a47-4ae3-ba46-dd083fb19397.png)

####  Model Analytics  
**Model Analytics** 是 LiteLLM Dashboard 内置的指标分析模块，展示模型在不同维度上的使用情况。根据官方 Medium 教程：

+ 你可以在 UI 中点击左侧的 **“Usage”**（使用情况）或类似面板，查看各模型、团队、API Key 的调用频率、Token 使用、费用消耗等重要指标。
+ 这些数据支持企业运营团队监控成本、评估模型性能、优化调用策略。

####  Model Retry Settings  
+ **Model Retry Settings** 用于配置当模型调用发生错误（如超时、速率限制、内容策略违规等）时，LiteLLM Router 应该如何自动重试请求。
+ 这些设置可在 UI 中查看或修改，也可以通过 `config.yaml` 中的 `router_settings.retry_policy` 来统一配置。

 在 UI 中会列出多个错误类型对应的重试次数，例如：  ![](https://cdn.nlark.com/yuque/0/2025/png/48196500/1753619403296-cf9f3fcf-349c-4328-832e-5acd5bd8f8be.png)

+ **BadRequestErrorRetries: 对应的是客户端发送了非法或无效的请求，比如参数缺失或格式错误。  **
+ **AuthenticationErrorRetries: 表示身份验证失败，例如 API Key 无效或权限不足。  **
+ **TimeoutErrorRetries: 当请求超过指定超时时间未响应时。**
+ **RateLimitErrorRetries： 表示达到服务商的速率限制（如每分钟请求数或 token 限额）。  **
+ **InternalServerErrorRetries： 后端服务未响应、处理失败或服务器错误。  **
+ **ContentPolicyViolationErrorRetries： 当生成内容违反内容安全或策略限制时触发 。  **

在 **`**config.yaml**`** 文件内 **`**router_settings.retry_policy**`** 中设置全局默认策略：

```cpp
router_settings:
  retry_policy:
    TimeoutErrorRetries: 3
    RateLimitErrorRetries: 2
    InternalServerErrorRetries: 1
    AuthenticationErrorRetries: 0
    ContentPolicyViolationErrorRetries: 4
```



####  模型实例和逻辑模型  
+ 模型实例：指**具体部署的后端模型**，   每一个模型实例具有唯一标识（id），如 `azure-us-east-35-turbo`、`openai-gpt-3_5_turbo`，关联具体部署和访问凭证方案（API key、endpoint）。
+  逻辑模型 ：指**抽象的模型**，表示对外统一可见的模型名称，例如 `gpt-3.5-turbo`。 可以有多个模型实例映射到同一个逻辑模型，为该逻辑模型提供负载均衡、fallback、延迟优选等策略支持。  
+  为什么要这种映射机制？  
    - **高可用与容错**：提供一个逻辑模型的接口，其中有多个模型实例映射到这一个逻辑模型。即使其中的一个模型实例挂掉，LiteLLM也能自动fallback到其他实例，不会导致整个任务直接垮掉。
    - **负载均衡调度**：Router 支持多种策略（如最低延迟、最低成本、RPM/TPM 限额感知等），逻辑模型层级可覆盖多个实例，系统自动选择最优实例执行请求。
    - 在Aico或者是Dify这类型的平台上，如果要切换工作流中大模型的选择，需要手工操作。在一些较为复杂的工作流流程中，如果依靠手工进行操作，会大大降低开发效率。而通过使用这种逻辑模型，用户可以在Aico平台通过创建一个openAI的API兼容的模型，然后让其访问代理服务器，那么在后台它就可以通过用户的配置来自动选择用户希望的推理服务。

### 四、Usage
 在 **LiteLLM Dashboard** 中，切换至 **“Usage”** 或 **“Model Analytics”** 页面，即可访问 **Usage Dashboard（使用仪表板）**，它是 LiteLLM 提供的一体化使用、成本与性能分析中心，帮助你实时追踪 LLM 使用情况与费用分布。  

+ 使用标准 OpenAI 兼容字段 `"usage": {prompt_tokens, completion_tokens, total_tokens}` 记录每次调用的成本详情，支持以 USD 或 token 数形式显示。  
+ 图表展示模型调用次数（Requests）、成功/失败率、错误类型分布。便于判断特定模型或团队是否存在失败频次居高的问题。 
+ 支持按不同维度查看使用数据，如某团队或某 Virtual Key 的使用趋势与费用使用速率，便于资源分配优化与权限调整。  ![](https://cdn.nlark.com/yuque/0/2025/png/48196500/1753619432590-3a2cd45c-90c3-4023-9eb5-bdb8e264308d.png)

### 五、Teams
在 **LiteLLM Dashboard** 中，导航至 **Teams**（团队管理）页面可对团队进行统一管理、权限分配和日志策略配置。以下是该功能的全面说明：  

<font style="color:rgb(28, 30, 33);">①Teams -> Create New Team</font>

![](https://cdn.nlark.com/yuque/0/2025/png/48196500/1753619465321-b3ffb640-d98c-4ed7-8c7b-565cdce66fd2.png)

②创建新的team，其中Organization为一个组织，一个组织可以有多个team，

![](https://cdn.nlark.com/yuque/0/2025/png/48196500/1753619525203-399bbaad-0f01-46d7-a530-6be2e3962878.png)

### 六、Internal Users
1. <font style="color:rgb(28, 30, 33);">在代理上将具有权限的用户添加到团队</font>

前往Internal Users->+New User![](https://cdn.nlark.com/yuque/0/2025/png/48196500/1753619620692-e34b19f6-f769-41b7-8168-211d3be28e4c.png)

2. <font style="color:rgb(28, 30, 33);">与用户分享邀请链接</font>![](https://cdn.nlark.com/yuque/0/2025/png/48196500/1753619656186-e12d2597-adff-40a6-af6a-bee4109ee254.png)
3. <font style="color:rgb(28, 30, 33);">用户通过邮箱+密码验证登录</font>![](https://cdn.nlark.com/yuque/0/2025/png/48196500/1753619705186-f9e330c9-64a6-44e3-b9c2-09db779a9577.png)
4. <font style="color:rgb(28, 30, 33);">用户现在可以创建自己的密钥</font>![](https://cdn.nlark.com/yuque/0/2025/png/48196500/1753619751313-5070e936-a932-46d6-98e1-d668eab683aa.png)

### 七、Model Hub 
**Model Hub** 是一个面向开发者和使用者的模型展示页面，能让团队成员方便地查看哪些模型已经在 LiteLLM Proxy 中部署且被允许访问。 在模型列表中点击 **“Make Public”**（公开）操作，将某些模型向整个组织或指定用户公开。

#### 1.进入管理
<font style="color:rgb(28, 30, 33);">导航到管理 UI 中的模型中心页面</font>![](https://cdn.nlark.com/yuque/0/2025/png/48196500/1753617551701-8e963ee2-f2f9-4124-862b-9d7799cbb597.png)

#### 2.选择你想要的模型
<font style="color:rgb(28, 30, 33);">单击</font>Make Public并选择您想要展示的模型。

![](https://cdn.nlark.com/yuque/0/2025/png/48196500/1753617604109-955e0a34-bc03-4544-bb6d-45d9892745b1.png)

#### 3.确认
![](https://cdn.nlark.com/yuque/0/2025/png/48196500/1753617660999-282dafda-b878-41f6-aae2-9fb5df8f1e4f.png)

### 八、Logs
![](https://cdn.nlark.com/yuque/0/2025/png/48196500/1753619345570-69f1833e-4220-4de4-adfc-83a17449643a.png)Dashboard 中的 Logs 页面（通常在左侧导航菜单）用于**可视化查看每一次模型调用记录**，包括调用详情、消耗情况、异常信息等。展示每次请求的 **Virtual Key**、**Team**、调用的 **模型**、是否成功、Token 消耗、费用估算等数据。



### 九、Guardrails  
**Guardrails** 提供可插拔的策略机制，用于监控和控制 LLM 生成行为，包括：

+ **PII（个人身份信息）屏蔽**
+ **Prompt 注入检测**
+ **敏感内容过滤**
+ **内容格式验证**（如输出结构必须符合定义）


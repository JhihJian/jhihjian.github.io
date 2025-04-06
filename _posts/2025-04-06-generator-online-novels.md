---
layout: post
title: "基于LLM的智能中文网络小说生成器开发实践"
date: 2024-04-06
categories: [AI, NLP, 开发实践]
tags: [LLM, Gemini, 自然语言生成, Python, 网络小说]
---

# 基于LLM的智能中文网络小说生成器开发实践

## 项目背景

随着大型语言模型(LLM)技术的发展，AI生成内容的能力日益增强。我开发了一个智能中文网络小说生成器，旨在探索LLM在创意写作领域的应用潜力。本文记录开发过程中的关键技术点和思考。

## 系统架构

系统基于**代理模式**设计，分为四个核心代理：

1. **世界观构建代理(WorldBuilderAgent)**：负责创建小说的基础世界观
2. **角色创建代理(CharacterCreatorAgent)**：基于世界观创建小说角色
3. **剧情设计代理(PlotDesignerAgent)**：设计故事情节和章节大纲
4. **内容创建代理(ContentCreatorAgent)**：生成具体章节内容

![系统架构](/assets/img/novel-generator-arch.png)

## 关键技术点

### 1. LLM封装

为保证模型接口一致性，我创建了抽象基类`BaseLLM`：

```python
class BaseLLM:
    """LLM基类，定义模型接口"""
    
    async def generate(self, prompt: str, temperature: float = 0.7, max_tokens: int = None) -> str:
        """生成文本"""
        raise NotImplementedError
        
    async def generate_with_history(self, messages: List[Dict[str, str]], 
                                    temperature: float = 0.7, 
                                    max_tokens: int = None) -> str:
        """基于对话历史生成文本"""
        raise NotImplementedError
```

然后实现了`GeminiLLM`类封装Google Gemini API，关键代码：

```python
async def generate(self, prompt: str, temperature: float = 0.7, max_tokens: int = None) -> str:
    """生成文本响应"""
    # 准备生成配置
    generation_config = self.default_generation_config.copy()
    generation_config["temperature"] = temperature
    if max_tokens:
        generation_config["max_output_tokens"] = max_tokens
    
    # 使用同步API并在事件循环中运行
    loop = asyncio.get_event_loop()
    try:
        response = await loop.run_in_executor(
            None,
            lambda: self.model.generate_content(
                prompt,
                generation_config=generation_config,
                safety_settings=self.safety_settings,
            )
        )
        
        # 提取文本响应
        return response.text
    except Exception as e:
        # 错误处理...
```

### 2. 工具注册与调用机制

一个重要优化是使用`agno`库实现工具注册与调用机制。代理类能够注册工具函数并直接调用：

```python
def _register_tools(self):
    """注册代理可用的工具"""
    
    @tool(description="生成新的小说世界观")
    async def generate_world(description: str) -> Dict[str, Any]:
        """生成世界观的工具函数"""
        # 工具实现...
        
    # 直接设置tools属性
    self.tools = {
        "generate_world": generate_world,
        # 其他工具...
    }

async def _direct_call_tool(self, tool_name: str, **kwargs):
    """直接调用工具函数，绕过Agent.run机制"""
    if tool_name not in self.tools:
        raise ValueError(f"工具 {tool_name} 不存在")
        
    # 获取工具函数
    tool_func = self.tools[tool_name].entrypoint
    
    # 直接调用工具函数
    return await tool_func(**kwargs)
```

这种方式比使用字符串拼接提示词更可控且可维护。

### 3. 数据模型设计

创建了明确的数据模型表示世界观、角色和剧情，保证了数据一致性：

```python
class World:
    """世界观数据模型"""
    
    def __init__(self, name: str, description: str, background: str = "", 
                 cultures: List[Dict[str, Any]] = None, 
                 natural_laws: Dict[str, Any] = None,
                 id: Optional[str] = None):
        # 初始化代码...

class Character:
    """角色数据模型"""
    
    def __init__(self, name: str, world_id: str, 
                 basic_info: Dict[str, Any] = None,
                 abilities: Dict[str, Any] = None,
                 goals: List[str] = None,
                 id: Optional[str] = None):
        # 初始化代码...

class Plot:
    """剧情数据模型"""
    
    def __init__(self, title: str, world_id: str, 
                 background: str = "",
                 main_plot: Dict[str, Any] = None,
                 turning_points: List[Dict[str, Any]] = None,
                 chapters: List[Dict[str, Any]] = None,
                 id: Optional[str] = None):
        # 初始化代码...
```

### 4. 提示词工程

用于构建高质量的提示词模板，使LLM生成结构化、符合预期的输出：

```python
def _build_chapter_prompt(self, plot: Plot, world: World, characters: List[Character], 
                         chapter: Dict[str, Any], chapter_index: int) -> str:
    """构建生成章节内容的提示词"""
    
    # 构建世界观摘要
    world_summary = world.get_summary()
    
    # 构建角色摘要
    characters_summary = ""
    for i, character in enumerate(characters):
        characters_summary += f"\n角色{i+1}：{character.get_summary()}\n"
    
    # 构建剧情摘要
    plot_summary = plot.get_summary()
    
    # 章节信息
    chapter_title = chapter.get("title", f"第{chapter_index+1}章")
    chapter_summary = chapter.get("summary", "")
    
    # 构建场景信息
    scenes_info = ""
    if "scenes" in chapter and chapter["scenes"]:
        scenes_info = "\n".join([f"- {scene}" for scene in chapter["scenes"]])
    
    # 构建上下文（前后章节信息）
    context = self._build_chapter_context(plot, chapter_index)
    
    return f"""你是一个专业的中文小说作家，请根据以下信息创作精彩的小说章节内容。

世界观概要：
{world_summary}

角色概要：
{characters_summary}

剧情概要：
{plot_summary}

章节信息：
标题：{chapter_title}
概要：{chapter_summary}

章节场景：
{scenes_info}

相邻章节上下文：
{context}

请根据以上信息，创作一个完整、生动、情节连贯的章节内容。内容应当忠实于剧情概要和章节概要，但可以根据需要添加细节、对话和描写，使故事更加丰富和引人入胜。请确保写作风格流畅，情节发展合理，突出角色特点，并与整体剧情保持一致。

请直接返回创作的章节内容，不需要额外的解释或说明。"""
```

### 5. 错误处理与健壮性

提高系统健壮性的关键在于全面的错误处理：

1. **LLM响应解析错误处理**：
```python
def _parse_character_response(self, response: str) -> Dict[str, Any]:
    """解析LLM生成的角色数据"""
    try:
        # 尝试解析为JSON
        if "{" in response and "}" in response:
            json_str = self._extract_json(response)
            character_data = json.loads(json_str)
            return character_data
    except json.JSONDecodeError:
        print(f"解析角色响应为JSON时出错，尝试手动解析")
    except Exception as e:
        print(f"解析角色响应时发生未知错误: {str(e)}")
    
    # 回退到手动解析
    return self._manual_parse_character(response)
```

2. **日期时间处理**：
```python
def to_dict(self) -> Dict[str, Any]:
    """将剧情对象转换为字典"""
    # 处理created_at和updated_at可能是字符串或datetime的情况
    created_at_str = self.created_at
    if isinstance(self.created_at, datetime.datetime):
        created_at_str = self.created_at.isoformat()
        
    updated_at_str = self.updated_at
    if isinstance(self.updated_at, datetime.datetime):
        updated_at_str = self.updated_at.isoformat()
        
    return {
        # 其他字段...
        "created_at": created_at_str,
        "updated_at": updated_at_str,
    }
```

## 自动化脚本设计

为简化使用，我创建了一键生成脚本`auto_novel_generator.py`，用户只需提供简单描述，即可完成从世界观构建到小说生成的全流程：

```python
async def generate_novel(description: str, num_characters: int = 3, num_chapters: int = 3) -> bool:
    """从描述生成完整小说的流程"""
    # 确保数据目录存在
    data_dir = getattr(config, "DATA_DIR", "./data")
    os.makedirs(data_dir, exist_ok=True)
    
    # 创建存储实例
    storage = JsonStorage(base_dir=data_dir)
    
    # 1. 创建世界观
    world = await create_world(storage, description)
    if not world:
        print("无法继续生成小说，世界观创建失败")
        return False
    
    # 2. 创建角色
    characters = await create_characters(storage, world, num_characters)
    if not characters:
        print("无法继续生成小说，角色创建失败")
        return False
    
    # 3. 创建剧情
    plot_description = f"基于世界观'{world.name}'和已创建的角色，创建一个有吸引力的故事情节，{description}"
    plot = await create_plot(storage, world, characters, plot_description)
    if not plot:
        print("无法继续生成小说，剧情创建失败")
        return False
    
    # 4. 生成章节内容
    chapters = await generate_chapters(storage, world, plot, characters, num_chapters)
    if not chapters:
        print("章节内容生成失败")
        return False
    
    # 生成完整小说合集文件...
    
    return True
```

使用方式简单：
```bash
./auto_novel_generator.py --description "一个修仙世界，灵气充沛，有各种宗门和秘境，主要讲述一个平凡少年从凡人到修仙者的故事" --characters 3 --chapters 3
```

## 技术挑战与解决方案

### 1. 生成内容一致性问题

**挑战**：LLM生成的内容在多次调用间难以保持一致性，特别是在持续的故事情节中。

**解决方案**：
- 为每个生成步骤提供全面的上下文信息
- 设计结构化的提示词模板，明确指出需要保持一致的元素
- 将关键信息（如人物特性、世界规则）在每次调用时重新提供

### 2. 结构化输出解析

**挑战**：LLM输出的结构化内容(如JSON)格式不稳定，影响后续处理。

**解决方案**：
- 实现多层级的解析策略，先尝试直接JSON解析，失败后使用正则表达式提取关键信息
- 针对每种数据类型设计专门的备用解析函数
- 当所有解析方法失败时，提供合理的默认值确保系统继续运行

### 3. API调用错误处理

**挑战**：外部API(如Gemini)可能因配额限制、网络问题等原因失败。

**解决方案**：
- 实现自动重试机制
- 异步操作使用事件循环和异常处理
- 当模型不可用时能够自动降级使用备选模型

```python
# 异步API调用示例
try:
    loop = asyncio.get_event_loop()
    response = await loop.run_in_executor(
        None,
        lambda: self.model.generate_content(
            prompt,
            generation_config=generation_config,
            safety_settings=self.safety_settings,
        )
    )
    return response.text
except Exception as e:
    error_msg = str(e)
    if "models/gemini-pro is not found" in error_msg:
        # 尝试使用备选模型
        self.model_name = "gemini-1.5-pro"
        self.model = genai.GenerativeModel(model_name=self.model_name)
        # 重试操作...
```

## 性能与优化

1. **异步处理**：使用`asyncio`支持并发操作，减少等待时间
2. **缓存机制**：对常用数据实现缓存，避免重复生成
3. **优化提示词**：精简提示词长度，减少token消耗，同时保持生成质量

## 未来改进方向

1. **内容协调与修订**：引入内容审校代理，对生成内容进行自动修订
2. **记忆增强**：设计更好的记忆机制，使代理能够记住并利用过去生成的内容
3. **用户反馈整合**：整合用户反馈系统，根据反馈调整生成策略
4. **多模型支持**：扩展支持更多LLM模型，如Claude、GPT等

## 结论

基于LLM的智能小说生成系统展示了AI在创意写作领域的潜力。关键的技术点包括模块化代理设计、结构化数据模型、提示词工程、错误处理机制等。通过合理组织这些技术元素，系统能够生成结构连贯、内容丰富的小说内容。

代码已开源在GitHub上：[智能中文网络小说生成器](https://github.com/jhihjian/novel-generator)，欢迎交流讨论。 